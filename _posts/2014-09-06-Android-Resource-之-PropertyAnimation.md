---
layout:     post
title:      Android Resource 之 Property Animation
date:       2014-09-06 13:31:22
summary:    Android Resource
categories: android resource
---

前篇文章说过，Android框架还提供了两种动画体系，前一篇已经总结了视图动画(View Animation)的用法，本篇则接着总结另一种动画体系——属性动画(Property Animation)的用法。

视图动画只能作用于View，而且视图动画改变的只是View的绘制效果，View真正的属性并没有改变。比如，一个按钮做平移的动画，虽然按钮的确做了平移，但按钮可点击的区域并没随着平移而改变，还是在原来的位置。而属性动画则可以改变真正的属性，从而实现按钮平移时点击区域也跟着平移。通俗点说，属性动画其实就是在一定时间内，按照一定规律来改变对象的属性，从而使对象展现出动画效果。

属性动画是在android 3.0引入的动画体系，如果还想适配基本已经灭绝的2.x版本，只好绕道了。
属性动画和视图动画一样，可以通过xml文件定义，不同的是，视图动画的xml文件放于res/anim/目录下，而属性动画的xml文件则放于res/animator/目录下。一个是anim，一个是animator，别搞错了。同样的，在Java代码里引用属性动画的xml文件时，则用R.animator.filename，不同于视图动画，引用时为R.anim.filename。

属性动画主要有三个元素：animator、objectAnimator、set。
相对应的有三个类：ValueAnimator、ObjectAnimator、AnimatorSet。
ValueAnimator是基本的动画类，处理值动画，通过监听某一值的变化，进行相应的操作。ObjectAnimator是ValueAnimator的子类，处理对象动画。AnimatorSet则为动画集，可以组合另外两种动画或动画集。相应的三个标签元素的关系也一样。
样式开发主要还是用xml的形式，所以这里主要还是讲标签的用法。

## animator

animator标签与对应的ValueAnimator类提供了属性动画的核心功能，包括计算动画值、动画时间细节、是否重复等。执行属性动画分两个步骤：

1. 计算动画值
2. 将动画值应用到对象和属性上

ValuAnimiator只完成第一步，即只计算值，要实现第二步则需要在值变化的监听器里自行更新对象属性。
通过animator标签可以很方便的对ValuAnimiator进行设置，可设置的属性如下：

* android:duration 动画从开始到结束持续的时长，单位为毫秒
* android:startOffset 设置动画执行之前的等待时长，单位为毫秒
* android:repeatCount 设置动画重复执行的次数，默认为0，即不重复；可设为-1或infinite，表示无限重复
* android:repeatMode 设置动画重复执行的模式，可设为以下两个值其中之一：

	* restart 动画重复执行时从起点开始，默认为该值
	* reverse 动画会反方向执行
* android:valueFrom 动画开始的值，可以为int值、float值或color值

* android:valueTo 动画结束的值，可以为int值、float值或color值

* android:valueType 动画值类型，若为color值，则无需设置该属性

	* intType 指定动画值，即以上两个value属性的值为整型
	* floatType 指定动画值，即以上两个value属性的值为浮点型，默认值
* android:interpolator 设置动画速率的变化，比如加速、减速、匀速等，需要指定Interpolator资源。具体用法在View Animation篇已经讲过，这里不再重复

接着，用一个实例讲解具体的用法吧。在这个例子里，将一个按钮的宽度进行缩放，从100%缩放到20%。
xml文件的代码如下：

```
<!-- res/animator/value_animator.xml -->
<?xml version="1.0" encoding="utf-8"?>
<animator xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="3000"
    android:valueFrom="100"
    android:valueTo="20"
    android:valueType="intType" />
```

可看到，值的变化从100到20，动画时长3000毫秒，以下则是目标按钮的xml代码：

```
<Button
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:background="@drawable/bg_btn_normal"
    android:onClick="onScaleWidth"
    android:text="点我"
    android:textColor="@android:color/white" />
```

按钮默认是填充屏幕宽度的，点击时的执行方法为onScaleWidth，以下则是onScaleWidth方法的代码：

```
public void onScaleWidth(final View view) {
    // 获取屏幕宽度
    final int maxWidth = getWindowManager().getDefaultDisplay().getWidth();
    ValueAnimator valueAnimator = (ValueAnimator) AnimatorInflater.loadAnimator(this, R.animator.value_animator);
    valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
        @Override
        public void onAnimationUpdate(ValueAnimator animator) {
            // 当前动画值，即为当前宽度比例值
            int currentValue = (Integer) animator.getAnimatedValue();
            // 根据比例更改目标view的宽度
            view.getLayoutParams().width = maxWidth * currentValue / 100;
            view.requestLayout();
        }
    });
    valueAnimator.start();
}
```

从View Animation篇中已经知道，视图动画是通过AnimationUtils类的loadAnimation()方法获取xml文件相对应的Animation类实例，而属性动画则是通过AnimatorInflater类的loadAnimation()方法获取相应的Animator类实例。

另外，ValueAnimator通过添加AnimatorUpdateListener监听器监听值的变化，从而再手动更新目标对象的属性。

最后，通过调用valueAnimator.start()方法启动动画。

## objectAnimator

objectAnimator标签对应的类为ObjectAnimator，为ValueAnimator的子类。objectAnimator标签与animator标签不同的是，objectAnimator可以直接指定动画的目标对象的属性。标签可设置的属性除了和<animator>一样的那些，另外多了一个：

* android:propertyName 目标对象的属性名，要求目标对象必须提供该属性的setter方法，如果动画的时候没有初始值，还需要提供getter方法

还是用实例说明具体用法，还是用上面的例子，将一个按钮的宽度进行缩放，从100%缩放到20%，但这次改用<objectAnimator>实现。
以下为xml文件的代码：

```
<!-- res/animator/object_animator.xml -->
<?xml version="1.0" encoding="utf-8"?>
<objectAnimator xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="3000"
    android:propertyName="width"
    android:valueFrom="100"
    android:valueTo="20"
    android:valueType="intType" />
```

与animator的例子相比，就只是多了一个android:propertyName的属性，设置值为width。也就是说，动画改变的属性为width，值将从100逐渐减到20。另外，值是从setWidth()传递过去的，再从getWidth()获取。而且，这里设置的值代表的是比例值，因此，还需要进行计算转化为实际的宽度值。最后，对象实际的宽度值为view.getLayoutParams().width。因此，我将用一个包装类来包装原始的view对象，对其提供setWidth()和getWidth()方法，代码如下：

```
private static class ViewWrapper {
    private View target; //目标对象
    private int maxWidth; //最长宽度值

    public ViewWrapper(View target, int maxWidth) {
        this.target = target;
        this.maxWidth = maxWidth;
    }

    public int getWidth() {
        return target.getLayoutParams().width;
    }

    public void setWidth(int widthValue) {
        //widthValue的值从100到20变化
        target.getLayoutParams().width = maxWidth * widthValue / 100;
        target.requestLayout();
    }
}
```

上面setWidth()的代码里，根据比例值转化为了实际的宽度值。最后，动画处理的代码如下：

```
public void onScaleWidth(View view) {
    // 获取屏幕宽度
    int maxWidth = getWindowManager().getDefaultDisplay().getWidth();
    // 将目标view进行包装
    ViewWrapper wrapper = new ViewWrapper(view, maxWidth);
    // 将xml转化为ObjectAnimator对象
    ObjectAnimator objectAnimator = (ObjectAnimator) AnimatorInflater.loadAnimator(this, R.animator.object_animator);
    // 设置动画的目标对象为包装后的view
    objectAnimator.setTarget(wrapper);
    // 启动动画
    objectAnimator.start();
}
```

ObjectAnimator提供了属性的设置，但相应的需要有该属性的setter和getter方法。而ValueAnimator则只是定义了值的变化，并不指定目标属性，所以也不需要提供setter和getter方法，但只能在AnimatorUpdateListener监听器里手动更新属性。不过，也因为没有指定属性，所以其实更具灵活性了，你可以在监听器里根据值的变化做任何事情，比如更新多个属性，比如在缩放宽度的同时做垂直移动。

为了对View更方便的设置属性动画，Android系统也提供了View的一些属性和相应的setter和getter方法：

* alpha：透明度，默认为1，表示不透明，0表示完全透明
* pivotX 和 pivotY：旋转的轴点和缩放的基准点，默认是View的中心点
* scaleX 和 scaleY：基于pivotX和pivotY的缩放，1表示无缩放，小于1表示收缩，大于1则放大
* rotation、rotationX 和 rotationY：基于轴点(pivotX和pivotY)的旋转，rotation为平面的旋转，rotationX和rotationY为立体的旋转
* translationX 和 translationY：View的屏幕位置坐标变化量，以layout容器的左上角为坐标原点
* x 和 y：View在父容器内的最终位置，是左上角坐标和偏移量（translationX，translationY）的和

## set
set标签对应于AnimatorSet类，可以将多个动画组合成一个动画集，如上面提到的在缩放宽度的同时做垂直移动，可以将一个缩放宽度的动画和一个垂直移动的动画组合在一起。
set标签有一个属性可以设置动画的时序关系：

* android:ordering 设置动画的时序关系，取值可为以下两个值之一：
	* together 动画同时执行，默认值
	* sequentially 动画按顺序执行
	
那如果想有些动画同时执行，有些按顺序执行，该怎么办呢？因为<set>标签是可以嵌套其他<set>标签的，也就是说可以将同时执行的组合在一个<set>标签，再嵌在按顺序执行的<set>标签内。

看实例代码吧，以下为xml文件：

```
<!-- res/animator/animator_set.xml -->
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android"
    android:ordering="together">
    <objectAnimator
        android:duration="3000"
        android:propertyName="width"
        android:valueFrom="100"
        android:valueTo="20"
        android:valueType="intType" />
    <objectAnimator
        android:duration="3000"
        android:propertyName="marginTop"
        android:valueFrom="0"
        android:valueTo="100"
        android:valueType="intType" />
</set>
```

以上代码可实现两个同时执行的动画，一个将width从100缩放到20，一个将marginTop从0增加到100。多了一个marginTop属性，那么，在ViewWrapper添加setMarginTop()方法，添加后的ViewWrapper类代码如下：

```
private static class ViewWrapper {
    private View target;
    private int maxWidth;

    public ViewWrapper(View target, int maxWidth) {
        this.target = target;
        this.maxWidth = maxWidth;
    }

    public int getWidth() {
        return target.getLayoutParams().width;
    }

    public void setWidth(int widthValue) {
        target.getLayoutParams().width = maxWidth * widthValue / 100;
        target.requestLayout();
    }

    public void setMarginTop(int margin) {
        LinearLayout.LayoutParams layoutParams = (LinearLayout.LayoutParams) target.getLayoutParams();
        layoutParams.setMargins(0, margin, 0, 0);
        target.setLayoutParams(layoutParams);
    }
}
```

最后，动画处理的代码：

```
public void onScaleWidth(View view) {
    // 获取屏幕宽度
    int maxWidth = getWindowManager().getDefaultDisplay().getWidth();
    // 将目标view进行包装
    ViewWrapper wrapper = new ViewWrapper(view, maxWidth);
    // 将xml转化为ObjectAnimator对象
    AnimatorSet animatorSet = (AnimatorSet) AnimatorInflater.loadAnimator(this, R.animator.animator_set);
    // 设置动画的目标对象为包装后的view
    animatorSet.setTarget(wrapper);
    // 启动动画
    animatorSet.start();
}
```

这样就搞定了，实现了宽度缩放和垂直移动的效果。