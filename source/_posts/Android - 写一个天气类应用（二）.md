---
title: Android - 启动页
date: 2018-01-09 15：30
tags: ['Android']
categories: Android
---

继续撸那个天气类的 App，这次把写启动页过程中学到的知识点记录一下：
> 模仿项目：[SeeWeather](https://github.com/xcc3641/SeeWeather)

<!-- more -->

首先简单讲一下项目的文件目录结构
```
├── java
│   └── com
│       └── example
│           └── lisen
│               └── seeweathercp
│                   ├── base
│                   ├── common
│                   │   └── utils
│                   ├── component
│                   └── modules
│                       ├── city
│                       │   ├── adapter
│                       │   ├── db
│                       │   ├── domain
│                       │   └── ui
│                       ├── launch
│                       ├── main
│                       │   ├── adapter
│                       │   ├── domain
│                       │   ├── listener
│                       │   └── ui
│                       ├── service
│                       └── setting
│                           └── ui
└── res
    ├── anim
    ├── drawable
    ├── drawable-v21
    ├── drawable-v24
    ├── layout
    ├── menu
    ├── mipmap-anydpi-v26
    ├── mipmap-hdpi
    ├── mipmap-mdpi
    ├── mipmap-night
    ├── mipmap-xhdpi
    ├── mipmap-xxhdpi
    ├── mipmap-xxxhdpi
    ├── raw
    ├── values
    ├── values-v21
    └── xml
```

其中的 modules 文件夹就是用来存放应用各个模块对应的 Activity, 我们可以在 launch 文件夹上右键选择 `create -> class`, 这里不选择 `create -> activity`，因为启动页的页面内容只有一张图片，不需要单独的一个 layout.xml 文件来描述页面的结构，直接通过 AndroidManifest.xml 里面 style 的设置就可以完成了。

完成创建之后，会在对应的目录下能够找到 `laungch/FirstActivity`，在 AndroidManifest.xml 文件里面也可以找到对 FirstActivity 的注册内容。

### AndroidManifest.xml

在 AndroidManifest.xml 文件里面我们更改成如下内容：

```xml
<activity
  android:name=".modules.launch.FirstActivity"
  android:theme="@style/FirstTheme">
  <intent-filter>
      <action android:name="android.intent.action.MAIN" />

      <category android:name="android.intent.category.LAUNCHER" />
  </intent-filter>
</activity>
```

这里我们需要记录的知识点是 `android:theme="@style/FirstTheme"`，这里设置 theme 属性是 Android 中实现样式的三种方式之一，是最高层级的样式设置，在 AndroidManifest.xml 中对 Application 设置就是意味着这个应用下所有的视图元素都将实现 theme 中的样式，是全局样式。对 Activity 设置 theme 类似，只不过是针对该 Activity 下面的所有视图元素。

所以对于 `android:theme="@style/FirstTheme"` 在 FirstActivity 中所有视图元素都会有这些属性的特性。

这里 @style 是指 `res/values/styles.xml` 文件，是集中用来存放 `<style>` 标签声明的样式：

```xml
<style name="FirstTheme" parent="Theme.AppCompat">
    <item name="android:windowNoTitle">true</item>
    <item name="android:windowBackground">@mipmap/first_backpng</item>
</style>
```

 这个 `<style>` 的 name 就是这个样式的名字，类似于 css 的 class 名字的存在，parent 是继承的意思。这个样式下只有两个属性：
  - `<item name="android:windowNoTitle">true</item>`：声明该页面不需要 Title；
  - `<item name="android:windowBackground">@mipmap/first_backpng</item>`：声明背景图为 first_backpng;

这里 @mipmap 就是 `/res/mipmap/`，我们直接将整个 mipmap 文件夹 copy 复制就 OK 了。

关于 Andorid 的样式，想多记录一点东西，Android 设置元素样式的方法有三种：`Attr`、`Style` & `Theme`，但是也没有相关书籍专门介绍这一类东西，只看 Android 的官方文档比较枯燥，所以建议一边撸项目一边看别人代码，遇见没见过的直接问 Google 比较实在点。这里关于 Android 样式有两篇文件（ [Attr、Style和Theme详解](https://www.jianshu.com/p/dd79220b47dd) & [ANDROID样式的开发](https://keeganlee.me/post/android/20150830) ）讲得比较深入可以了解一下。

另外按照上一节中所写，我们将启动页设置为桌面 UI 启动项的页面，同时也是项目的首页面。

完成了启动页的页面再来看一下启动页的 Java 代码中的 UI 逻辑：

## FirstActivity.class

```Java
public class FirstActivity extends Activity {

    private static final String TAG = FirstActivity.class.getSimpleName();

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        overridePendingTransition(android.R.anim.fade_in, android.R.anim.fade_out);
        super.onCreate(savedInstanceState);
        Observable.timer(1, TimeUnit.SECONDS)
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(aLong -> {
                    MainActivity.launch(this);
                    finish();
                });
    }
}
```

这里面的逻辑也十分简单，就只有两个：
1. 页面渐隐渐现的动画
2. 延迟启动主活动

### 渐隐渐现的启动动画
`overridePendingTransition(android.R.anim.fade_in, android.R.anim.fade_out)` 负责实现这个逻辑，该函数为原生的 API，用于实现 Activity 在切换和退出过程中的动效。第一个参数是启动动效，第二个参数是结束动效。
对动效的具体实现都在 anim 文件中完成声明或者系统自带的一些简易动效，`overridePendingTransition` 负责执行。

### 延迟启动主活动
```java
Observable.timer(1, TimeUnit.SECONDS)
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(aLong -> {
        MainActivity.launch(this);
        finish();
    });
```

这里 Observable.timer 就是延迟执行，类似于 JS 中 `setTimeout`。timer 的第一个参数是数值，第二个参数是单位。这里使用的就是 Android 开发过程中非常流行的一种异步处理库 RxJava。其具体实现和 RxJS 都是遵循 RX 架构的设计思想。因为 RxJava 的使用十分频繁，所以对于它的了解因为更多一些，这里可以推荐两个教程：[ReactiveX/RxJava文档中文版](https://www.gitbook.com/book/mcxiaoke/rxdocs/details) & [给 Android 开发者的 RxJava 详解](https://gank.io/post/560e15be2dca930e00da1083)

以上就是实现一个简单的启动页 & 简单动效的过程中所查阅到的知识点。
