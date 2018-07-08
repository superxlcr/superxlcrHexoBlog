---
title: Android Activity savedInstanceState
tags: [android]
categories: [android]
date: 2017-06-01 11:14:54
description: Android Activity savedInstanceState
---
最近了解了一些关于Android的savedInstanceState相关的知识，在此进行一下总结。


在Android的Activity控件中的onCreate方法中，我们可以获得的一个参数为savedInstanceState：


```java
    @Override
    protected void onCreate(Bundle savedInstanceState)
```


该参数的作用是什么呢？

这时，我们就得提到构造savedInstanceState的另一个方法：


```java
    @Override
    protected void onSaveInstanceState(Bundle outState)
```


该方法的作用为：
Called to retrieve per-instance state from an activity before being killed so that the state can be restored in onCreate(Bundle) or onRestoreInstanceState(Bundle) (the Bundle populated by this method will be passed to both).
即：用于保存Activity被杀死前一刻的每个实例的状态，然后当Activity被重启时，传递给onCreate与onRestoreInstanceState方法
该方法的调用时间为：
This method is called before an activity may be killed so that when it comes back some time in the future it can restore its state.

即：当一个Activity可能被杀死时调用。由于是在Activity可能被杀死时调用，但许多情况下调用了onSaveInstanceState方法后Activity并不一定会被杀死，因此其与调用onCreate与onRestoreInstanceState并不是成对的。常见的调用时间有：

1. 当用户按下HOME键时
2. 长按HOME键，选择运行其他的程序时
3. 按下电源按键（关闭屏幕显示）时
4. 从activity A中启动一个新的activity时
5. 屏幕方向切换时

另外，该方法与Activity的生命周期没有关系，并不会依照特定的顺序被调用。但如果被调用时，一定保证会在onStop方法前被调用。
值得注意的是：
The default implementation takes care of most of the UI per-instance state for you by calling onSaveInstanceState() on each view in the hierarchy that has an id, and by saving the id of the currently focused view (all of which is restored by the default implementation of onRestoreInstanceState(Bundle)). If you override this method to save additional information not captured by each individual view, you will likely want to call through to the default implementation, otherwise be prepared to save all of the state of each view yourself.


据博主所了解，onSaveInstanceState默认方法会为我们自动存储当前Activity对应布局中的View以及Fragment相关的状态，如果需要存储某些变量的值则需要我们重写onSaveInstanceState方法自行操作，重写方法时应注意调用父类方法来保存Activitiy中的UI 状态，通过onRestoreInstanceState方法恢复数据同理。
下面为一个Activity切换屏幕方向恢复数据的例子：


```java
public class MainActivity extends AppCompatActivity {

    private static final String TAG = "MyLog";

    private TextView textView;
    private EditText editText;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        if (savedInstanceState == null) { // 初次运行
            textView = (TextView) findViewById(R.id.text_view);
            editText = (EditText) findViewById(R.id.editText);
            textView.setText("before");
        } else { // 恢复数据
            textView = (TextView) findViewById(R.id.text_view);
            textView.setText("after");
        }
    }

    @Override
    protected void onSaveInstanceState(Bundle outState) {
        // 保存各种View的状态
        super.onSaveInstanceState(outState);
        Log.d(TAG, "onSaveInstanceState");
    }

    @Override
    protected void onRestoreInstanceState(Bundle savedInstanceState) {
        // 恢复各种View的状态
        super.onRestoreInstanceState(savedInstanceState);
        Log.d(TAG, "onRestoreInstanceState");
    }
}
```


Activity的布局xml文件包含了一个TextView与一个EditText。



首次运行：
![首次运行_pic1](1.jpg)

在EditText中输入内容：
![输入内容_pic2](2.jpg)

进行屏幕切换：
![屏幕切换_pic3](3.jpg)

不调用onRestoreInstanceState的父类方法，其余步骤同上，屏幕切换后：


```java
    @Override
    protected void onRestoreInstanceState(Bundle savedInstanceState) {
        // 恢复各种View的状态
//        super.onRestoreInstanceState(savedInstanceState);
        Log.d(TAG, "onRestoreInstanceState");
    }
```

![不使用父类方法，屏幕切换_pic4](4.jpg)



可以看到，在没有调用父类方法时，EditText的输入内容变回了原本的Name而非我们输入的input something，由此看来，savedInstanceState确实具有保存Activity中布局的UI状态的功能。
因此，在我们实现自定义View控件的时候也应该记得实现View的onSaveInstanceState方法，用于存储UI控件的状态（默认实现为什么都不保存）。


PS：savedInstance虽然默认会保留fragment相关信息，但只是保留了fragment 对象本身而已（不需要我们再次add fragment），但每次初始化图像时均会重新调用onCreateView重新绘制，因此UI上的状态也会丢失（比如EditText输入的内容等），此时我们需要重写fragment的onSaveInstanceState自行保存其中的UI状态，并在onViewStateRestored方法中恢复：


```java
public class MyFragment extends Fragment {

    private static final String VIEWS_TAG = "my_fragment:views";

    @Nullable
    @Override
    public View onCreateView(LayoutInflater inflater, @Nullable ViewGroup container, Bundle savedInstanceState) {
        return inflater.inflate(R.layout.fragment_my, container, false);
    }

    @Override
    public void onViewStateRestored(Bundle savedInstanceState) {
        super.onViewStateRestored(savedInstanceState);
        // 恢复UI状态
        if (savedInstanceState != null && getView() != null) {
            SparseArray<Parcelable> sparseArray = savedInstanceState.getSparseParcelableArray(VIEWS_TAG);
            getView().restoreHierarchyState(sparseArray);
        }
    }

    @Override
    public void onSaveInstanceState(Bundle outState) {
        super.onSaveInstanceState(outState);
        // 保存UI状态
        SparseArray<Parcelable> sparseArray = new SparseArray<Parcelable>();
        if (getView() != null) {
            getView().saveHierarchyState(sparseArray);
            outState.putSparseParcelableArray(VIEWS_TAG, sparseArray);
        }
    }
}
```



