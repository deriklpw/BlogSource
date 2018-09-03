---
title: Android事件分发机制（二）
---

ViewGroup进行事件分发的过程中，多次把事件传递给了子View，开始View的事件分发。那么，View的事件分发如何进行？
<!--more-->
关于View事件分发的几个问题：

1. View进行事件分发的目的？

确定事件能否被消费，以及响应事件具体的类型。

2. View什么情况下会消费事件？

（1）View设置的OnTouchListener，返回true。

（2）重写View的onTouchEvent()方法时，返回true。

（3）View为可点击状态，默认消费事件。例如Button，CheckBox，ImageButton等。

3. View进行事件响应的顺序？

OnTouchListener --> onTouchEvent --> OnLongClickListener --> OnClickListener。

### View的事件分发

View的事件分发主要涉及两个方法：

（1）public boolean dispatchTouchEvent(MotionEvent event)

（2）public boolean onTouchEvent(MotionEvent event)

1. 检测是否设置OnTouchListener
```python
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }
```
先检测View是否有设置OnTouchListener，如果有设置，调用监听器的回调方法。回调方法返回true，消费事件，即view.dispatchTouchEvent()值为true，相当于告诉ViewGroup，当前事件被消费了。

2. 调用View自身的onTouchEvent()方法
```python
            if (!result && onTouchEvent(event)) {
                result = true;
            }
```
View未设置OnTouchListener，或OnTouchListener中onTouch()回调方法返回值为false，则继续调用View自身的onTouchEvent()方法。如果onTouchEvent()方法返回true，即消费了事件，返回false，未消费事件。

事件传递给onTouchEvent()方法后，需要在其中确认事件类型的响应顺序，下面进入这个方法：

（1）判断View使能
```python
        if ((viewFlags & ENABLED_MASK) == DISABLED) {
            if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
                setPressed(false);
            }
            // A disabled view that is clickable still consumes the touch
            // events, it just doesn't respond to them.
            return (((viewFlags & CLICKABLE) == CLICKABLE
                    || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
                    || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE);
        }
```
View为DISABLED，被禁用，且不可点击时，直接返回false，告诉ViewGroup，自己没有消费事件。但如果被禁用，却是可点击的（点击，长点击等），依然会返回true。这种情况下View什么也没做，依然把事件消费了。

（2）View未被禁用，继续处理事件
```python
if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {
//省略
return true;
}
```
只要View是clickable，或者悬停、长按时可显示工具提示，就会返回true，消费事件。

（3）关于点击和长按事件的处理
```python
                        if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                            // This is a tap, so remove the longpress check
                            removeLongPressCallback();

                            // Only perform take click actions if we were in the pressed state
                            if (!focusTaken) {
                                // Use a Runnable and post this rather than calling
                                // performClick directly. This lets other visual state
                                // of the view update before click actions start.
                                if (mPerformClick == null) {
                                    mPerformClick = new PerformClick();
                                }
                                if (!post(mPerformClick)) {
                                    performClick();
                                }
                            }
                        }
```
在手指抬起，View收到ACTION_UP时，先检测是否是长按。如果不是长按，则执行removeLongPressCallback()，移除长按的CheckForLongPress对象，系统不会对长按进行响应。否则，执行performClick()，开始调用点击事件监听器中的回调方法，进行事件响应。

# 总结
ViewGroup拿到事件后，先调用了自己的onInterceptTouchEvent()判断是否要进行拦截。如果拦截，则会将ViewGroup当成View进行事件分发。不拦截，则遍历其中的子View，挨个询问里面的子View有没有要消费事件的。消费，则事件传递结束，不消费，传递给下一个View，都不消费，则又传回到ViewGroup，把自己当成View进行事件分发。在ViewGroup当成View进行事件分发的时候，若没有消进行消费，这次事件被丢弃。事件传递的整个过程看起来像一个回路，是一个经典的责任链式设计模式。

