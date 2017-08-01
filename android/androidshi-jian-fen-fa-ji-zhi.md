小结：

1）事件传递的顺序是先外层容器，之后再是具体的View。

2）OnTouch事件先于OnTouchEvent事件，OnTouchEvent先于OnClick事件。

3）onTouch事件的返回值为true会拦截onTouchEvent事件。

4）onTouchEvent与onClick有关联。

5）父控件onInterceptTouchEvent返回true会拦截子控件的事件。

6）如果enable为false，因为短路onTouch不会执行。

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



