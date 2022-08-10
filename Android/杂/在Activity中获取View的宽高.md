## 在activity中如何正确获取View的宽高

在开发中，有时候需要在Activity中获取某个 view 的宽高，如果是直接在生命周期方法中使用 getWidth，getHeight，这二个方法其实是拿不到宽高的，可以发现最后拿到的值其实都是 0.

那么这是为什么呢？这是因为**View的measure过程和Activity的生命周期方法不是同步执行的，因此无法保证Activity执行了onCreate、onStart、onResume时某个View已经测量完毕了，如果View没有测量完毕，那么获得的宽高就是0。**


准确的在Activity中测量一个View的宽高尺寸

### Activity / View # onWindowFocusChanged

onWindowFocusChanged这个方法的含义是：View已经初始化完毕，宽高已经准备好了，这个时候去获取宽高是没有问题的。需要注意的是，onWindowFocusChanged会被调用多次，当Activity的窗口得到焦点和失去焦点的时候都会被调用一次。

```java
@Override
public void onWindowFocusChanged(boolean hasFocus) {
    super.onWindowFocusChanged(hasFocus);
    if (hasFocus) {
        // 获取View的宽高
        int width = view.getMeasuredWidth();
        int height = view.getMeasuredHeight();
    }
}
```

### view.post(runnable)

通过post可以将一个runnable投递到消息队列的尾部，然后等待Looper调用此runnable的时候，View也已经初始化好了。

```java
@Override
protected void onStart() {
    super.onStart();
    view.post(new Runnable() {
        @Override
        public void run() {
            // 获取View的宽高
            int width = mTextView.getMeasuredWidth();
            int height = mTextView.getMeasuredHeight();
        }
    });
}
```

### ViewTreeObserver.addOnGlobalLayoutListener

使用ViewTreeObserver的众多回调接口也可以，比如使用OnGlobalLayoutListener这个接口，当View树的状态发生改变或者View树内部的View的可见性发生改变的时候，onGlobalLayout方法将被回调，因此这是获取View的宽高一个很好的机会。值得注意的是，伴随着View树的状态改变等，onGlobalLayout会被多次调用。

```java
@Override
protected void onStart() {
    super.onStart();

    ViewTreeObserver observer = view.getViewTreeObserver();
    observer.addOnGlobalLayoutListener(new ViewTreeObserver.OnGlobalLayoutListener() {
        @Override
        public void onGlobalLayout() {
            // 获取View的宽高
            view.getViewTreeObserver().removeOnGlobalLayoutListener(this);
            int width = view.getMeasuredWidth();
            int height = view.getMeasuredHeight();
        }
    });
}
```

### view.measure(int widthMeasureSpec , int heightMeasureSpec)

通过手动测量View的宽高，需要根据View的LayoutParams来分不同情况处理

1. match_parent

  无法测量出具体的宽高，根据View的测量过程，构造这种measureSpec需要知道parentSize，即父容器的剩下空间，而这个时候我们无法知道parentSize的大小，所以理论上我们不可能测量出View的大小

2. 具体的数值

  直接makeMeasureSpec固定值，然后调用 view..measure就可以了

  ```java
  int widthMeasureSpec = MeasureSpec.makeMeasureSpec(100, MeasureSpec.EXACTLY);
  int heightMeasureSpec = MeasureSpec.makeMeasureSpec(100, MeasureSpec.EXACTLY);
  view.measure(widthMeasureSpec,heightMeasureSpec);
  ```
3. wrap_content

  在最大化模式下，用View理论上能支持的最大值去 构造MeasureSpec是合理的。

  ```java
  int widthMeasureSpec = MeasureSpec.makeMeasureSpec((1<<30)-1, MeasureSpec.AT_MOST);
  int heightMeasureSpec = MeasureSpec.makeMeasureSpec((1<<30)-1, MeasureSpec.AT_MOST);
  view.measure(widthMeasureSpec,heightMeasureSpec);
  关于View测量的一个工具类
  import android.view.View;
  import android.view.ViewGroup;
  import android.view.ViewTreeObserver;
  ```


布局参数设置工具类:

```java
public class LayoutParamerUtils {

    // 描述: 获取 指定view 的 viewTreeObserver
    public static void getViewObserver(final View viewtest, final ViewObserverListener viewObserverListener) {

        if(viewtest == null){
            return;
        }

        viewtest.getViewTreeObserver().addOnGlobalLayoutListener(new  ViewTreeObserver.OnGlobalLayoutListener(){
            @Override
            public void onGlobalLayout() {

                viewtest.getViewTreeObserver().removeOnGlobalLayoutListener(this);

                if(viewObserverListener != null){
                    viewObserverListener.onViewObserverResult();
                }
            }
        });
    }

    // getViewObserver 回调接口
    public interface ViewObserverListener{
        void onViewObserverResult();
    }
}
```
