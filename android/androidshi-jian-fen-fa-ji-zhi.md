小结：

1）事件传递的顺序是先外层容器，之后再是具体的View。

2）OnTouch事件先于OnTouchEvent事件，OnTouchEvent先于OnClick事件。

3）onTouch事件的返回值为true会拦截onTouchEvent事件。

4）onTouchEvent与onClick有关联。

5）父控件onInterceptTouchEvent返回true会拦截子控件的事件。





