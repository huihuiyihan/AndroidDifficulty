---
title: 【View工作原理】ViewRoot、DecorView、MeasureSpec和LayoutParams
date: 2016-06-07 19:28
tags:
  - View
  - Window
---

## 一、窗口层级关系

### 1、PhoneWindow
 - 是Android中最基本的窗口系统，每个Activity会创建并持有一个PhoneWindow对象，是Activity和整个View系统交互的接口。

### 2、DecorView
  - 1、Dispatch ViewRoot分发来的key、touch、trackball等外部事件。
  - 2、DecorView有一个直接的子View，我们称之为System Layout,这个View是从系统的Layout.xml中解析出的，它包含当前UI的风格，如是否带title、是否带process bar等。
  - 3、作为PhoneWindow与ViewRoot之间的桥梁，ViewRoot通过DecorView设置窗口属性。

<!--more-->

### 3、System Layout
 - 目前android根据用户需求预设了几种UI 风格，通过PhoneWindow通过解析预置的layout.xml来获得包含有不同Window decorations的layout，我们称之为System Layout，我们将这个System Layout添加到DecorView中。
 - 预设风格可以通过PhoneWindow方法requestFeature()来设置，需要注意的是这个方法需要在setContentView()方法调用之前调用。

### 4、Content Parent
 - Content Parent这个ViewGroup对象才是真真正正的ContentView的parent，我们的ContentView终于找到了寄主，它其实对应的是System Layout中的id为“content”的一个FrameLayout。这个FrameLayout对象包括的才是我们的Activity的layout。

### 5、Activity Layout
 - ActivityLayout就是我们通过SetContentView设置的Layout，是真正和用户交互的部分。

![这里写图片描述](http://hi.csdn.net/attachment/201111/10/0_1320932280LPp2.gif)

----------

## 二、ViewRoot

 - ViewRoot是GUI管理系统与GUI呈现系统之间的桥梁，根据ViewRoot的定义，我们发现它并不是一个View类型，而是一个Handler。

 - 它的主要作用如下：
  - 1、向DecorView分发收到的用户发起的event事件，如按键，触屏，轨迹球等事件；
  - 2、与WindowManagerService交互，完成整个Activity的GUI的绘制。View的三个流程都是通过ViewRoot来完成的。


----------

## 三、MeasureSpec

 - 一个MeasureSpec封装了父布局传递给子布局的布局要求，每个MeasureSpec代表了一组宽度和高度的要求。一个MeasureSpec由大小和模式组成。它有三种模式：UNSPECIFIED(未指定),父元素部队自元素施加任何束缚，子元素可以得到任意想要的大小；EXACTLY(完全)，父元素决定自元素的确切大小，子元素将被限定在给定的边界里而忽略它本身大小；AT_MOST(至多)，子元素至多达到指定大小的值。

 - 它常用的三个函数：
  - 1、static int getMode(int measureSpec):根据提供的测量值(格式)提取模式(上述三个模式之一)
  - 2、static int getSize(int measureSpec):根据提供的测量值(格式)提取大小值(这个大小也就是我们通常所说的大小)
  - 3、static int makeMeasureSpec(int size,int mode):根据提供的大小值和模式创建一个测量值(格式)


----------

## 四、LayoutParams

 - LayoutParams封装的是View向父容器传达自己意愿的信息，它封装了Layout的位置、高、宽等信息，表达自己想变成一个什么尺寸的View。

 - 使用示例：

``` Java
SwipeRefreshLayout refreshLayout = new NsRefreshLayout(this);
refreshLayout.setLayoutParams(new FrameLayout.LayoutParams(FrameLayout.LayoutParams.MATCH_PARENT, FrameLayout.LayoutParams.MATCH_PARENT));
ScrollView scrollView = new ScrollView(this);
TextView textView = new TextView(this);
textView.setText("TextView");
scrollView.addView(textView, new FrameLayout.LayoutParams(FrameLayout.LayoutParams.MATCH_PARENT, FrameLayout.LayoutParams.MATCH_PARENT));
refreshLayout.addView(scrollView, new FrameLayout.LayoutParams(FrameLayout.LayoutParams.MATCH_PARENT, FrameLayout.LayoutParams.MATCH_PARENT));
```
