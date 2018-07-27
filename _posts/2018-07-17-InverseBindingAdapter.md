---
layout: post
title: "InverseBindingAdapter研究"
categories: android
author: "Jeremy Liao"
---

本文会分析InverseBindingAdapter并介绍其应用场景

# 一、InverseBindingAdapter 和 BindingAdapter
BindingAdapter主要定义赋值给一个view，反过来，InverseBindingAdapter定义从一个view取值
# 二、InverseBindingAdapter的使用
TextView的android:text="@={vm.data}"可以把textview同viewmodel双向绑定，这是在默认的TextViewBindingAdapter实现的：
```java
public class TextViewBindingAdapter {

    private static final String TAG = "TextViewBindingAdapters";
    @SuppressWarnings("unused")
    public static final int INTEGER = 0x01;
    public static final int SIGNED = 0x03;
    public static final int DECIMAL = 0x05;

    @BindingAdapter("android:text")
    public static void setText(TextView view, CharSequence text) {
        final CharSequence oldText = view.getText();
        if (text == oldText || (text == null && oldText.length() == 0)) {
            return;
        }
        if (text instanceof Spanned) {
            if (text.equals(oldText)) {
                return; // No change in the spans, so don't set anything.
            }
        } else if (!haveContentsChanged(text, oldText)) {
            return; // No content changes, so don't set anything.
        }
        view.setText(text);
    }

    @InverseBindingAdapter(attribute = "android:text", event = "android:textAttrChanged")
    public static String getTextString(TextView view) {
        return view.getText().toString();
    }
  
  ...
    
    @BindingAdapter(value = {"android:beforeTextChanged", "android:onTextChanged",
            "android:afterTextChanged", "android:textAttrChanged"}, requireAll = false)
    public static void setTextWatcher(TextView view, final BeforeTextChanged before,
            final OnTextChanged on, final AfterTextChanged after,
            final InverseBindingListener textAttrChanged) {
        final TextWatcher newValue;
        if (before == null && after == null && on == null && textAttrChanged == null) {
            newValue = null;
        } else {
            newValue = new TextWatcher() {
                @Override
                public void beforeTextChanged(CharSequence s, int start, int count, int after) {
                    if (before != null) {
                        before.beforeTextChanged(s, start, count, after);
                    }
                }

                @Override
                public void onTextChanged(CharSequence s, int start, int before, int count) {
                    if (on != null) {
                        on.onTextChanged(s, start, before, count);
                    }
                    if (textAttrChanged != null) {
                        textAttrChanged.onChange();
                    }
                }

                @Override
                public void afterTextChanged(Editable s) {
                    if (after != null) {
                        after.afterTextChanged(s);
                    }
                }
            };
        }
        final TextWatcher oldValue = ListenerUtil.trackListener(view, newValue, R.id.textWatcher);
        if (oldValue != null) {
            view.removeTextChangedListener(oldValue);
        }
        if (newValue != null) {
            view.addTextChangedListener(newValue);
        }
    }
```
这里的关键点就是@InverseBindingAdapter去定义取值操作，注意这里有个event = "android:textAttrChanged"，这指的是当这个事件android:textAttrChanged发生的时候才去从view取值到viewmodel。那么如何定义android:textAttrChanged何时发生呢，这里是个纯粹的观察者，TextView的话是需要定义一个TextWatcher然后在onTextChanged的时候回调：textAttrChanged.onChange()。

这里还有一个实例，就是为一个自定的控件定义InverseBindingAdapter：
```java
public final class PlusMinusBindingAdapter {

    private PlusMinusBindingAdapter() {
    }

    @BindingAdapter("android:count")
    public static void setCount(PlusMinusLayout view, int count) {
        view.setCount(count);
    }

    @InverseBindingAdapter(attribute = "android:count", event = "countChanged")
    public static int getCount(PlusMinusLayout view) {
        return view.getCount();
    }

    @BindingAdapter(value = {"countChanged"}, requireAll = false)
    public static void setDelegate(PlusMinusLayout view, final InverseBindingListener listener) {
        final PlusMinusLayout.Delegate delegate;
        if (listener == null) {
            delegate = null;
        } else {
            delegate = new PlusMinusLayout.Delegate() {
                @Override
                public void onCountChanged(int count, boolean isPlus) {
                    if (listener != null) {
                        listener.onChange();
                    }
                }
            };
        }
        view.setDelegate(delegate);
    }
}
```
跟前面讲解的套路是一样的，这里就不多说了。