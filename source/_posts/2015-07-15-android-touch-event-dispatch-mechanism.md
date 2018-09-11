---
layout: post
title: Android touch event mechanism
date: 2015-07-15
categories:
- android
tags:
- view,
- TouchEvent
---
当我们在自定义view时，除了考虑界面的布局、绘制之外, 通常还要考虑事件的冲突，这就不得不理解view系统touch事件的分发机制
<!-- more -->
## Android事件分发机制结论记录：

* 1、在view的dispatchTouchEvent方法中onTouch方法优先于onTouchEvent执行，onTouch执行的条件是：①onTouchListener不为null、②view处于enable状态，若onTouch返回true，则onTouchEvent就不会执行，若返回false，则会执行；
<!--excerpt-->

* 2、view的onClick方法是在onTouchEvent方法中处理的，若view是CLICKABLE的，则在onTouchEvent中会消费事件；因此，当view设置了setOnClickListener或者setOnLongClickListener时，这个view会消费touch事件；

* 3、ViewGroup默认不拦截事件，即ViewGroup的onInterceptTouchEvent默认返回false；

* 4、在dispatchTouchEvent中若找到一个消费了事件的target，则后续事件会直接发送给target进行处理；

* 5、事件分发顺序：Activity#dispatchTouchEvent -> PhoneWindow(Window)#superDispatchTouchEvent -> DecorView(Framelayout -> ViewGroup)#dispatchTouchEvent -> 子view

* 6、若父容器拦截了事件，即ViewGroup的onInterceptTouchEvent返回了true，则此事件的后续事件都由此父容器处理，且后续事件都不会分发给父容器的onInterceptTouchEvent方法，因为子view不处理，父容器就当成普通view来处理事件(mMotionTarget==null)，普通view默认是没有子view的，所以就不会调用onInterceptTouchEvent；若前一个事件是子view消费的，则后续事件分发到父容器时，是会判断是否截断的，即调用onInterceptTouchEvent，若截断则会给目标target发送一个MotionEvent.ACTION_CANCEL消息；

