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



# 欧拉角与四元数

每次输入的xyz旋转值都是都是从000开始直接输出旋转结果的过程

开始输入10 90 0 这时看到z轴在之前x轴位置，然后去把x调到20，想绕着目前的x轴旋转，实际上看依然是现在的z轴旋转，因为第二次调x不是相对于10 90 0 的状态给x加10，而是从000开始计算20 90 0的结果是什么，都是相对于最开始的轴旋转的，这就是动态欧拉角的计算方法

变换的顺序是预先确定的顺序而非实际输入的顺序。（并不会因为我们每次输入一次，欧拉角计算参考的坐标系同步为我们改完后的坐标系）
**欧拉角是用固定顺序对初始坐标系进行变换**



个人感觉：就是说，Unity中直接按角度旋转，是用欧拉角描述的，在代码中想要使物体沿自身坐标轴旋转，用欧拉角都是按照初始状态计算的（既然依次设定每个轴的旋转都是按照最开始的轴，也就是按模型初始坐标轴旋转嘛，并不是真正的“相对”，还是一个固定不变的轴，就跟按世界坐标旋转似的），所以欧拉角不适合用来运用到相对自身转动，这时用**四元数**描述**自身旋转**最合适



**检视面板上的Rotation的值是相对于父物体的旋转角度，而不是相对于自身坐标轴的旋转角度（即（30，40，50）并不是自身相对初始坐标轴的xyz各旋转了30 40 50）**

如果父物体有初始旋转角度，在scene中用旋转工具按某个轴旋转物体，本物体的Rotation三个值都会改变，因为这个值是相对于父物体的角度值

而相对自身坐标轴的旋转角度其实无法直接获得，因为transform.localRotation.eulerAngles这个欧拉角是根据当前物体的旋转四元数转换过来的值，并不是相对开始旋转的xyz值



transform.rotation = 和 *= Quaternion.Euler(0, 10, 0)的区别：

=四元数：无视初始角度，<u>从世界角度为0的状态</u>，物体按初始y轴（就是世界y轴）旋转10度

*=四元数：即transform.rotation = transform.rotation * Quaternion.AngleAxis(10, Vector3.up)，先将物体<u>从世界角度为0的状态</u>，旋转rotation值，再以旋转后的自身坐标轴的y轴旋转10度

使用四元数的时候，第一个四元数旋转是按0开始的旋转，后面叠加的四元数旋转是按前面旋转后的自身轴向来旋转



四元数左乘三维向量，将该向量按照四元数表示的角度旋转

```csharp
Vector3 point = new Vector3(0,0,10);
Vector3 newPoint = Quaternion1 * point;
```

多个四元数相乘，组合旋转效果，顺序有关，先进行旋转1，再进行旋转2

```csharp
Quaternion qt1 = Quaternion1 * Quaternion2;
```



Unity API

transform.rotation = Quaternion.LookRotation(dir)

look只会旋转本地xy轴，z轴不会旋转

