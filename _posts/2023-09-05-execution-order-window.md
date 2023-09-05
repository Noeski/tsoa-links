---
layout: post
author: Noeski
tags: [Unity]
---

![Screenshot](/images/posts/2023-09-05/execution-order.png)

This Editor tool will display a list of all `MonoBehaviour`s in a project (packages included) and their respective execution order. This includes the [execution order](https://docs.unity3d.com/Manual/class-MonoManager.html) that can be set in the editor and the [undocumented](https://uninomicon.com/defaultexecutionorder) `DefaultExecutionOrder` attribute which can be set on a class. 

To use, add this script to your Editor folder and open the window from `Windows` -> `General` -> `Script Execution Order`.

```csharp
using System;
using System.Collections.Generic;
using UnityEditor;
using UnityEngine;

public class ExecutionOrderWindow : EditorWindow {
    private static class Style {
        public static GUIStyle rightAlignedText;
        public static Color[] listColors;

        static Style() {
            rightAlignedText = new GUIStyle(EditorStyles.label) { alignment = TextAnchor.MiddleRight };

            byte alpha = 255;

            listColors = new Color[] {
                new Color32(255, 0, 0, alpha),
                new Color32(255, 128, 0, alpha),
                new Color32(255, 255, 0, alpha),
                new Color32(128, 255, 0, alpha),
                new Color32(0, 255, 0, alpha),
                new Color32(0, 255, 128, alpha),
                new Color32(0, 255, 255, alpha),
                new Color32(0, 128, 255, alpha),
                new Color32(0, 0, 255, alpha),
                new Color32(128, 0, 255, alpha),
                new Color32(255, 0, 255, alpha),
                new Color32(255, 0, 128, alpha),
            };
        }
    }

    [MenuItem("Window/General/Script Execution Order")]
    public static void ShowWindow() {
        ExecutionOrderWindow window = GetWindow<ExecutionOrderWindow>();
        window.titleContent = new GUIContent("Execution Order");
        window.Show();
    }

    private struct ScriptExecutionOrder : IComparable<ScriptExecutionOrder> {
        public MonoScript script;
        public int executionOrder;

        public int CompareTo(ScriptExecutionOrder other) {
            if(executionOrder != other.executionOrder) {
                return executionOrder.CompareTo(other.executionOrder);
            }
            else {
                return script.name.CompareTo(other.script.name);
            }
        }
    }

    private List<ScriptExecutionOrder> _scripts = new List<ScriptExecutionOrder>();
    private Vector2 _scrollPosition;

    private void OnEnable() {
        ReloadScripts();
    }

    private void OnGUI() {
        EditorGUILayout.BeginVertical();
        _scrollPosition = EditorGUILayout.BeginScrollView(_scrollPosition);

        var previousExecutionOrder = int.MinValue;
        var colorIndex = -1;

        foreach(var script in _scripts) {
            var executionOrder = script.executionOrder;

            if(executionOrder != previousExecutionOrder) {
                colorIndex = (colorIndex + 1) % Style.listColors.Length;
                previousExecutionOrder = executionOrder;
            }

            GUI.color = Style.listColors[colorIndex];
            EditorGUILayout.BeginHorizontal();
            GUI.color = Color.white;

            var rect = GUILayoutUtility.GetRect(4, 0, GUILayout.ExpandWidth(false), GUILayout.ExpandHeight(true));
            EditorGUI.DrawRect(rect, Style.listColors[colorIndex]);
            DrawMonoScript(script.script);
            EditorGUILayout.LabelField(executionOrder.ToString(), Style.rightAlignedText, GUILayout.Width(120f));
            EditorGUILayout.EndHorizontal();
        }

        EditorGUILayout.EndScrollView();
        EditorGUILayout.EndVertical();
    }

    private void DrawMonoScript(MonoScript script) {
        GUIContent content = EditorGUIUtility.ObjectContent(script, script.GetType());
        EditorGUILayout.LabelField(content);

        var evt = Event.current;
        var rect = GUILayoutUtility.GetLastRect();

        if(evt.type == EventType.MouseDown && evt.button == 0 && rect.Contains(evt.mousePosition)) {
            EditorGUIUtility.PingObject(script);
            evt.Use();
        }
    }

    private void ReloadScripts() {
        _scripts.Clear();

        var scripts = MonoImporter.GetAllRuntimeMonoScripts();

        foreach(var script in scripts) {
            if(!IsValidScript(script)) {
                continue;
            }

            var executionOrder = MonoImporter.GetExecutionOrder(script);

            if(executionOrder == 0) {
                executionOrder = GetDefaultExecutionOrderFor(script.GetClass());
            }

            _scripts.Add(new ScriptExecutionOrder { script = script, executionOrder = executionOrder });
        }

        _scripts.Sort();
    }

    private bool IsValidScript(MonoScript script) {
        if(script.GetClass() == null) {
            return false;
        }

        bool isMonoBehaviour = typeof(MonoBehaviour).IsAssignableFrom(script.GetClass());
        bool isScriptableObject = typeof(ScriptableObject).IsAssignableFrom(script.GetClass());
        if(!isMonoBehaviour && !isScriptableObject) {
            return false;
        }

        return true;
    }

    private int GetDefaultExecutionOrderFor(Type klass) {
        var attribute = GetCustomAttributeOfType<DefaultExecutionOrder>(klass);
        if(attribute == null) {
            return 0;
        }

        return attribute.order;
    }

    private T GetCustomAttributeOfType<T>(Type klass) where T : Attribute {
        var attributeType = typeof(T);

        var attrs = klass.GetCustomAttributes(attributeType, true);
        if(attrs != null && attrs.Length != 0) {
            return (T)attrs[0];
        }

        return null;
    }
}

```
