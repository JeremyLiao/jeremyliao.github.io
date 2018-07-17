---
layout: post
title: "Android KeyEvent分发过程"
tagline: "分析Android KeyEvent分发过程"
categories: junk
author: "Jeremy Liao"
meta: "Springfield"
---

本文会详细分析Android KeyEvent分发过程，以及相关api的作用

# 一、背景

Activity和View有三个override的方法dispatchKeyEvent、onKeyUp、onKeyDown，其中View还有一个setOnKeyListener方法。只知道这几个方法是设置处理key event的，但是一直不太理解这几个方法的作用和相互影响。所以准备写个demo彻底搞清楚这几个方法的作用和原理。

# 二、测试准备

### 新建三个类

```java
public class KeyDispatchDemo extends Activity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_key_dispatch);
    }

    @Override
    public boolean dispatchKeyEvent(KeyEvent event) {
        Log.d("Activity", "dispatchKeyEvent: " + event.getAction() + " | " + event.getKeyCode());
        return super.dispatchKeyEvent(event);
    }

    @Override
    public boolean onKeyUp(int keyCode, KeyEvent event) {
        Log.d("Activity", "onKeyUp: " + keyCode);
        return super.onKeyUp(keyCode, event);
    }

    @Override
    public boolean onKeyDown(int keyCode, KeyEvent event) {
        Log.d("Activity", "onKeyDown: " + keyCode);
        return super.onKeyDown(keyCode, event);
    }
}
```

```java
public class KeyDispatchLinearLayout extends LinearLayout {

    public KeyDispatchLinearLayout(Context context) {
        super(context);
    }

    public KeyDispatchLinearLayout(Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
    }

    public KeyDispatchLinearLayout(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }

    @Override
    public boolean dispatchKeyEvent(KeyEvent event) {
        Log.d("ViewGroup", "dispatchKeyEvent: " + event.getAction() + " | " + event.getKeyCode());
        return super.dispatchKeyEvent(event);
    }

    @Override
    public boolean onKeyUp(int keyCode, KeyEvent event) {
        Log.d("ViewGroup", "onKeyUp: " + keyCode);
        return super.onKeyUp(keyCode, event);
    }

    @Override
    public boolean onKeyDown(int keyCode, KeyEvent event) {
        Log.d("ViewGroup", "onKeyDown: " + keyCode);
        return super.onKeyDown(keyCode, event);
    }
}
```

```java
public class KeyDispatchView extends View {

    public KeyDispatchView(Context context) {
        super(context);
    }

    public KeyDispatchView(Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
    }

    public KeyDispatchView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }

    @Override
    public boolean dispatchKeyEvent(KeyEvent event) {
        Log.d("View", "dispatchKeyEvent: " + event.getAction() + " | " + event.getKeyCode());
        return super.dispatchKeyEvent(event);
    }

    @Override
    public boolean onKeyUp(int keyCode, KeyEvent event) {
        Log.d("View", "onKeyUp: " + keyCode);
        return super.onKeyUp(keyCode, event);
    }

    @Override
    public boolean onKeyDown(int keyCode, KeyEvent event) {
        Log.d("View", "onKeyDown: " + keyCode);
        return super.onKeyDown(keyCode, event);
    }
}
```

### 布局文件：

<pre class="has">
&lt;?xml version="1.0" encoding="utf-8"?&gt;
&lt;LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"&gt;

    &lt;com.lhl.common.keyevent.KeyDispatchLinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"&gt;

        &lt;com.lhl.common.keyevent.KeyDispatchView
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:background="@color/blue"
            android:focusable="true" /&gt;
    &lt;/com.lhl.common.keyevent.KeyDispatchLinearLayout&gt;

&lt;/LinearLayout&gt;
</pre>

# 三、Key event分发流程分析

首先在关键方法上都打上断点，按下键盘，发出一个key event事件。

## 1、dispatchKeyEvent的执行流程

dispatchKeyEvent的分发主要是：DecorView——&gt;Activity——&gt;ViewGroup——&gt;View

![](https://123.sankuai.com/km/api/file/30788782/31289083)

一个Key event是首先传到DecorView中，DecorView通过下面的代码分发给Activity：

```java
if (!mWindow.isDestroyed()) {
    final Window.Callback cb = mWindow.getCallback();
    final boolean handled = cb != null && mFeatureId < 0 ? cb.dispatchKeyEvent(event)
            : super.dispatchKeyEvent(event);
    if (handled) {
        return true;
    }
}
```

其中mWindow.getCallback()就是目标Activity

在Activity的dispatchKeyEvent方法里面，传递给了PhoneWindow的superDispatchKeyEvent方法

![](https://123.sankuai.com/km/api/file/30788782/31128218)

PhoneWindow的superDispatchKeyEvent方法会调到ViewGroup的dispatchKeyEvent方法：

![](https://123.sankuai.com/km/api/file/30788782/31128224)

ViewGroup会一层一层的把key event传递给持有焦点的child：

![](https://123.sankuai.com/km/api/file/30788782/31288831)

总结：在这个dispatchKeyEvent链中，只要有一个返回true，就是没调super.dispatchKeyEvent方法，那么key event的传递就会终止。

## 2、onKeyDown的执行流程

view.onKeyDown的执行：

![](https://123.sankuai.com/km/api/file/30788782/31322838)

和dispatch的流程一致，key event从DecorView分发到View的dispatchKeyEvent

在View.dispatchKeyEvent分发到KeyEvent.dispatch:

![](https://123.sankuai.com/km/api/file/30788782/31321884)

最后分发到View.onKeyDown

如果View.onKeyDown没有返回true，那么key event会继续流传给Activity：

![](https://123.sankuai.com/km/api/file/30788782/31129825)

最后会传给Activity.onKeyDown

总结：View.onKeyDown如果返回true，表示View消耗了这个事件，那么这个事件不会再流转到Activity

## 3、onKeyUp的执行流程

onKeyUp的流程和onKeyDown类似，不再累述。

总结：View.onKeyUp如果返回true，表示View消耗了这个事件，那么这个事件不会再流转到Activity。并且onKeyUp的流转和onKeyDown是独立的，不会相互影响。
