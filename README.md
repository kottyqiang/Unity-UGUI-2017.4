### 重点关注的几个知识点
1. EventSystem UGUI的事件机制
2. Graphic
3. Mask、RectMask


### 关注的几个问题
1. 为什么尽量不勾选raycast
    - 不需要产生交互的就没必要勾选，避免检测
2. 为什么用UICollider
    - 避免不必要的渲染
3. 为什么要尽量避免动静分离
    - 避免ReBatch
        -  Batch构建过程是指Canvas通过结合网格绘制它所承载的UI元素，生成适当的渲染命令发送给Unity图形流水线。Batch的结果被缓存复用，直到这个Canvas被标为dirty，当Canvas中某一个构成的网格改变的时候就会标记为dirty。
4. 为什么要用RectMask2D取代Mask
    - Mask使用模板测试,RectMask使用UnityGet2DClipping然后clip掉裁剪的偏远
    - Mask要渲染两次，首尾之间是裁剪区域的渲染，最低也要3次
    - RectMask2D理论上可以只用渲染两次，底板一次，中间元素一次(因为clipRect值不同)
    - 在一些列表中，RectMask2D会做优化，例如完全在裁剪区域之外就不会渲染，部分再渲染区域会Clip类似于透明度测试，而Mask完全依赖模板测试，顶点着色器和偏远着色器都要执行。

### EventSystem 事件系统主要流程
1. EventSystem.Update  每个渲染帧刷一次
    - 备注1：m_SystemInputModules赋值
2. 调用当前使用的InputModule.Process
    - 备注2：现在的InputModule一般是StandaloneInputModule
    - 备注3：调用OnUpdateSelected
    - 备注4：调用OnMove
    - 备注5：调用OnSubmit
    - 备注6：调用OnCancel
3. 移动设备ProcessTouchEvents(下面使用ProcessTouchEvents作为流程)，PC调用ProcessMouseEvent
4. PointerInputModule.GetTouchPointerEventData方法，PointerInputModule为StandaloneInputModule父类。使用PointerEventData存储点击相关数据(例如点击位置、距离上一帧滑动距离)
5. EventSystem.RaycastAll。这里会使用所有的Raycasters进行检测。然后对检测结果排序。
    - 备注7：Raycaster检测
        - 没有勾选raycastTarget会过滤，不会进行一系列检测，例如RectTransformUtility.RectangleContainsScreenPoint、camera farClipPlane的深度检测、各自实现的IsRaycastLocationValid等。
    - 备注8：检测排序方式
        - 内部维护了graphic depth
6. 找到排序在最前面的检测结果
7. StandaloneInputModule.ProcessTouchPress方法，触发各种事件。
    - 备注9：DeselectIfSelectionChanged 设置选中的gameObject
    - 备注10：调用OnPointerExit
    - 备注11：调用OnPointerDown
    - 备注12：双击、多次点击、长点击实现
    - 备注13：调用OnPointerClick
    - 备注14：调用OnDrop
    - 备注15：调用OnEndDrag
    - 备注16：调用OnPointerUp
    - 备注17：调用OnInitializePotentialDrag
    - 备注18：调用OnPointerEnter
    - 备注20：调用OnBeginDrag
    - 备注21：调用OnDrag

### Graphic渲染流程
1. 关键方法Graphic SetLayoutDirty  SetVerticesDirty   SetMaterialDirty
2. CanvasUpdateRegistry   PerformUpdate    Canvas.willRenderCanvases时调用
3.  SetVerticesDirty  SetLayoutDirty
    - OnRectTransformDimensionsChange  自身尺寸发生变化的时候回调
4. Mask
    1. 前提是Image或者text使用的shader要支持stencil test
    2. 本质上是利用模板测试避免像素更新达到裁剪的显示效果
5. RectMask2D
    1. 利用UnityGet2DClipping根据设置的矩形进行裁剪

### 导致ReBuild的原因
1. 三种rebuild  
    1. Layout rebuild       用于布局
    2. vertices rebuild     用于重新构建Mesh,关键方法OnPopulateMesh
    3. material rebuild     主要是用于设置mat shader参数

1. SetAllDirty 
    1. OnTransformParentChanged
    2. OnEnable
    3. OnDidApplyAnimationProperties
    4. sprite
    5. overrideSprite
    6. SetNativeSize
    7. font
    8. FontTextureChanged

2. SetLayoutDirty （会取父一级的ILayoutGroup）
    1. OnRectTransformDimensionsChange
    2. text
    3. supportRichText
    4. resizeTextForBestFit
    5. resizeTextMinSize
    6. resizeTextMaxSize
    7. alignment
    8. fontSize
    9. horizontalOverflow
    10. verticalOverflow
    11. lineSpacing
    12. fontStyle
    13. 综上，text很多会导致layout rebuild，一般都是挂载了布局组件(各种布局组件,例如
    Grid Layout Group，水平/垂直布局组件，还有fitter一集scollRect)
3. SetVerticesDirty
    1. OnRectTransformDimensionsChange
    2. color
    3. type ( Simple,Sliced,Tiled,Filled)
    4. preserveAspect
    5. fillCenter、fillMethod、fillAmount、fillClockwise、fillOrigin
    6. texture
    7. uvRect
    8. shadow-effectColor、effectDistance、useGraphicAlpha
    9. BaseMeshEffect-shadow/outline一类的: OnEnable、OnDisable、OnDidApplyAnimationProperties
    10. text-text、supportRichText、resizeTextForBestFit、resizeTextMinSize、resizeTextMaxSize、alignment、alignByGeometry、fontSize、horizontalOverflow、verticalOverflow、lineSpacing、fontStyle
4. SetMaterialDirty
    1. graphic-material
    2. MaskableGraphic-showMaskGraphic、OnEnable、OnDisable、maskable、OnTransformParentChanged、OnCanvasHierarchyChanged、RecalculateMasking
    3. rawImage-texture

### UGUI的坑
1. 在设置alphaHitTestMinimumThreshold的情况下(在做一些不规则点击的时候有可能会使用到它)
    1. 图片导入的时候如果勾选Tight会有问题，因为勾选Tight会剔除掉图片周边的透明像素(如果周边没有透明像素那就没问题)，导致计算出现问题，使得点击出现偏移
    2. 接收射线检测的graphic IsRaycastLocationValid返回true，但父节点如果IsRaycastLocationValid返回false,那么射线检测会返回false，即失败。
2. 批处理注意事项
    1. 同一个图集中的元素才会进行批处理，不同图集的元素重叠在一起时会中断批处理，text也会中断批处理。


### Unity UGUI官方博文优化
1. https://www.jianshu.com/p/1949c96da4c6   基础介绍
2. https://www.jianshu.com/p/7f4b55507d0b   Rebuild Rebatch 概述
3. https://www.jianshu.com/p/82eb8fae3b9d   分析工具
4. https://www.jianshu.com/p/a57928fe201d   优化手段
5. https://www.jianshu.com/p/0d1ff6c25544   UI控件的性能注意要点
6. https://www.jianshu.com/p/0a1a5bf2fe36   优化补充手段