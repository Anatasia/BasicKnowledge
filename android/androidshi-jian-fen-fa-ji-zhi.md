总结：

1）事件接收先从父控件到子控件，如果父控件onInterceptTouchEvent为true，则表示拦截事件。

2）dispatchTouchEvent的ACTION\_DOWN事件中，会清除上一次的点击目标列表，且重置disallowIntercept状态为false，表示拦截，但是真正的拦截状态还是靠onInterceptTouchEvent函数的返回值决定。

3）如果为复杂的自定义控件，有滑动事件处理，还需要重写onInterceptTouchEvent。

4）如果onLongClick执行，api 23 默认时间为500毫秒，则onClick不执行。

5）如果onTouch事件返回为true，则会拦截onTouchEvent事件，onClick，onLongClick事件均不在执行。

## 1.分析：

当屏幕接收到点击事件时会首先传递到Activity的dispatchTouchEvent：

```
/**
 * Called to process touch screen events.  You can override this to
 * intercept all touch screen events before they are dispatched to the
 * window.  Be sure to call this implementation for touch screen events
 * that should be handled normally.
 *
 * @param ev The touch screen event.
 *
 * @return boolean Return true if this event was consumed.
 */
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();
    }
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    return onTouchEvent(ev);
}
```

这里执行了三步

1）第一告诉用户ACTION\_DOWN，用户可以复写onUserInteraction来处理点击开始 ；

2）调用了getWindow\(\).superDispatchTouchEvent\(ev\)，这里的getWindow得到是PhoneWindow对象，因此执行的PhoneWindow的superDispatchTouchEvent函数；

3）调用了Activity的onTouchEvent事件。

PhoneWindow的superDispatchTouchEvent又调用了DecorView的superDispatchTouchEvent函数，每一个Activity都有一个PhoneWindow，每一个PhoneWindow都有一个DecorView，DecoView继承自FrameLayout，这里又调用了super.dispatchTouchEvent\(event\)，FrameLayout里面是没有改函数的，所以最终执行的是ViewGroup的dispatchTouchEvent函数。

