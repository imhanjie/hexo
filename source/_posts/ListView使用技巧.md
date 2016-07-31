---
title: ListView使用技巧
date: 2016-02-09 13:35:10
tags: [Android,Note,Android群英传]
---

Android群英传-第4章ListView使用技巧 总结

<!--more-->

+ 1、设置Item之间的分隔线
``` java
android:divider="your color"(设置分隔线透明可以@null)
android:dividerHeight="10dp"
```
+ 2、隐藏ListView滚动条
``` java
android:scrollbars="none"
```
+ 3、取消ListView的Item点击效果
``` java
android:listSelector="#00000000"
```
+ 4、设置ListView需要显示在第几项
``` java
mListView.setSelection(position);
```
这个方法类似scrollTo，是瞬间完成移动的，如果需要平滑移动，可以这样：
``` java
mListView.smoothScrollBy(distance,duration);
mListView.smoothScrollByOffset(offset);
mListView.smoothScrollToPosition(index);
```
+ 5、遍历ListView中的所有Item
``` java
for(int i = 0; i < mListView.getChildCount(); i++){
	View view = mListView.getChildAt(i);
}
```
+ 6、ListView为空时(没有数据)显示对应的提示View
``` java
mListView.setEmptyView(your empty view);
```
+ 7、OnScrollListener内的onScroll()回调方法内的四个参数(onScroll在滑动时一直调用)
	+ firstVisibleItem：当前能看见的第一个Item的ID(从0开始)
	+ visibleItemCount：当前能看见的Item总数(包括没显示完整的item)
	+ totalItemCount：整个ListView的Item总数

小应用1：
``` java
if(firstVisibleItem + visibleItemCount == totalItemCount && totalItemCount > 0){
	// 滚动到最后一行
}
```
小应用2：
``` java
if(firstVisibleItem > lastVisibleItemPosition){
	// 上滑
}else if(firstVisibleItem  < lastVisibleItemPosition){
	// 下滑
}
// lastVisibleItemPosition可以是定义的成员变量来记录上一次第一个可视的item的id
```
另外ListView还提供了一个封装好的2个方法：
``` java
// 获取可视区域内最后一个Item的id
mListView.getLastVisiblePosition();
// 获取可视区域内第一个Item的id
mListView.getFirstVisiblePosition();
```

+ 8、ViewConfiguration.get(this).getScaledTouchSlop();这个方法可以获取系统认为产生了滑动的像素距离(正值)

+ 9、在BaseAdapter中除了几个常见的可覆盖的方法，还有2个比较有意思的方法：
``` java
@Override
public int getItemViewType(int position){
	return type;
}
// 这个方法可以根据position获取对应的item的type，有什么用呢？可以的getView方法内调用getItemViewType(position)方法，获取对应position的类型，例如可以根据类型加载不同的布局文件(类似聊天对话框)的实现。
@Override
public int getViewTypeCount(){
	return count;
}
// 这个方法和上面的方法是对应的，一共有多少种类型，就返回对应的count值，例如类似普通的聊天对话框，这里就可以返回2(in,out)
```





