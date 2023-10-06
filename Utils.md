# 坐标转换



Canvas 

Render Mode

Screen Space - Overlay

画布大小就是屏幕分辨率

Screen Space - Camera

画布大小就是屏幕分辨率

World Space

画布大小是自定义的Width和Height



UI Scale Mode

Constant Pixel Scale：无论屏幕分辨率如何，UI 元素都保持相同的像素尺寸

开始分辨率(1920x1080)，UI在(960, 540)画布右上角，后来分辨率(1366x768)，UI仍然在(960, 540)，要移到画布右上角则移到(683, 384)

Scale With Screen Size：UI尺度根据屏幕分辨率缩放

分辨率(1920x1080)，UI坐标以画布中心为原点，右上角为(960, 540)



世界->屏幕

世界物体对应到屏幕的点

```csharp
Vector2 screenPoint = Camera.main.WorldToScreenPoint(worldPoint)
```

屏幕->世界

法1：屏幕射线检测

屏幕坐标z轴可为任意值不会有影响，射线构建只使用xy轴

```csharp
Ray ray = Camera.main.ScreenPointToRay(screenPoint);
if (Physics.Raycast(ray, out RaycastHit hitInfo))
{
    Vector3 worldPoint = hitInfo.point;
}
```

法2：屏幕坐标转换到世界中相对相机z距离对应的面

z距离是相对相机的z轴长度（planeZ为正，转换到相机前面的某个面）

```csharp
Vector3 worldPoint = Camera.main.ScreenToWorldPoint(new Vector3(screenPoint.x, screenPoint.y, planeZ));
```

UI世界->屏幕

UI的世界坐标转屏幕

Canvas Render Mode 为Overlay，则uiCamera为null，其余两种为对应的Camera

UI的世界坐标z轴可为任意值，算得的屏幕坐标z轴都是0

```csharp
Vector2 screenPoint = RectTransformUtility.WorldToScreenPoint(uiCamera, uiTransform.position);
```

为Overlay时，UI的世界坐标等于屏幕坐标（xy轴相等，z轴可清零）

```csharp
Vector2 screenPoint = uiTransform.position;
```

屏幕->UI世界

rectTransform据实验是可以任意的rect，一般使用移动目标的rect或目标的parent的rect

```csharp
RectTransformUtility.ScreenPointToWorldPointInRectangle(rectTransform, screenPoint, uiCamera, out var worldPoint);
uiTransform.position = worldPoint;
```



另外的

屏幕->UI本地

rectTransform为父物体的rect

```csharp
RectTransformUtility.ScreenPointToLocalPointInRectangle(parentRect, screen, uiCamera, out var localPoint);
uiTransform.localPosition = localPoint;
```



