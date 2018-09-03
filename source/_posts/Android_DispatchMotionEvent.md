---
title: Android事件分发机制（一）
---
事件分发机制有点复杂，而且似乎笼罩着一层神秘的面纱。为了揭开它，决定进去源码里面看一看，并把过程记录下来，作为一份笔记。如果对大家理解事件分发机制有所帮助，那是再好不过的事情。首先，将稍微整理事件分发机制中，需要理清的几个问题，然后才开始看源码。
<!--more-->

# 关于事件分发的几个问题

1. 为什么要进行事件分发？

用户在Android系统屏幕上进行操作后，会有相应的事件产生。当产生事件的区域，有多个组件可以响应这个事件时，Android系统需要事件分发机制，来决定该事件传递给哪一个组件进行处理。

2. 什么是事件分发？

是指Android系统对用户行为产生的事件，进行传递处理的过程。

3. 事件分发指的是什么事件？

用户操作行为所产生的MotionEvent事件，具体可以是：ACTION_DOWN, ACTION_UP, ACTION_MOVE, ACTION_CANCEL等。

4. 如何进行事件分发？

采用责任链式的设计模式，事件层层传递，从上往下，再从下往上，寻找最终消费事件的组件，未找到，则将该次事件丢弃。

# 事件分发详解

事件分发的主体主要是ViewGroup和View，虽然ViewGroup也是继承自View，但是在事件的处理上ViewGroup和View所需要考虑的因素不同，处理过程有所不同，因为ViewGroup除了要考虑自己，还需要考虑其中的各个子View。至于Activity，理解了ViewGroup事件分发，Activity差不多也通了。所以，将事件分发分为两类进行分析：

（1） ViewGroup的事件分发。

（2） View的事件分发。

### ViewGroup事件分发

ViewGroup事件分发主要涉及到以下三个方法：

（1）public boolean dispatchTouchEvent(MotionEvent ev)。

（2）public boolean onInterceptTouchEvent(MotionEvent ev)。

（3）public boolean onTouchEvent(MotionEvent ev)。

其中的核心方法是dispatchTouchEvent()，所以从这个方法开始分析。onInterceptTouchEvent()会在dispatchTouchEvent()的执行过程中被调用。onTouchEvent()是ViewGroup父类View的方法，只有当ViewGroup被当成View进行事件分发的时候，才会被调用。

ViewGroup被当成View进行事件分发的情况：

（1）ViewGroup对事件进行了拦截。

（2）事件发生在没有子View的区域。

（3）ViewGroup中所有子View未消费事件，事件被回给了ViewGroup。

开始贴源码，抓关键点进行分析与理解。我们先在源码ViewGroup.java类中，找到public boolean dispatchTouchEvent(MotionEvent ev)这个函数，按从上往下顺序看这个方法。

1. 获取事件后，初始化相关变量，并判断是否进行了拦截
找到dispatchTouchEvent()这个方法后，我们可以先看到下面这几行代码。本来想多贴一点，但是贴在这里密密麻麻，容易让人产生恐惧，而且找不到重点。所以，只贴关键的几行。后面也基本这样，每个重要的地方，只挑关键的几行代码贴出来，并加以说明。
```python
                cancelAndClearTouchTargets(ev);
                resetTouchState();
```
	这两行是取消和清除上一次事件，并重置相关的变量。
```python
                intercepted = onInterceptTouchEvent(ev);
```
	这一行主要在检测ViewGtoup有没有拦截事件。onInterceptTouchEvent()方法，依据它的执行结果，改变intercepted标志的值，这个值将作为事件是否分发给子View的依据。而onInterceptTouchEvent()方法里的代码如下：
```python
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        if (ev.isFromSource(InputDevice.SOURCE_MOUSE)
                && ev.getAction() == MotionEvent.ACTION_DOWN
                && ev.isButtonPressed(MotionEvent.BUTTON_PRIMARY)
                && isOnScrollbarThumb(ev.getX(), ev.getY())) {
            return true;
        }
        return false;
    }
```
这几行是这个方法的所有代码，在开发的时候，可以重写这个方法，进行事件拦截。重写这个方法，让它返回true，表示不分发事件给子View，ViewGroup会被当成View进行事件分发。当intercepted为false时，它后面一大块和子View相关的代码才会被执行。

2. 判断是否取消了这次事件
```python
            // Check for cancelation.
            final boolean canceled = resetCancelNextUpFlag(this)
                    || actionMasked == MotionEvent.ACTION_CANCEL;
```
3. 未被拦截，并且未取消
```python
            if (!canceled && !intercepted) {
            // 省略
            }
```
当if (!canceled && !intercepted)条件满足的时候，开始处理ViewGroup里面的组件。

4. 拿到事件发生位置的点坐标
```python
            final float x = ev.getX(actionIndex);
            final float y = ev.getY(actionIndex);
```
这个地方先拿到坐标，是为了后面判断子View有没有进行事件分发的条件。

5. 收集子View
```python
            final ArrayList<View> preorderedList = buildTouchDispatchChildList();
```
buildTouchDispatchChildList()这个方法，它返回了一个List集合。这个集合里面装着，按Z轴方向从小到大排序的所有子View，即Z值较大的子View放在这个List的后面。

6. 遍历ViewGroup里的子View
```python
            for (int i = childrenCount - 1; i >= 0; i--) {
            //省略
            }
```
上一步已经拿到了一个按Z轴从小到大排好序的子View集合。这里遍历的时候是从后往前，所以，先处理List最后面的子View，即布局中盖在最上面的那个子View。

7. 检测当前拿到的子View是不是处在事件产生的位置
```python
            if (!canViewReceivePointerEvents(child)
                     || !isTransformedTouchPointInView(x, y, child, null)) {
               ev.setTargetAccessibilityFocus(false);
               continue;
            }
```
先检测能否能接受事件，并且和前面拿到的点坐标结合，判断当前遍历到的这个View是否包含这个点坐标。不能接受事件或不包含事件的点坐标，continue，跳过后面的代码，开始下一次循环。如果子View包含这个坐标点的话，帮子View包装一下，赋值给newTouchTarget，然后break，跳出循环，不再处理剩下的子View。因为它表示当前View正在接收处理事件，不需要继续分发给其他View。

那么，事件发生的点坐标在非子View和子View区域时，ViewGroup分别做了什么？

（1）事件的点坐标在非子View区域
遍历所有子View，发现它们都不在事件发生点的位置，for循环里后面的代码都不会执行。最终清空了一下装子View的List，跳出了if (!canceled && !intercepted)后面的语句块，而到了下面这里。
```python
            // Dispatch to touch targets.
            if (mFirstTouchTarget == null) {
                // No touch targets so treat this as an ordinary view.
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } else {
            //省略
            }
```
mFirstTouchTarget为空，表示没有子View能处理这个事件，而开始执行dispatchTransformedTouchEvent()方法，child的入参为null。
其中关键的代码如下：
```python
        if (child == null) {
            handled = super.dispatchTouchEvent(transformedEvent);
        }
```
可以看到，child为空的时候，调用了super.dispatchTouchEvent()，即ViewGroup父类的dispatchTouchEvent()，而ViewGroup的父类是View。此时，ViewGroup被当成了一个View进行事件分发。

（2）事件点坐标在子View区域
事件点坐标在子View区域时，继续循环里面的代码：
```python
            if(dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
            //省略
            }
```

调用dispatchTransformedTouchEvent()方法后，会执行里面的下面这部分代码：
```python
            if (child == null || child.hasIdentityMatrix()) {
                if (child == null) {
                    handled = super.dispatchTouchEvent(event);
                } else {
                    final float offsetX = mScrollX - child.mLeft;
                    final float offsetY = mScrollY - child.mTop;
                    event.offsetLocation(offsetX, offsetY);

                    handled = child.dispatchTouchEvent(event);

                    event.offsetLocation(-offsetX, -offsetY);
                }
                return handled;
            }
```
其中child对象，就是我们当前处理的这个子View。人家有料，所以handled = child.dispatchTouchEvent(event)会执行，开始了它的事件分发，这里也涉及到了View的事件分发，后面再单独介绍。这个事件如果被这个子View消费了，事件传递结束，它下面的其他子View也就没什么事。因为dispatchTransformedTouchEvent()返回值为true的话，break，退出了for循环，后边的子View没有机会。如果返回值为false，那表示当前的活（事件）这个子View干不了，换下一个。如果事件点坐标位置的所有子View都不消费事件，也会执行如下代码：
```python
            // Dispatch to touch targets.
            if (mFirstTouchTarget == null) {
                // No touch targets so treat this as an ordinary view.
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            }
```
这时候和事件发生在ViewGroup的非子View区域时一样，执行了相同的代码块，把ViewGroup当成View来进行事件分发。值得注意的是，此时，这些子View已经进行过分发事件，即它们都会调用各自的dispatchTouchEvent()方法，只是都没有消费事件。





