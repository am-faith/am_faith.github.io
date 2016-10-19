---
layout:     post
title:      Android Resource 之 drawable
date:       2014-09-01 22:21:29
summary:    Android Resource
categories: android resource
---

Android有很多种drawable类型，除了前几篇详细讲解的shape、selector、layer-list，还有上一篇提到的color、bitmap、clip、scale、inset、transition、rotate、animated-rotate、lever-list等等，本篇文章将汇总介绍所有剩下的drawable资源。

## 普通图片
图片是最常用的drawable资源，格式包括：png(推荐)、jpg(可接受)、gif(不建议)。用图片资源需要根据不同屏幕密度提供多张不同尺寸的图片，它们的关系如下表：

|密度分类  |密度值范围  |代表分辨率	|图标尺寸  |图片比例|
|:--------------:|:-----------:|:---------------:|:------------:|:-----------:|
|mdpi|	120~160dpi|	320x480px|	48x48px|	1|
|hdpi|	160~240dpi|	480x800px|	72x72px|	1.5|
|xhdpi|	240~320dpi|	720x1280px|	96x96px|	2|
|xxhdpi|	320~480dpi|	1080x1920px|144x144px|	3|
|xxxhdpi|	480~640dpi|	1440x2560px|192x192px|	4|

本来还有一个ldpi的，但现在这种小屏幕的设备基本灭绝了，所以不需要再考虑适配。如上表所示，一套图片一般需要提供5张不同比例的图片。还好有切图工具，可以让切图变得简单，这里推荐两款：Cutterman和Cut&Slice me，都是Photoshop下的插件，输出支持android、ios和web三种平台。
使用切图工具虽然方便了，但还是无法避免一套图片需要提供多张不同尺寸的图片，这会加大安装包的大小。另外，需要对图片做改动时，比如换个颜色，必须更换所有尺寸图片。所以，建议尽量减少引入图片，而通过使用shape、layer-list等自己画，易于修改和维护，也减少了安装包大小，适配性也更好。

## bitmap标签
可以通过bitmap标签对图片做一些设置，如平铺、拉伸或保持图片原始大小，也可以指定对齐方式。看看bitmap标签的一些属性吧：

* android:src 必填项，指定图片资源，只能是图片，不能是xml定义的drawable资源

* android:gravity 设置图片的对齐方式，比如在layer-list中，默认会尽量填满整个视图，导致图片可能会被拉伸，为了避免被拉伸，就可以设置对齐方式，可取值为下面的值，多个取值可以用 | 分隔：

	* top 图片放于容器顶部，不改变图片大小
	* bottom 图片放于容器底部，不改变图片大小
	* left 图片放于容器左边，不改变图片大小
	* right 图片放于容器右边，不改变图片大小
	* center 图片放于容器中心位置，包括水平和垂直方向，不改变图片大小
	* fill 拉伸整张图片以填满容器的整个高度和宽度，默认值
	* center_vertical 图片放于容器垂直方向的中心位置，不改变图片大小
	* center_horizontal 图片放于容器水平方向的中心位置，不改变图片大小
	* fill_vertical 在垂直方向上拉伸图片以填满容器的整个高度
	* fill_horizontal 在水平方向上拉伸图片以填满容器的整个宽度
	* clip_vertical 附加选项，裁剪基于垂直方向的gravity设置，设置top时会裁剪底部，设置bottom时会裁剪顶部，其他情况会同时裁剪顶部和底部
	*clip_horizontal 附加选项，裁剪基于水平方向的gravity设置，设置left时会裁剪右侧，设置right时会裁剪左侧，其他情况会同时裁剪左右两侧
	
* android:antialias 设置是否开启抗锯齿

* android:dither 设置是否抖动，图片与屏幕的像素配置不同时会用到，比如图片是ARGB 8888的，而屏幕是RGB565

* android:filter 设置是否允许对图片进行滤波，对图片进行收缩或者延展使用滤波可以获得平滑的外观效果

* android:tint 给图片着色，比如图片本来是黑色的，着色后可以变成白色

* android:tileMode 设置图片平铺的方式，取值为下面四种之一：

	* disable 不做任何平铺，默认设置
	* repeat 图片重复铺满
	* mirror 使用交替镜像的方式重复图片的绘制
	* clamp 复制图片边缘的颜色来填充容器剩下的空白部分，比如引入的图片如果是白色的边缘，那么图片所在的容器里除了图片，剩下的空间都会被填充成白色
	
* android:alpha 设置图片的透明度，取值范围为0.0~1.0之间，0.0为全透明，1.0为全不透明，API Level最低要求是11，即Android 3.0

* android:mipMap 设置是否可以使用mipmap，但API Level最低要求是17，即Android 4.2

* android:autoMirrored 设置图片是否需要镜像反转，当布局方向是RTL，即从右到左布局时才有用，API Level 19(Android 4.4)才添加的属性

* android:tileModeX 和tileMode一样设置图片的平铺方式，只是这个属性只设置水平方向的平铺方式，这是API Level 21(Android 5.0)才添加的属性

* android:tileModeY 和tileMode一样设置图片的平铺方式，只是这个属性只设置垂直方向的平铺方式，这是API Level 21(Android 5.0)才添加的属性

* android:tintMode 着色模式，也是API Level 21(Android 5.0)才添加的属性

## 点九图片
点九图片文件扩展名为：.9.png，通过点九图片可以做局部拉伸，比如，一张圆角矩形图片，我们不想让它的四个边角都被拉伸从而导致模糊失真，使用点九图就可以控制拉伸区域，让四个边角保持完美显示。效果如下图：

![drawable1](https://github.com/am-faith/am_faith.github.io/tree/gh-pages/images/post/点九效果图.jpg)

画点九图一般用Android SDK工具集里的draw9patch工具，只需要在四条边画黑线就可以了，如下图所示：

![drawable2](https://github.com/am-faith/am_faith.github.io/tree/gh-pages/images/post/点九图拉伸区域图.jpg)

拉伸区域就是图片会被拉伸的部分，可以为1个点，也可以为一条线，甚至也可以为断开的几个点或几条线，总之，有黑点的地方就会被拉伸，没有黑点的地方就不会被拉伸。而显示内容区域其实就等于默认给使用的控件设置了padding，控件的内容只能显示在内容区域内。

## nine-patch标签

使用nine-patch标签可以对点九图片做一些设置处理，不过可设置的属性并不多：

* android:src 必填项，必须指定点九类型的图片
* android:dither 设置是否抖动，图片与屏幕的像素配置不同时会用到，比如图片是ARGB 8888的，而屏幕是RGB565
* android:tint 给图片着色，比如图片本来是黑色的，着色后可以变成白色
* android:tintMode 着色模式，API Level 21(Android 5.0)才添加的属性
* android:alpha 设置图片的透明度，取值范围为0.0~1.0之间，0.0为全透明，1.0为全不透明，API Level最低要求是11
* android:autoMirrored 设置图片是否需要镜像反转，当布局方向是RTL，即从右到左布局时才有用，API Level 19(Android 4.4)才添加的属性



## color标签

color标签是drawable里最简单的标签了，只有一个属性：android:color，指定颜色值。这个标签一般很少用，因为基本都可以通过其他更方便的方式定义颜色。另外，颜色值一般都在colors.xml文件中定义，其根节点为resources。看看两者的不同：

```
<color xmlns:android="http://schemas.android.com/apk/res/android"
    android:color="#FFFFFF" />
```

```
<resources>
    <color name="white">#FFFFFF</color>
</resources>
```

引用的时候，前一种通过@drawable/white引用，后一种通过@color/white引用。

## inset标签

使用inset标签可以对drawable设置边距，其用法和View的padding类似，只不过padding是设置内容与边界的距离，而inset则可以设置背景drawable与View边界的距离。inset标签的可设置属性如下：

* android:drawable 指定drawable资源，如果不设置该属性，也可以定义drawable类型的子标签
* android:visible 设置初始的可见性状态，默认为false
* android:insetLeft 左边距
* android:insetRight 右边距
* android:insetTop 顶部边距
* android:insetBottom 底部边距
* android:inset 设置统一边距，会覆盖上面四个属性，但API Level要求为21，即Android 5.0

## clip标签

使用clip标签可以对drawable进行裁剪，在做进度条时很有用。通过设置level值控制裁剪多少，level取值范围为0~10000，默认为0，表示完全裁剪，图片将不可见；10000则完全不裁剪，可见完整图片。看看clip标签可以设置的属性：

* android:drawable 指定drawable资源，如果不设置该属性，也可以定义drawable类型的子标签
* android:clipOrientation 设置裁剪的方向，取值为以下两个值之一：

	* horizontal 在水平方向上进行裁剪，条状的进度条就是水平方向的裁剪
	* vertical 在垂直方向上进行裁剪
* android:gravity 设置裁剪的位置，可取值如下，多个取值用 | 分隔：

	* top 图片放于容器顶部，不改变图片大小。当裁剪方向为vertical时，会裁掉图片底部
	* bottom 图片放于容器底部，不改变图片大小。当裁剪方向为vertical时，会裁掉图片顶部
	* left 图片放于容器左边，不改变图片大小，默认值。当裁剪方向为horizontal，会裁掉图片右边部分
	* right 图片放于容器右边，不改变图片大小。当裁剪方向为horizontal，会裁掉图片左边部分
	* center 图片放于容器中心位置，包括水平和垂直方向，不改变图片大小。当裁剪方向为horizontal时，会裁掉图片左右部分；当裁剪方向为vertical时，会裁掉图片上下部分
	* fill 拉伸整张图片以填满容器的整个高度和宽度。这时候图片不会被裁剪，除非level设为了0，此时图片不可见
	* center_vertical 图片放于容器垂直方向的中心位置，不改变图片大小。裁剪和center时一样
	* center_horizontal 图片放于容器水平方向的中心位置，不改变图片大小。裁剪和center时一样
	* fill_vertical 在垂直方向上拉伸图片以填满容器的整个高度。当裁剪方向为vertical时，图片不会被裁剪，除非level设为了0，此时图片不可见
	* fill_horizontal 在水平方向上拉伸图片以填满容器的整个宽度。当裁剪方向为horizontal时，图片不会被裁剪，除非level设为了0，此时图片不可见
	* clip_vertical 附加选项，裁剪基于垂直方向的gravity设置，设置top时会裁剪底部，设置bottom时会裁剪顶部，其他情况会同时裁剪顶部和底部
	* clip_horizontal 附加选项，裁剪基于水平方向的gravity设置，设置left时会裁剪右侧，设置right时会裁剪左侧，其他情况会同时裁剪左右两侧
	
那怎么设置level呢？android没有提供直接在xml里设置level的属性，这需要通过代码去设置。举例用法如下：

1. 定义clip.xml：

```
<?xml version="1.0" encoding="utf-8"?>
<clip xmlns:android="http://schemas.android.com/apk/res/android"
    android:clipOrientation="horizontal"
    android:drawable="@drawable/img4clip"
    android:gravity="left" />
```

2. 在ImageView中引用：

```
<ImageView
    android:id="@+id/img"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:background="@drawable/bg_img"
    android:src="@drawable/clip" />
```

3. 在代码中设置level：

```
ImageView img =  (ImageView) findViewById(R.id.img);
img.getDrawable().setLevel(5000); //level范围值0~10000 
```

## scale标签

使用scale标签可以对drawable进行缩放操作，和clip一样是通过设置level来控制缩放的比例。scale标签可以设置的属性如下：

* android:drawable 指定drawable资源，如果不设置该属性，也可以定义drawable类型的子标签
* android:scaleHeight 设置可缩放的高度，用百分比表示，格式为XX%，0%表示不做任何缩放，50%表示只能缩放一半
* android:scaleWidth 设置可缩放的宽度，用百分比表示，格式为XX%，0%表示不做任何缩放，50%表示只能缩放一半
* android:scaleGravity 设置drawable缩放后的位置，取值和bitmap标签的一样，就不一一列举说明了，不过默认值是left
* android:useIntrinsicSizeAsMinimum 设置drawable原有尺寸作为最小尺寸，设为true时，缩放基本无效，API Level最低要求为11

使用的时候，和clip一样，用法如下：

1. 定义scale.xml：

```
<?xml version="1.0" encoding="utf-8"?>
<scale xmlns:android="http://schemas.android.com/apk/res/android"
    android:drawable="@drawable/img4scale"
    android:scaleGravity="left"
    android:scaleHeight="50%"
    android:scaleWidth="50%"
    android:useIntrinsicSizeAsMinimum="false" />
```

2. 在ImageView中引用：

```
<ImageView
    android:id="@+id/img"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:background="@drawable/bg_img"
    android:src="@drawable/scale" />
```

3. 在代码中设置level：

```
ImageView img =  (ImageView) findViewById(R.id.img);
img.getDrawable().setLevel(5000); //level范围值0~10000 
```

# level-list标签

当需要在一个View中显示不同图片的时候，比如手机剩余电量不同时显示的图片不同，level-list就可以派上用场了。level-list可以管理一组drawable，每个drawable设置一组level范围，最终会根据level值选取对应的drawable绘制出来。level-list通过添加item子标签来添加相应的drawable，其下的item只有三个属性：

* android:drawable 指定drawable资源，如果不设置该属性，也可以定义drawable类型的子标签
* android:minLevel 该item的最小level值
* android:maxLevel 该item的最大level值

以下是示例代码

```
<?xml version="1.0" encoding="utf-8"?>
<level-list xmlns:android="http://schemas.android.com/apk/res/android">
    <item
        android:drawable="@drawable/battery_low"
        android:maxLevel="10"
        android:minLevel="0" />
    <item
        android:drawable="@drawable/battery_below_half"
        android:maxLevel="50"
        android:minLevel="10" />
    <item
        android:drawable="@drawable/battery_over_half"
        android:maxLevel="99"
        android:minLevel="50" />
    <item
        android:drawable="@drawable/battery_full"
        android:maxLevel="100"
        android:minLevel="100" />
</level-list>
```

那么，当电量剩下10%时则可以设置level值为10，将会匹配第一张图片：

`img.getDrawable().setLevel(10);`

item的匹配规则是从上到下的，当设置的level值与前面的item的level范围匹配，则采用。一般item的添加按maxLevel从小到大排序下来，此时minLevel可以不用指定也能匹配到。如上面代码就可以简化如下：

```
<?xml version="1.0" encoding="utf-8"?>
<level-list xmlns:android="http://schemas.android.com/apk/res/android">
    <item
        android:drawable="@drawable/battery_low"
        android:maxLevel="10" />
    <item
        android:drawable="@drawable/battery_below_half"
        android:maxLevel="50" />
    <item
        android:drawable="@drawable/battery_over_half"
        android:maxLevel="99" />
    <item
        android:drawable="@drawable/battery_full"
        android:maxLevel="100" />
</level-list>
```

但不能反过来将android:maxLevel="100"的item放在最前面，那样所有电量都只匹配第一条了。

## transition标签

transition其实是继承自layer-list的，只是，transition只能管理两层drawable，另外提供了两层drawable之间切换的方法，切换时还会有淡入淡出的动画效果。示例代码如下：


```
<?xml version="1.0" encoding="utf-8"?>
<transition xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:drawable="@drawable/on" />
    <item android:drawable="@drawable/off" />
</transition>
```

transition标签生成的Drawable对应的类为TransitionDrawable，要切换时，需要主动调用TransitionDrawable的startTransition()方法，参数为动画的毫秒数，也可以调用reverseTransition()方法逆向切换。

```
((TransitionDrawable)drawable).startTransition(500); //正向切换，即从第一个drawable切换到第二个
((TransitionDrawable)drawable).reverseTransition(500); //逆向切换，即从第二个drawable切换回第一个
```

## rotate标签

使用rotate标签可以对一个drawable进行旋转操作，在shape篇讲环形时最后举了个进度条时就用到了rotate标签。另外，比如你有一张箭头向上的图片，但你还需要一个箭头向下的图片，这时就可以使用rotate将向上的箭头旋转变成一张箭头向下的drawable。
先看看rotate标签的一些属性吧：

* android:drawable 指定drawable资源，如果不设置该属性，也可以定义drawable类型的子标签
* android:fromDegrees 起始的角度度数
* android:toDegrees 结束的角度度数，正数表示顺时针，负数表示逆时针
* android:pivotX 旋转中心的X坐标，浮点数或是百分比。浮点数表示相对于drawable的左边缘距离单位为px，如5; 百分比表示相对于drawable的左边缘距离按百分比计算，如5%; 另一种百分比表示相对于父容器的左边缘，如5%p; 一般设置为50%表示在drawable中心
* android:pivotY 旋转中心的Y坐标
* android:visible 设置初始的可见性状态，默认为false

示例代码如下，目标是将一张箭头向上的图片转180度，转成一张箭头向下的图片：

```
<?xml version="1.0" encoding="utf-8"?>
<rotate xmlns:android="http://schemas.android.com/apk/res/android"
    android:drawable="@drawable/ic_arrow"
    android:fromDegrees="0"
    android:pivotX="50%"
    android:pivotY="50%"
    android:toDegrees="180" />
```

将它引用到ImageView里，发现图片根本没有转变。其实，要让它可以旋转，还需要设置level值。level取值范围为0~10000，应用到rotate，则与fromDegrees~toDegrees相对应，如上面例子的角度范围为0~180，那么，level取值0时，则旋转为0度；level为10000时，则旋转180度；level为5000时，则旋转90度。因为level默认值为0，所以图片没有转变。那么，我们想转180度，其实可以将fromDegrees设为180，而不设置toDegrees，这样，不用再在代码里设置level图片就可以旋转180了。

## animation-list标签

通过animation-list可以将一系列drawable构建成帧动画，就是将一个个drawable，一帧一帧的播放。通过添加item子标签设置每一帧使用的drawable资源，以及每一帧持续的时间。示例代码如下：

```
<?xml version="1.0" encoding="utf-8"?>
<animation-list xmlns:android="http://schemas.android.com/apk/res/android"
    android:oneshot="false">
    <item
        android:drawable="@drawable/anim1"
        android:duration="1000" />
    <item
        android:drawable="@mipmap/anim2"
        android:duration="1000" />
    <item
        android:drawable="@mipmap/anim3"
        android:duration="1000" />
</animation-list>
```

android:oneshot属性设置是否循环播放，设为true时，只播放一轮就结束，设为false时，则会轮询播放。

android:duration属性设置该帧持续的时间，以毫秒数为单位。

animation-list对应的Drawable类为AnimationDrawable，要让动画运行起来，需要主动调用AnimationDrawable的start()方法。另外，如果在Activity的onCreate()方法里直接调用start()方法会没有效果，因为view还没有初始化完成是播放不了动画的。

## animated-rotate

rotate标签只是将原有的drawable转个角度变成另一个drawable，它是静态的。而animated-rotate则会让drawable不停地做旋转动画。
animated-rotate可设置的属性只有四个：

* android:drawable 指定drawable资源，如果不设置该属性，也可以定义drawable类型的子标签
* android:pivotX 旋转中心的X坐标
* android:pivotY 旋转中心的Y坐标
* android:visible 设置初始的可见性状态，默认为false

示例代码：

```
<?xml version="1.0" encoding="utf-8"?>
<animated-rotate xmlns:android="http://schemas.android.com/apk/res/android"
    android:drawable="@drawable/img_daisy"
    android:pivotX="50%"
    android:pivotY="50%"
    android:visible="false" />
```

## 写在最后

至此，drawable资源基本都讲完了，但还不是全部，Android 5.0新增的几个标签：animated-selector、vector、animated-vector、ripple，因为还没弄清楚具体的用法，而且也涉及到Material Design，所以不在本篇讲解，后续做Material Design专题分享的时候会再详细讲解用法。