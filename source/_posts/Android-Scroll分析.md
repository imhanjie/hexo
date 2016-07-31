---
title: Android Scroll分析
date: 2016-03-02 21:51:48
tags: [Android,Notes,Android群英传]
---

> Android群英传第五章总结

<!--more-->

### 一、滑动效果是如何产生的

滑动一个View本质上是移动一个View，原理和动画效果的实现非常相似，都是通过不断改变View坐标来实现这一效果的，所以实现滑动的思想基本是一致的：**当触摸View时，系统记下当前触摸的坐标；当手指移动时，系统记下移动后的触摸点坐标，从而获取到相对于前一次坐标点的偏移量offset，并通过offset来修改View的坐标**，这样不断重复，从而实现滑动过程。实现滑动的基本方法有7种。在实现之前，先来了解一下Android中View视图的一些基本概念。

### 二、Android坐标系和视图坐标系的区别

* **Android坐标系：**将屏幕最左上角的顶点作为坐标系的原点，原点向右为x轴正方向，原点向下为y轴正方向
* **视图坐标系：**原点向右为x轴正方向，原点向下为y轴正方向，不过和Android坐标系不同，原点为其父视图左上角

### 三、触控事件——MotionEvent

> 触控事件MotionEvent在用户交互中，占着举足轻重的定位。下面列出了MotionEvent中封装的一些常用的事件常量(前面三个最最常用)。

``` java
// 单点触摸按下动作
public static final int ACTION_DOWN = 0;
// 单点触摸离开动作
public static final int ACTION_UP = 1;
// 触摸点移动动作
public static final int ACTION_MOVE = 2;
// 触摸动作取消
public static final int ACTION_CANCEL = 3;
// 触摸动作超出边界
public static final int ACTION_OUTSIDE = 4;
// 多点触摸按下动作
public static final int ACTION_POINTER_DOWN = 5;
// 多点离开动作
public static final int ACTION_POINTER_UP = 6;
```
#### Android中，系统提供了非常多的方法来获取坐标值，相对距离等，下面列举了一些很常用的API

##### 第一大类：View提供的获取坐标的方法
* getTop()：View自身的**顶边**到其父布局**顶边**的距离
* getLeft()：View自身的**左边**到其父布局**左边**的距离
* getRight()：View自身的**右边**到其父布局**左边**的距离
* getBottom()：View自身的**底边**到其父布局**顶边**的距离

##### 第二大类：MotionEvent提供的方法
* getX()：获取点击事件距离控件左边的距离，即视图坐标
* getY()：获取点击事件距离控件顶边的距离，即视图坐标
* getRawX()：获取点击事件距离整个屏幕左边的距离，即绝对坐标
* getRawY()：获取点击事件距离整个屏幕顶边的距离，即绝对坐标


### 四、实现滑动的七种方法

> 在介绍方法之前，先给出获取偏移量的代码模板(使用的视图坐标，也可以使用绝对坐标来计算，但是要注意在使用绝对坐标时记得要重置mLastX和mLastY的值)：

``` java
...
private int mLastX;
private int mLastY;
...

@Override
    public boolean onTouchEvent(MotionEvent event) {
        int x = (int) event.getX();
        int y = (int) event.getY();
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN: {
                mLastX = x;
                mLastY = x;
                break;
            }
            case MotionEvent.ACTION_MOVE: {
                // 计算偏移量
                int offsetX = x - mLastX;
                int offsetY = y - mLastY;
                break;
            }
            case MotionEvent.ACTION_UP: {
                break;
            }
        }
        return true;
    }
```

#### 1、layout方法

> 根据offsetX和offsetY来重新调用layout方法

``` java
layout(getLeft() + offsetX, getTop() + offsetY, getRight() + offsetX, getBottom() + offsetY);
```


#### 2、offsetLeftAndRight()和offsetTopAndBottom()

>这个方法相当于系统提供了一个对左右、上下移动的API的封装(和第一种layout方法比，推荐这种)


``` java
// 同时对left和right进行偏移
offsetLeftAndRight(offsetX);
// 同时对top和bottom进行偏移
offsetTopAndBottom(offsetY);
```

#### 3、LayoutParams

> LayoutParams保存了一个View的布局参数，因此在程序中通过改变LayoutParams来动态修改一个布局的位置参数，从而达到改变View位置的效果

``` java
ViewGroup.MarginLayoutParams layoutParams = (ViewGroup.MarginLayoutParams) getLayoutParams();
layoutParams.leftMargin = getLeft() + offsetX;
layoutParams.topMargin = getTop() + offsetY;
setLayoutParams(layoutParams);
```

#### 4、scrollTo和scrollBy

> scrollTo和scrollBy移动的是其父布局大小的面板(可视区域)，可以理解为手机后面有一块大大的画布，scrollTo和scrollBy移动的时手机面板，后面的画布保持不动，所以如果将面板向左上移动滑动，后面的画布视图将向右下移动，所以为了和我们手指滑动的方向一致，offsetX和offsetY必须取其对应的负数

``` java
((View) getParent()).scrollBy(-offsetX, -offsetY);
```

#### 5、Scroller

使用Scroller的步骤：

* 初始化Scroller

``` java
mScroller = new Scroller(context);
```

* 重写computeScroll()方法，实现模拟滑动

``` java
    @Override
    public void computeScroll() {
        super.computeScroll();
        if (mScroller.computeScrollOffset()) {
            int currX = mScroller.getCurrX();
            int currY = mScroller.getCurrY();
            ((View) getParent()).scrollTo(currX, currY);
			// 再次触发computeScroller()方法
            invalidate();
        }
    }
```

* startScroll开启模拟过程

``` java
            case MotionEvent.ACTION_UP: {
                int scrollX = ((View) getParent()).getScrollX();
                int scrollY = ((View) getParent()).getScrollY();
                mScroller.startScroll(
                        scrollX,
                        scrollY,
                        -scrollX,
                        -scrollY);
                Log.d("bingo", "UP:scrollX:" + scrollX + "   scrollY" + scrollY);
				// 触发computeScroller()方法
                invalidate();
                break;
            }
```

#### 6、属性动画

#### 7、ViewDragHelper

> support库中的DrawerLayout和SlidingPaneLayout两个强大的布局背后就是靠ViewDragHelper来实现的，通过ViewDragHelper，基本可以实现各种不同的滑动、拖放需求，因此这个方法也是各种滑动解决方案中的终极绝

下面通过一个简单仿Android手机QQ的侧滑菜单的实现来基本使用下ViewDragHelper，详细的解释中代码已经给出注释

``` java
public class DragViewGroup extends FrameLayout {

    private ViewDragHelper mViewDragHelper;
    private View mMenuView;
    private View mMainView;
    private int mWidth;

    public DragViewGroup(Context context) {
        super(context);
    }

    public DragViewGroup(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public DragViewGroup(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        initView();
    }

    private void initView() {
        /**
         * 初始化ViewDragHelper，第一个参数是要监听的View，通常是一个
         * ViewGroup(此类也是继承FrameLayout的)，第二个参数是一个Callback回调，
         * 这个回调就是整个ViewDragHelper的逻辑核心
         */
        mViewDragHelper = ViewDragHelper.create(this, callback);
    }

    @Override
    protected void onFinishInflate() {
        super.onFinishInflate();
        mMenuView = getChildAt(0);
        mMainView = getChildAt(1);
    }

    @Override
    protected void onSizeChanged(int w, int h, int oldw, int oldh) {
        super.onSizeChanged(w, h, oldw, oldh);
        mWidth = mMenuView.getMeasuredWidth();
    }

    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        /**
         * 拦截事件：将事件传递给ViewDragHelper进行处理
         */
        return mViewDragHelper.shouldInterceptTouchEvent(ev);
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        /**
         * 拦截事件：将触摸事件传递给ViewDragHelper，此操作必不可少
         */
        mViewDragHelper.processTouchEvent(event);
        return true;
    }

    /**
     * 处理computeScroll()：因为ViewDragHelper内部也是通过Scroller来实现平滑移动的
     * (一般写下面的代码模板即可)
     */
    @Override
    public void computeScroll() {
        if (mViewDragHelper.continueSettling(true)) {
            ViewCompat.postInvalidateOnAnimation(this);
        }
    }

    /**
     * 处理回调Callback
     */
    private ViewDragHelper.Callback callback = new ViewDragHelper.Callback() {

        /**
         * 何时开始检测触摸事件
         */
        @Override
        public boolean tryCaptureView(View child, int pointerId) {
            // 如果当前触摸的child是mMainView时开始检测
            return mMainView == child;// 指定mMainView可以被移动拖动
        }

        /**
         * 处理垂直滑动
         */
        @Override
        public int clampViewPositionVertical(View child, int top, int dy) {
            return 0; // 返回值为0则不发生滑动
        }

        /**
         * 处理水平滑动
         */
        @Override
        public int clampViewPositionHorizontal(View child, int left, int dx) {
            Log.d("bingo", "left:" + left + "  dx:" + dx);
//            if (dx > 0) {
//                return left + 10;
//            } else {
//                return left - 10;
//            }
            /**
             * 返回值可以控制滑动的距离，根据dx可以判断是左滑还是右滑，这里return left
             * 代表滑动和手指滑动的距离一样
             */
            return left;
        }

        /**
         * 拖动结束后调用(手指离开屏幕时调用)
         */
        @Override
        public void onViewReleased(View releasedChild, float xvel, float yvel) {
            // 手指抬起后缓慢移动到指定位置
            if (mMainView.getLeft() < 500) {
                // 关闭菜单
                // 相当于Scroller的startScroll方法
                mViewDragHelper.smoothSlideViewTo(mMainView, 0, 0);
                ViewCompat.postInvalidateOnAnimation(DragViewGroup.this);
            } else {
                // 打开菜单
                mViewDragHelper.smoothSlideViewTo(mMainView, mWidth / 2, 0);
                ViewCompat.postInvalidateOnAnimation(DragViewGroup.this);
            }
        }
    };

}
``` 