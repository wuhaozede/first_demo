# 鼠标点选

### 一. 思路

##### 1. 如何获取鼠标位置

本项目主要基于ArcGIS Engine实现相关的要素加载与编辑功能，其中ArcGIS Engine，提供了名为MapControl的控件，该控件的特点就是可以将空间要素添加到指定的绘图区域，并且以图的形式将空间要素可视化，由控件提供了可以对空间要素进行选取和修改的窗口，所以我们只需要获取鼠标在窗口的点击事件即可实现相关的操作。

具体可以通过该控件提供的click事件，对事件进行函数的绑定，通过函数的输入参数获取鼠标的点击事件位置。

##### 2. 如何判断要素位置

MapControl控件提供了一个空间要素可视化的窗口。加载空间要素的方式是根据空间要素的坐标以及相关的地理坐标系与投影，所以要素的位置与窗口的二维坐标之间，是由函数的转换关系。具体的，该控件提供了一套转换经纬度与鼠标坐标的接口，并且封装了更高级的方法，在选取元素的时候可以进行调用。

##### 3. 多个要素层叠如何选取

在空间要素加载的同时，控件会绑定另一个名为TOCControl的控件，该控件可以将显示在MapControl中的空间元素以层叠关系显示在指定可视化区域。也就是说，TOCControl控件提供了一个空间要素的层叠关系索引，我们可以根据索引去判断，在某一经纬度上，空间要素之间的层叠关系，并且以最上层的空间要素作为选取的对象进行操作。

### 二. 实现代码

##### 1. 获取鼠标位置

```csharp
private void axMapControl1_OnMouseMove(object sender, IMapControlEvents2_OnMouseMoveEvent e)
        {
            // 显示当前比例尺,整数
            toolStripStatusScale.Text = " 比例尺 1:" + axMapControl1.MapScale.ToString("f0");

            // 显示当前坐标，保留小数点后四位
            toolStripStatusCoordinates.Text = String.Format(" 当前坐标 X = {0}, Y={1} {2}",
                                                             e.mapX.ToString("f4"),
                                                             e.mapY.ToString("f4"),
                                                             YCMap.Utils.SystemHelper.ConvertEsriUnit(axMapControl1.MapUnits));
        }
```

##### 2. 判断要素位置与多个要素层叠选取

基本代码，但是arcgis提供了高级封装，无需调用

```csharp
IFields pFields=pFeaClass.Fields;
            for (int i = 0; i < pFields.FieldCount; i++)
            {
                dataGridView1.Columns.Add(pFields.Field[i].Name, pFields.Field[i].AliasName);
            }
            IFeatureCursor pFeaCursor = pFeaClass.Search(null, true);
            IFeature pFeature=pFeaCursor.NextFeature();
            while (pFeature != null)
            {
                int index= dataGridView1.Rows.Add();
                
                for (int i = 0; i < pFields.FieldCount; i++)
                {
                    dataGridView1.Rows[index].Cells[i].Value = pFeature.get_Value(i);
                    String[] val=new String[5];
                    dataGridView1.Rows.Add(val);
                }
                pFeature=pFeaCursor.NextFeature();
            }
```

### 三. 调试过程

在对于获取鼠标位置调试的时候，发生了错误信息。错误为，`axMapControl1未将对象引用到对象的实例`，错误指定到对于`axMapControl1`控件上，查找了相关资料后，我们将`axMapControl1`控件的属性值进行输出，发现了控件并没有被代码定义，所以在代码之前，我们加入了空间的可用断，从而解决了这个问题。

```csharp
private void axMapControl1_OnMouseUp(object sender, IMapControlEvents2_OnMouseUpEvent e)
{
    if (e.button == 1)
    {
        for (int i = 0; i < axMapControl1.Map.LayerCount; i++)
        {
        	if (axMapControl1.Map.get_Layer(i) is IFeatureLayer)//在此处检查
            {
	            if (geometry == null)
	                continue;
	            spatialFilter.Geometry = geometry;
	            featureLayer = axMapControl1.Map.get_Layer(i) as IFeatureLayer;
	            featureCursorBuffer = featureLayer.Search(spatialFilter, false);
	            feature = featureCursorBuffer.NextFeature();
	        }
        }
    }
    else if (e.button == 2)
    {

    }
}
```

# 地图的翻转平移旋转

### 一. 思路

##### 1. 如何选中地图

选中地图的私户与选中要素的思路相反，因为都是基于MapControl控件开发的，同样的用MapControl控件获取鼠标在空间当中的坐标之后，首先判断鼠标是否选中元素，如果没有选中元素的话，就可以通过工具栏中的选项设置鼠标的属性，同时判断鼠标属性与鼠标的位置坐标，从而判断鼠标是否选中地图还是选中空间元素。

##### 2. 如何判断鼠标移动轨迹

在判定完鼠标是否选中地图之后，如果选中地图且没有选中元素的话，同时鼠标一直是处于按压状态，就可以开始计算鼠标的实时坐标，并且将坐标实时的输送给地图控件MapControl，由MapControl控件对整个地图进行实时的更新，从而达到判断鼠标移动轨迹的效果。

##### 3. 如何旋转坐标

由控件MapControl提供了一套对地理坐标实时更新，实时转化的接口，并不需要实现鼠标坐标转化，只要通过接口去调整MapControl控件中的坐标轴方向，就可以达到旋转地图的效果，同时，地图空间可以实时的更新鼠标坐标系与地图坐标系之间的角度关系，从而达到地图的旋转效果。

### 二. 实现代码

##### 1.  选中地图

```csharp
// 添加按钮到工具条
OpenNewMapDocument openMapDoc = new OpenNewMapDocument(m_controlsSynchronizer);
axToolbarControl1.AddItem(openMapDoc, -1, 0, false, -1, esriCommandStyles.esriCommandStyleIconOnly);
```

##### 2.  判断鼠标移动轨迹

```csharp
private void axTOCControl1_OnMouseDown(object sender, ITOCControlEvents_OnMouseDownEvent e)
        {
            //如果不是右键按下直接返回
            if (e.button != 2) return;

            esriTOCControlItem itemType = esriTOCControlItem.esriTOCControlItemNone;
            IBasicMap basicMap = null;
            ILayer layer = null;
            object other = null;
            object data = null;

            //判断所选菜单的类型
            m_tocControl.HitTest(e.x, e.y, ref itemType, ref basicMap, ref layer, ref other, ref data);

            //确定选定的菜单类型， Map 或是图层菜单
            if (itemType == esriTOCControlItem.esriTOCControlItemMap)
                m_tocControl.SelectItem(basicMap, null);
            else
                m_tocControl.SelectItem(layer, null);
            //设置 CustomProperty 为 layer ( 用于自定义的 Layer 命令)
            m_mapControl.CustomProperty = layer;
            //弹出右键菜单
            if (itemType == esriTOCControlItem.esriTOCControlItemMap)
                m_menuMap.PopupMenu(e.x, e.y, m_tocControl.hWnd);
            if (itemType == esriTOCControlItem.esriTOCControlItemLayer)
                m_menuLayer.PopupMenu(e.x, e.y, m_tocControl.hWnd);
        }
```

### 三. 调试过程

在这部分的代码调试中，并没有遇到突出的问题，主要是地图旋转在空间中已经提供了一套非常完善的接口，是不需要主动调用就可以实现的，但是一开始我并没有发现这个接口，所以导致写了很多没有用的代码，但是后来都使用高级接口替换掉了。所以这个可以体现出，对于一个没有使用的工具，我们需要去先熟悉再使用，否则就会出现一些功能或者一些快捷操作，无法熟练的使用，导致浪费很多时间。
