---
title: Android - AndroidManifest.xml
date: 2018-01-09 15：30
tags: ['Android']
categories: Android
---

私下花了蛮多时间学习 Android 的，最近模仿一个开源的天气类应用，把过程中查找到的一些知识点记录下来，做下学习笔记。
> 模仿项目：[SeeWeather](https://github.com/xcc3641/SeeWeather)

<!-- more -->

## 配置文件内容
```XML
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.lisen.seeweathercp">

    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
    <uses-permission android:name="android.permission.INTERNET" />
    <!-- 悬浮窗 -->
    <uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />

    <application
        android:name=".base.BaseApplication"
        android:allowBackup="true"
        android:hardwareAccelerated="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:largeHeap="true"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">

        <service android:name=".modules.service.AutoUpdateService"/>

        <activity
            android:name=".modules.launch.FirstActivity"
            android:theme="@style/FirstTheme">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
        <activity
            android:name=".modules.main.ui.MainActivity"
            android:launchMode="singleTask"
            android:screenOrientation="portrait"
            android:theme="@style/AppTheme" />
        <activity
            android:name=".modules.city.ui.ChoiceCityActivity"
            android:screenOrientation="portrait" />
        <activity
            android:name=".modules.main.ui.DetailCityActivity"
            android:screenOrientation="portrait" />
        <activity
            android:name=".modules.setting.ui.SettingActivity"
            android:screenOrientation="portrait"/>
    </application>

</manifest>
```

## android:name
android:name=".base.BaseApplication" 的意思是：为这个应用实现的Application子类的全限定名称。当应用启动时，这个类将在应用的其他组件之前被实例化。这个子类是可选的，在项目目录中实际对于 `com.example.lisen.seeweathercp.base.BaseApplication` 这个类。

所有在 AndrodiManifest.xml 里面注册的 Acitivity 都必须继承于 BaseActivity（具体的 Activity 可以是 BaseActivity 的子类的子类）。这样统一共同的父类可以全局提供方法 & 变量，省去重复编码相同的事。

```java
public abstract class BaseActivity extends RxAppCompatActivity {
    private static String TAG = BaseActivity.class.getSimpleName();

    @override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
      super.onCreate(savedInstance);
      setContentView(layoutId());
      ButterKnife.bind(this);
    }

    protected abstract int layoutId();

    @Override
    protected void onStop() {
        super.onStop();
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
    }

    public static void setdayTheme(AppCompatActivity activity) {
      // TODO
    }

    public static void setNightTheme(AppCompatActivity activity) {
      // TODO
    }

    public void setTheme(AppCompatActivity activity) {
      // TODO
    }
}
```
在我的 BaseApplication 中，我定义了 TAG 这个全局变量，用于在调试代码时，区分调试信息来自于哪个类。同时在每个 Activity 的共同生命周期中完成了 `setContent` 和 `ButterKnife.bind(this)` 的操作，这样每个 Activity 只需要在自己的 `onCreate` 函数里面实现 `super.onCreate()` 就不必再去写两行代码了。
所以我们可以看到自定义公共父类的意思在于提供全局函数和变量，减少每个 Activity 的重复工作。

不过我们也可以考虑这样的功能通过一个静态类来实现，为什么不呢？因为用静态类的话，在程序退出后不会立刻被 GC 垃圾回收，当再次进入的时候全局状态里面的一些变量可能会保持上一次的状态，因此导致初始化异常。所以通过在 application 标签里声明 `android:name` 指定自定义父类是一种更好的做法。

## android:hardwareAccelerated
`android:hardwareAccelerated = true` 使整个应用程序使用硬件加速，可以通过 `View.isHardwareAccelerated()` 的返回值来判断是否启动了硬件加速。

## android:largeHeap
`android:largeHeap` 会为整个应用请求系统分配更大的内存空间，但是我们不应该滥用这个属性，对于本身对内存要求高的图片或视频应用，我们应该申请更多的内存，但是如果仅仅是为了解决 `OutOfMemeoryError` 的报错，申请更多的内存空间是治标不治本的，不如多花点精力在代码层面上解决内存泄漏的问题。

## android:supportsRtl
从字面上看 `android:supportsRtl` 就是询问我们的应用是否需要支持 Rtl。Rtl 是指从右至左的布局，如果支持 Rtl, 并且 targeSdkVersion >= 17 ，那么系统就会激活各种 RTL 的 API，我们就可以实现 RTL 布局。

## android.intent.action.MAIN & android.intent.category.LAUNCHER
- android.intent.action.MAIN：决定应用的入口 Activity，也就是我们启动应用时首先显示哪一个 Activity;
- android.intent.category.LAUNCHER：表示 Activity 应该被列入系统的启动器，也就是说在桌面上添加一个桌面 UI 作为启动器。

那么当一个 Activity 同时拥有 `<action android:name="android.intent.action.MAIN" />` 和 `<category android:name="android.intent.category.LAUNCHER" />` 时，我们就可以认为这个 Activity 会在桌面上添加一个 UI 启动器，点击这个 UI 启动器会打开 App，并且第一个显示的 Activity 就是自己。

- 如果 App 没有一个 Activity 有 `android.intent.action.MAIN`，那么就是说这个 App 没有第一优先显示的 Activity ，App 启动时就会报错。
- 如果 App 没有一个 Activity 有 `android.intent.category.LAUNCHER`, 那么 App 在桌面上就没有启动 UI。
- 如果 App 多个 Activity 有 `android.intent.action.MAIN` 和 `android.intent.category.LAUNCHER`，那么桌面上就会有多个启动器的 UI，每个 UI 对应于各种 Activity 上 `android.intent.action.MAIN`，会优先启动该 Activity 显示。

所以我们可以了解到，为了 App 能够正常启动，至少有一个 Activity 需要同时拥有 `<action android:name="android.intent.action.MAIN" />` 和 `<category android:name="android.intent.category.LAUNCHER" />`。

对于这两个的各种配置的各种排列组合，想了解更加详细，可以阅读这篇[文章](http://blog.csdn.net/lindroid20/article/details/51993247)。

## android:launchMode
`android:launchMode` 是定义一个 Activity 的启动模式，有以下四个值可供选择：standard、singleTop、singleTask、singleInstance。

再次之前我们还需要理解一个概念：Task，我认为 Task 类似于前端的路由栈，也就是 location.history。

- standard：在一个 Task 中，每一个 Intent 访问该 Activity 时都会启动创建一个全新的实例。也就是说一个 Task 会存在多个同一个 Activity 的实例。
- singleTop：类似于 standard，但是一种场景下不会创建新的实例，就是当前 Task 的栈顶就是该 Activity 时，会直接复用该 Activity。
- singleTask：保证在 Task 中有且仅有一个 Acitivity 的实例，每一次访问都是唯一的一个实例。如果该实例已存在，又访问到这个实例时，会销毁 Task 中所有在这个实例之前的所有 Activity 实例。
- singleInstance：和 singleTask 比起来，更加霸道的一种模式，同样可以保证 Task 中有且仅有一个实例，但是存放 singleInstance 的实例的 Task 只能存放一个该模式的 Activity 的实例，其他启动的 Activity 会放入其他的 Task 中。
更多资料可以翻阅这篇[文章](http://droidyue.com/blog/2015/08/16/dive-into-android-activity-launchmode/index.html)。

以上就是在写 AndroidManifest.xml 时了解到的知识点，好记性不如烂笔头 🙃。
