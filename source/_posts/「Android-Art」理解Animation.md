title: 「Android-Art」理解Animation
date: 2016-07-27 10:05:33
tags: [Android,Notes,Android开发艺术探索]
---

理解Android中View动画、帧动画和属性动画的使用和原理

<!--more-->

### 以下整理自「Android开发艺术探索」第七章 ###

---

Android中的动画可以分为三种：
- View动画
	- 通过对场景里的对象不断做图像变换（平移、缩放、旋转、透明度）从而产生动画效果，他是一种渐近式动画，并且View动画支持自定义
- 帧动画
	- 通过顺序播放一系列图像从而产生动画效果，可以简单理解为图片切换动画，很显然，如果图片过多过大就会导致OOM
- 属性动画
	- 通过动态改变对象的属性从而达到动画效果，属性动画为API11的新特性，再低版本要使用属性的动画需要使用库

> 其实帧动画也属于View动画的一种，只不过他的平移、旋转等常见的View动画在表现形式上略有不同而已

## 一、View动画： ##

##### 种类： #####

1. TranslateAniamtion
2. ScaleAnimation
3. RotateAnimation
4. AlphaAnimation

这四种动画既可以通过XML定义，也可以通过代码来动态创建，对于View动画，建议采用XML来定义，应为XML格式的动画可读性更好

|名称|标签|子类|效果|
|-|-|-|-|
|平移动画|&lt;translate&gt;|TranslateAnimation|移动View|
|缩放动画|&lt;scale&gt;|ScaleAnimation|放大或缩小View|
|旋转动画|&lt;rotate&gt;|RotateAnimation|旋转View|
|透明度动画|&lt;alpha&gt;|AlphaAnimation|改变View的透明度|

xml路径：res/anim/file_name.xml

``` xml
<set
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:interpolator="@[package:]anim/interpolator_resource"
    android:shareInterpolator="["true" | "false"]">

    <alpha
        android:fromAlpha="float"
        android:toAlpha="float"/>

    <scale
        android:fromXScale="float"
        android:fromYScale="float"
        android:pivotX="float"
        android:pivotY="float"
        android:toXScale="float"
        android:toYScale="float"/>

    <translate
        android:fromXDelta="float"
        android:toXDelta="float"
        android:fromYDelta="float"
        android:toYDelta="float"/>

    <rotate
        android:fromDegrees="float"
        android:toDegrees="float"
        android:pivotX="float"
        android:pivotY="float"/>
    
    ...

</set>
```

从上面的语法可以看出，View动画既可以是单个动画，也可以由一系列动画组成

&lt;set&gt;标签标示动画集合，对应AnimationSet类，他可以包含若干个动画，并且它的内部也是可以嵌套其他动画集合的。

##### 加载XML中的动画并设置给View播放 ######

``` java
Animation animation = AnimationUtils.loadAnimation(this, R.anim.anim_demo);
imageView.startAnimation(aniamtion);
```

##### 代码来创建动画 #####

``` java
AlphaAnimation alphaAnimation = new AlphaAnimation(0,1);
alphaAnimation.setDuration(3000);
imageView.startAnimation(alphaAnimation);
```

##### 给animation设置动画监听 #####
``` java
animation.setAnimationListener(new Animation.AnimationListener() {
    @Override
    public void onAnimationStart(Animation animation) {
        
    }

    @Override
    public void onAnimationEnd(Animation animation) {

    }

    @Override
    public void onAnimationRepeat(Animation animation) {

    }
});
```
---

## 二、帧动画 ##

帧动画是顺序播放一组预先定义好的图片，类似于电影播放。不同于View动画，系统提供了另外一个类AnimationDrawable来使用帧动画。帧动画的使用比较简单，首先需要通过XML来定义一个AnimationDrawable，如下所示：
``` xml
<animation-list xmlns:android="http://schemas.android.com/apk/res/android">

    <item
        android:drawable="@drawable/image1"
        android:duration="500"/>

    <item
        android:drawable="@drawable/image2"
        android:duration="500"/>

    <item
        android:drawable="@drawable/image3"
        android:duration="500"/>

</animation-list>
```

##### 播放帧动画 #####
``` java
Button button = (Button)findViewById(R.id.button);
button.setBackgroundResource(R.drawable.fram_animation);
AnimationDrawable drawable = (AnimationDrawable)button.getBackground();
drawable.start();
```

### View动画的特殊使用场景 ###
##### 1、LayoutAnimation #####

例如给ListView的item添加出场动画(对所有的ViewGroup适用)

``` xml
<ListView
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:layoutAnimation="@anim/anim_demo"/>
```

##### 2、Activity的切换效果 #####
``` java
overridePendingTransition(enterAnim, exitAnim);
```
以上的一行代码必须在startActivity()下面执行或者在finish后执行，否则无效果

---

## 三、属性动画 ##

属性动画是API11新加入的特性，和View动画不同，他对作用对象进行了扩展，属性动画可以对任何的对象做动画，升值还可以没有对象。属性动画中有ValueAnimator、ObjectAnimator和AnimatorSet等概念。

##### View动画和属性动画的区别？ #####

- View动画：采用的是矩阵变换来改变View的形态的，而未真实改变View的LayoutParams
- 属性动画：采用反射改变View的属性，真实改变View的LayoutParams，功能也比View动画要强大，不仅局限于类似View动画的平移、旋转、缩放、透明度...


View动画开销小，而属性动画内部采用的反射，所以开销也比View动画大，所以[Android官方文档](https://developer.android.com/guide/topics/graphics/prop-animation.html#property-vs-view)也建议如果View动画能满足需求，可以不用使用属性动画.

> The view animation system, however, takes less time to setup and requires less code to write. If view animation accomplishes everything that you need to do, or if your existing code already works the way you want, there is no need to use the property animation system.

- 属性动画默认时间间隔300ms，默认帧率10ms/帧，即10ms调用一次UpdateListener监听器的回调

将背景从黑色到白色快速闪烁：

``` java
RelativeLayout rootLayout = (RelativeLayout) findViewById(R.id.root_layout);
ValueAnimator colorAnim = ObjectAnimator.ofInt(
        rootLayout,
        "backgroundColor",
        0xFF000000, 0xFFFFFFFF);
colorAnim.setDuration(100);
colorAnim.setEvaluator(new ArgbEvaluator());
colorAnim.setRepeatCount(ValueAnimator.INFINITE);
colorAnim.setRepeatMode(ValueAnimator.REVERSE);
colorAnim.start();
```

属性动画集合：

``` java
ImageView iv = (ImageView) findViewById(R.id.iv);
AnimatorSet set = new AnimatorSet();
// x轴旋转
ObjectAnimator animator1 = ObjectAnimator.ofFloat(iv, "rotationX", 0, -360);
// y轴旋转
ObjectAnimator animator2 = ObjectAnimator.ofFloat(iv, "rotationY", 0, -180);
// 平面旋转
animator2.setRepeatCount(ValueAnimator.INFINITE);
ObjectAnimator animator3 = ObjectAnimator.ofFloat(iv, "rotation", 0, -90);
set.playTogether(
        animator1, animator2, animator3
);
set.setInterpolator(new LinearInterpolator());
set.setDuration(1000);
set.start();
```

另外属性动画也可以用xml文件来表示：
- &lt;set&gt;: 对应AnimatorSet
- &lt;animator&gt;: 对应ValueAnimator
- &lt;objectAnimator&gt;: 对应ObjectAnimator

例如：

``` xml
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android"
     android:ordering="together">

    <objectAnimator
        android:duration="1000"
        android:propertyName="x"
        android:valueTo="200"
        android:valueType="floatType"/>

    <objectAnimator
        android:duration="3000"
        android:propertyName="y"
        android:valueTo="300"
        android:valueType="floatType"/>

</set>
```

应用上面的xml代码：
``` java
AnimatorSet set = AnimatorInflater.loadAnimator(context,R.animator.animator_demo);
set.setTraget(your_view);
set.start();
```

**开发中建议采用代码来实现属性动画**，因为代码实现较为简单。更重要的是，很多时候一个属性的起始值是无法提前确定的，例如一个View的宽度。


### 理解插值器和估值器 ###

**1、TimeInterpolator**：中文翻译为时间插值器，作用是根据时间流逝的百分比计算出当前属性值改、变的百分比，系统预置的插值器有：
- LinearInterpolator: 线性插值器-匀速动画
- AccelerateDecelerateInterpolator: 加速减速插值器-动画两头慢，中间快
- AccelerateInterpolator: 加速插值器-动画越来越快
- DecelerateInterpolator: 减速插值器-动画越来越慢
- [等等](https://developer.android.com/guide/topics/graphics/prop-animation.html)

LinearInterpolator的实现: 匀速动画，输入和输入保持一致(输入input是指时间流逝的百分比)
``` java
public class LinearInterpolator extends BaseInterpolator implements NativeInterpolatorFactory {

    public LinearInterpolator() {
    }

    public LinearInterpolator(Context context, AttributeSet attrs) {
    }

    public float getInterpolation(float input) {
        return input;
    }

	...
   
}
```


**2、TypeEvaluator**：中文翻译为类型估值器(算法)，作用是根据当前属性改变的百分比(上面插值器计算来的)来计算改变后的属性值，系统预置的估值器(算法)有：
- IntEvaluator
- FloatEvaluator
- ArgbEvaluator

IntEvaluator的实现：参数fraction即为插值器计算出来的属性改变的百分比，交由估值器计算应改变的属性值
``` java
public class IntEvaluator implements TypeEvaluator<Integer> {

    public Integer evaluate(float fraction, Integer startValue, Integer endValue) {
        int startInt = startValue;
        return (int)(startInt + fraction * (endValue - startInt));
    }

}
```

** 3、属性动画监听器**

主要有两个接口：
- AnimatorUpdateListener: 位于ValueAnimator内
- AnimatorListener: 位于Animator类内


AnimatorUpdateListener:
``` java
    public static interface AnimatorUpdateListener {

        void onAnimationUpdate(ValueAnimator animation);

    }
```
AnimatorListener:
``` java
    public static interface AnimatorListener {

        void onAnimationStart(Animator animation);

        void onAnimationEnd(Animator animation);

        void onAnimationCancel(Animator animation);

        void onAnimationRepeat(Animator animation);
    }
```


**4、属性动画的工作原理**

属性动画要求动画作用的对象提供该属性的get和set方法，属性动画根据外界传递的属性的初始值和最终值，以动画的效果多次去调用set方法，每次传递给set方法的值都不一样确切来说是随着时间的推移，所传递的值越来越接近最终值。总结一下，我们对object的属性abc做动画，如果想让动画生效，要同时满足两个条件，缺一不可：
- object必须要提供setAbc方法，如果动画的时候没有传统初始值，那么还要提供getAbc方法，因为系统要去取abc属性的初始值(如果这条不满足，程序直接crash)
- object的setAbc对属性abc所做的改变必须能通过某种方式反映出来，比如会带来UI的改变之类的(如果这条不满足，动画无效果但不会crash)

如果上述条件不满足，可以采用以下三种方法解决：
- 1、给你的对象加上get和set方法，如果你有权限的话
- 2、用一个类来包装原始对象，间接为其提供get和set方法
- 3、采用ValueAnimator，监听动画过程，自己实现属性的改变

对于第二个解决办法，提供一个实例，例如对Button包装：
``` java
private static class ViewWrapprer {

    private View mTarget;

    public ViewWrapprer(View target) {
        mTarget = target;
    }

    public int getWidth() {
        return mTarget.getWidth();
    }

    public void setWidth(int width) {
        mTarget.getLayoutParams().width = width;
        mTarget.requestLayout();
    }

}
```

## 四、使用动画需要注意的地方 ##

- 在属性动画若使用无限循环的动画，这类动画需要在Activity退出后及时停止，否则将导致Activity无法释放从而造成内存泄漏，可以调用动画的cancel()方法，View动画则不用担心此为问题
- 使用帧动画图片数量较多且图片较大的时候极易出现OOM，所以尽量避免使用帧动画
- 开启硬件加速会提高动画的流畅性
- 不要使用px，否则不同的设备效果可能会有所不同