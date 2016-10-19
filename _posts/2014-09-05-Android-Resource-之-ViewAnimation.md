---
layout:     post
title:      Android Resource 之 View Animation
date:       2014-09-05 23:31:22
summary:    Android Resource
categories: android resource
---

drawable篇讲过两个动画，animation-list定义帧动画，animated-rotate定义旋转动画，这两个属于drawable动画。除了drawable动画，Android框架还提供了另外两种动画体系：视图动画(View Animation)和属性动画(Property Animation)。视图动画比较简单，只能应用于各种View，可以做一些位置、大小、旋转和透明度的简单转变。属性动画则是在android 3.0引入的动画体系，提供了更多特性和灵活性，也可以应用于任何对象，而不只是View。本篇先讲视图动画。

视图动画可以通过xml文件定义，xml文件放于res/anim/目录下，根元素可以为：<alpha>, <scale>, <translate>, <rotate>, 或者<set>。其中，<set>标签定义的是动画集，它可以包含多个其他标签，也可以嵌套<set>标签。默认情况下，所有动画会同时播放；如果想按顺序播放，则需要指定startOffset属性；另外，还可以通过设置interpolator改变动画变化的速率，比如匀速、加速。

## alpha

alpha可以实现透明度渐变的动画效果，也就是淡入淡出的效果，可通过设置下面三个属性来设置淡入或淡出效果：

* android:duration 动画从开始到结束持续的时长，单位为毫秒
* android:fromAlpha 动画开始时的透明度，0.0为全透明，1.0为不透明，默认为1.0
* android:toAlpha 动画结束时的透明度，0.0为全透明，1.0为不透明，默认为1.0

当设置开始时透明度为0.0，结束时为1.0，就能实现淡入效果；相反，当设置开始时透明度为1.0，结束时为0.0，那就能实现淡出效果。示例代码如下：


```
<!-- res/anim/fade_in.xml -->
<?xml version="1.0" encoding="utf-8"?>
<alpha xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="1000"
    android:fromAlpha="0.0"
    android:toAlpha="1.0" />
```

将这动画效果添加到View上也只需要一行代码：

```
view.startAnimation(AnimationUtils.loadAnimation(this, R.anim.fade_in));
```

如果需要重用这个动画，也可以将其抽离出来。<alpha>标签对应的动画类为AlphaAnimation，父类为Animation，以上代码将AlphaAnimation抽离后的代码可以如下：

```
AlphaAnimation fadeInAnimation = (AlphaAnimation) AnimationUtils.loadAnimation(this, R.anim.fade_in);
view.startAnimation(fadeInAnimation);
```

## scale
scale可以实现缩放的动画效果，主要的属性如下：

* android:duration 动画从开始到结束持续的时长，单位为毫秒
* android:fromXScale 动画开始时X坐标上的缩放尺寸
* android:toXScale 动画结束时X坐标上的缩放尺寸
* android:fromYScale 动画开始时Y坐标上的缩放尺寸
* android:toYScale 动画结束时Y坐标上的缩放尺寸

PS：以上四个属性，0.0表示缩放到没有，1.0表示正常无缩放，小于1.0表示收缩，大于1.0表示放大

* android:pivotX 缩放时的固定不变的X坐标，一般用百分比表示，0%表示左边缘，100%表示右边缘
* android:pivotY 缩放时的固定不变的Y坐标，一般用百分比表示，0%表示顶部边缘，100%表示底部边缘

示例代码如下：

```
<!-- res/anim/zoom_out.xml -->
<?xml version="1.0" encoding="utf-8"?>
<scale xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="1000"
    android:fromXScale="1.0"
    android:fromYScale="1.0"
    android:pivotX="0%"
    android:pivotY="100%"
    android:toXScale="1.5"
    android:toYScale="1.5" />
```

scale标签对应的类为ScaleAnimation，父类也是Animation，添加到View上的用法和AlphaAnimation一样，代码如下：

```
ScaleAnimation zoomOutAnimation = (ScaleAnimation) AnimationUtils.loadAnimation(this, R.anim.zoom_out);
view.startAnimation(zoomOutAnimation);
```

## translate

translate可以实现位置移动的动画效果，可以是垂直方向的移动，也可以是水平方向的移动。坐标的值可以有三种格式：从-100到100，以"%"结束，表示相对于View本身的百分比位置；如果以"%p"结束，表示相对于View的父View的百分比位置；如果没有任何后缀，表示相对于View本身具体的像素值。主要的属性如下：

* android:duration 动画从开始到结束持续的时长，单位为毫秒
* android:fromXDelta 起始位置的X坐标的偏移量
* android:toXDelta 结束位置的X坐标的偏移量
* android:fromYDelta 起始位置的Y坐标的偏移量
* android:toYDelta 结束位置的Y坐标的偏移量

看示例吧，以下代码实现的是从左到右的移动效果，起始位置为相对于控件本身-100%的位置，即在控件左边，与控件本身宽度一致的位置；结束位置为相对于父控件100%的位置，即会移出父控件右边缘的位置。

```
<!-- res/anim/move_left_to_right.xml -->
<?xml version="1.0" encoding="utf-8"?>
<translate xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="2000"
    android:fromXDelta="-100%"
    android:fromYDelta="0"
    android:toXDelta="100%p"
    android:toYDelta="0" />
```

translate标签对应的类为TranslateAnimation，父类也是Animation，添加到View上的代码如下：

```
TranslateAnimation moveAnimation = (TranslateAnimation) AnimationUtils.loadAnimation(this, R.anim.move_left_to_right);
view.startAnimation(moveAnimation);
```

## rotate

rotate可以实现旋转的动画效果，主要的属性如下：

* android:duration 动画从开始到结束持续的时长，单位为毫秒
* android:fromDegrees 旋转开始的角度
* android:toDegrees 旋转结束的角度
* android:pivotX 旋转中心点的X坐标，纯数字表示相对于View本身左边缘的像素偏移量；带"%"后缀时表示相对于View本身左边缘的百分比偏移量；带"%p"后缀时表示相对于父View左边缘的百分比偏移量
* android:pivotY 旋转中心点的Y坐标，纯数字表示相对于View本身顶部边缘的像素偏移量；带"%"后缀时表示相对于View本身顶部边缘的百分比偏移量；带"%p"后缀时表示相对于父View顶部边缘的百分比偏移量

以下示例代码旋转角度从0到360，即旋转了一圈，旋转的中心点都设为了50%，即是View本身中点的位置。

```
<!-- res/anim/rotate_one.xml -->
<?xml version="1.0" encoding="utf-8"?>
<rotate xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="2000"
    android:fromDegrees="0"
    android:toDegrees="360"
    android:pivotX="50%"
    android:pivotY="50%" />
```

rotate标签对应的类为RotateAnimation，父类也是Animation，添加到View上的代码如下：

```
RotateAnimation rotateAnimation = (RotateAnimation) AnimationUtils.loadAnimation(this, R.anim.rotate_one);
view.startAnimation(rotateAnimation);
```

## set

set标签可以将多个动画组合起来，变成一个动画集。比如想将一张图片缩放的同时也做移动，这时候就要用set标签组合缩放动画和移动动画了。示例代码如下：

```
<!-- res/anim/move_and_scale.xml -->
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="2000">
    <translate
        android:fromXDelta="0"
        android:fromYDelta="0"
        android:toXDelta="200%"
        android:toYDelta="0" />
    <scale
        android:fromXScale="1.0"
        android:fromYScale="1.0"
        android:pivotX="0%"
        android:pivotY="100%"
        android:toXScale="1.5"
        android:toYScale="1.5" />
</set>
```

以上代码实现的动画效果为向右移动的同时也同步放大。set标签在视图动画中除了可以组合alpha, scale, translate, rotate这四种标签，也可以嵌套其他set标签。另外，set标签可嵌套的标签元素并不只有这几个，后面谈到属性动画时会再讲其他的标签及用法。

## 通用属性

仔细观察不难发现，以上五个标签都有android:duration属性，这是一个通用的属性，而除了android:duration，还有其他的通用属性，接下来看看都有哪些通用属性以及相应的用法：

* android:duration 动画从开始到结束持续的时长，单位为毫秒
* android:detachWallpaper 设置是否在壁纸上运行，只对设置了壁纸背景的窗口动画(window animation)有效。设为true，则动画只在窗口运行，壁纸背景保持不变
* android:fillAfter 设置为true时，动画执行完后，View会停留在动画的最后一帧；默认为false；如果是动画集，需在<set>标签中设置该属性才有效
* android:fillBefore 设置为true时，动画执行完后，View回到动画执行前的状态，默认即为true
* android:fillEnabled 设置为true时，android:fillBefore的值才有效，否则android:fillBefore会被忽略
* android:repeatCount 设置动画重复执行的次数，默认为0，即不重复；可设为-1或infinite，表示无限重复
* android:repeatMode 设置动画重复执行的模式，可设为以下两个值其中之一：

	* restart 动画重复执行时从起点开始，默认为该值
	* reverse 动画会反方向执行
* android:startOffset 设置动画执行之前的等待时长，毫秒为单位；重复执行时，每次执行前同样也会等待一段时间

* android:zAdjustment 表示被设置动画的内容在动画运行时在Z轴上的位置，取值为以下三个值之一：

	* normal 默认值，保持内容在Z轴上的位置不变
	* top 保持在Z周最上层
	* bottom 保持在Z轴最下层
* android:interpolator 设置动画速率的变化，比如加速、减速、匀速等，需要指定Interpolator资源，后面再详细讲解

PS：set标签还有个android:shareInterpolator属性，设置为true时则可将interpolator应用到所有子元素中

## Interpolator

通过interpolator可以定义动画速率变化的方式，比如加速、减速、匀速等，每种interpolator都是 Interpolator 类的子类，Android系统已经实现了多种interpolator，对应也提供了公共的资源ID，如下表：


|Interpolator class |Resource ID | Description |
|:--------------:|:-----------:|:---------------:|
|AccelerateDecelerateInterpolator	|@android:anim/accelerate_decelerate_interpolator	|在动画开始与结束时速率改变比较慢，在中间的时候加速|
|AccelerateInterpolator |	@android:anim/accelerate_interpolator	|在动画开始时速率改变比较慢，然后开始加速|
|AnticipateInterpolator |	@android:anim/anticipate_interpolator	|动画开始的时候向后然后往前抛|
|AnticipateOvershootInterpolator	| @android:anim/anticipate_overshoot_interpolator |	动画开始的时候向后然后向前抛，会抛超过目标值后再返回到最后的值 |
| BounceInterpolator |	@android:anim/bounce_interpolator|	动画结束的时候会弹跳 |
|CycleInterpolator|	@android:anim/bounce_interpolator	|动画循环做周期运动，速率改变沿着正弦曲线|
|DecelerateInterpolator	|@android:anim/decelerate_interpolator	|在动画开始时速率改变比较快，然后开始减速|
|LinearInterpolator	|@android:anim/decelerate_interpolator	|动画匀速播放|
|OvershootInterpolator|	@android:anim/overshoot_interpolator	|动画向前抛，会抛超过最后值，然后再返回|


如果系统提供的以上Interpolator还不符合你的效果，也可以自定义。自定义的方式有两种，一种是通过继承 Interpolator 父类或其子类；另一种是通过自定义的xml文件，可以更改上表中Interpolator的属性。自定义的xml文件需存放于res/anim/目录下，根标签与上表相应的有九种如下：

* accelerateDecelerateInterpolator在动画开始与结束时速率改变比较慢，在中间的时候加速。没有可更改设置的属性，所以设置的效果和系统提供的一样
* accelerateInterpolator在动画开始时速率改变比较慢，然后开始加速。有一个属性可以设置加速的速率

	* android:factor 浮点值，加速的速率，默认为1
* anticipateInterpolator动画开始的时候向后然后往前抛。有一个属性设置向后拉的值

	* android:tension 浮点值，向后的拉力，默认为2，当设为0时，则不会有向后的动画了
* anticipateOvershootInterpolator动画开始的时候向后然后向前抛，会抛超过目标值后再返回到最后的值。可设置两个属性

	* android:tension 浮点值，向后的拉力，默认为2，当设为0时，则不会有向后的动画了
	* android:extraTension 浮点值，拉力的倍数，默认为1.5(2*1.5)，当设为0时，则不会有拉力了
* bounceInterpolator 动画结束的时候会弹跳。没有可更改设置的属性

* cycleInterpolator 动画循环做周期运动，速率改变沿着正弦曲线。有一个属性设置循环次数

	* android:cycles 整数值，循环的次数，默认为1
* decelerateInterpolator 在动画开始时速率改变比较快，然后开始减速。有一个属性设置减速的速率

	* android:factor 浮点值，减速的速率，默认为1
* linearInterpolator 动画匀速播放。没有可更改设置的属性

* overshootInterpolator 动画向前抛，会抛超过最后值，然后再返回。有一个属性

	* android:tension 浮点值，超出终点后的拉力，默认为2
	
具体用法，就举个示例吧，先定义个interpolator的xml文件，代码如下：

```
<!-- res/anim/my_interpolator.xml -->
<?xml version="1.0" encoding="utf-8"?>
<anticipateOvershootInterpolator 
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:tension="3"
    android:extraTension="2" />
```

接着，将其设置到要应用的动画的android:interpolator属性即可，代码如下：

```
<!-- res/anim/rotate_one.xml -->
<?xml version="1.0" encoding="utf-8"?>
<rotate xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="2000"
    android:fromDegrees="0"
    android:toDegrees="360"
    android:pivotX="50%"
    android:pivotY="50%"
    android:interpolator="@anim/my_interpolator" />
```

## 写在最后

视图动画的应用主要就这些了，比较简单，当然也有其局限性。比如只能应用于View，也只能做渐变、缩放、旋转和移动，以及这些动画的组合。下一篇再详细讲解属性动画，属性动画可以轻而易举的做到许多视图动画做不到的事，比如说图片的翻转。