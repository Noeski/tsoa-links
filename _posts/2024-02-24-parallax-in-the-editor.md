---
layout: post
author: Noeski
image: parallax.gif
tags: [Unity]
---

{% include image.html name="parallax.gif" alt="Screenshot" %}

## Parallax
Parallax is a very common effect in 2D games where layers move at slightly different speeds or directions than the camera to give the effect of depth and perspective.

There are several ways to do this, but for this tutorial we will create a `ParallaxLayer` script and an accompanying prefab that can be placed in the scene.

```csharp
public class ParallaxLayer : MonoBehaviour {
    public Transform content;
    public float horizontalFactor;
    public float verticalFactor;
}
```

Now we need one more script that can manage the group of layers and modify their content's position based on a given camera position.

```csharp
public class ParallaxManager : MonoBehaviour {
#if UNITY_EDITOR
    public float cameraHeight;
    public float cameraAspectRatio;
#endif

    [HideInInspector]
    public List<ParallaxLayer> layers = new List<ParallaxLayer>();

#if UNITY_EDITOR
    public void OnValidate() {
        GetComponentsInChildren<ParallaxLayer>(layers);
    }
#endif

    public void ResetLayers() {
        foreach(var layer in layers) {
            var layerContentPosition = layer.content.position;

            layerContentPosition.x = 0;
            layerContentPosition.y = 0;

            layer.content.localPosition = layerContentPosition;
        }
    }

    public void UpdateLayers(Vector2 cameraPosition) {
        foreach(var layer in layers) {
            var layerPosition = (Vector2)layer.transform.position;
            var layerContentPosition = layer.content.position;
            var difference = cameraPosition - layerPosition;

            layerContentPosition.x = layerPosition.x + difference.x * layer.horizontalFactor;
            layerContentPosition.y = layerPosition.y + difference.y * layer.verticalFactor;


            layer.content.position = layerContentPosition;
        }
    }
}
```

One important thing to note is that there should be a single `ParallaxManager` in the scene and a group of `ParallaxLayer` prefabs that are children of the manager in order to automatically keep track of the layers.

{% include image.html name="parallax-manager.png" alt="Screenshot" %}
{% include image.html name="parallax-layer.png" alt="Screenshot" %}
{% include image.html name="parallax-manager-setup.png" alt="Screenshot" %}

## Scene View Toolbar Button
In order to preview the parallax effect in the editor we need to create a button that will let us toggle updating our `ParallaxManager` using the `SceneView`'s camera - this script should live inside of an `Editor` folder since we only want it to be compiled when using the editor.

```csharp
[EditorToolbarElement(id, typeof(SceneView))]
public class ParallaxToolbarButton : EditorToolbarToggle {
    public const string id = "Scene View/Parallax Button";

    private bool _isActive;
    private List<ParallaxManager> _parallaxManagers = new List<ParallaxManager>();

    public ParallaxToolbarButton() {
        icon = EditorGUIUtility.FindTexture("scenevis_visible_hover");
        tooltip = "Toggle parallax";

        this.RegisterValueChangedCallback(evt => {
            if(evt.newValue) {
                StartPreview();
            }
            else {
                StopPreview();
            }
        });

        RegisterCallback<AttachToPanelEvent>(OnAttachedToPanel);
        RegisterCallback<DetachFromPanelEvent>(OnDetachFromPanel);
    }

    private void OnAttachedToPanel(AttachToPanelEvent evt) {
        EditorApplication.playModeStateChanged += OnPlayModeStateChanged;
        SceneView.duringSceneGui += OnSceneGUI;

        if(EditorApplication.isPlayingOrWillChangePlaymode) {
            SetEnabled(false);
        }
    }

    private void OnDetachFromPanel(DetachFromPanelEvent evt) {
        EditorApplication.playModeStateChanged -= OnPlayModeStateChanged;
        SceneView.duringSceneGui -= OnSceneGUI;

        if(_isActive) {
            StopPreview();
        }
    }

    private void StartPreview() {
        _isActive = true;
        _parallaxManagers.AddRange(Object.FindObjectsOfType<ParallaxManager>());

        foreach(var parallaxManager in _parallaxManagers) {
            parallaxManager.OnValidate();
        }

        SceneView.RepaintAll();
    }

    private void StopPreview() {
        _isActive = false;

        foreach(var parallaxManager in _parallaxManagers) {
            parallaxManager.ResetLayers();
        }

        _parallaxManagers.Clear();

        SceneView.RepaintAll();

        SetValueWithoutNotify(false);
    }

    private void OnPlayModeStateChanged(PlayModeStateChange state) {
        if(state == PlayModeStateChange.ExitingEditMode || state == PlayModeStateChange.EnteredPlayMode) {
            if(_isActive) {
                StopPreview();
            }

            SetEnabled(false);
        }
        else {
            SetEnabled(true);
        }
    }

    private void OnSceneGUI(SceneView sceneView) {
        if(!_isActive ||
           EditorApplication.isPlayingOrWillChangePlaymode ||
           PrefabStageUtility.GetCurrentPrefabStage() != null) {
            return;
        }

        var cameraPosition = sceneView.camera.transform.position;

        foreach(var parallaxManager in _parallaxManagers) {
            parallaxManager.UpdateLayers(cameraPosition);

            var cameraSize = new Vector2(parallaxManager.cameraAspectRatio * parallaxManager.cameraHeight, parallaxManager.cameraHeight);
            Handles.DrawSolidRectangleWithOutline(new Rect(cameraPosition - cameraSize * 0.5f, cameraSize), Color.clear, Color.green);
        }
    }
}

[Overlay(typeof(SceneView),
    _id,
    _displayName,
    defaultDisplay = true,
    defaultDockZone = DockZone.TopToolbar,
    defaultLayout = Layout.HorizontalToolbar)]
[Icon("Icons/Overlays/GridAndSnap.png")]
public class ParallaxToolbar : ToolbarOverlay {
    private const string _id = "Scene View/Parallax";
    private const string _displayName = "Parallax";

    public ParallaxToolbar() : base(
        ParallaxToolbarButton.id) { }
}
```

The important code here is inside `OnSceneGUI` where we manually update each `ParallaxManager` found in the current scene using the `SceneView`'s camera. It's also important to note that we draw a preview of the camera's orthographic bounds using the `cameraHeight` and `cameraAspectRatio` settings in the `ParallaxManager` which gives a better idea of how the layers will look in game when moving around in the scene.

{% include image.html name="toolbar-button.png" alt="Screenshot" %}

Now the toolbar should appear containing a single button that will let us toggle on/off the parallax effect in the editor.
