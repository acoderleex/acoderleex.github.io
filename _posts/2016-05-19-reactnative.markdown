---
layout: post
title:  "Android 组件化开发"
date:   2016-05-19 11:34:25
datakey: 5555555
dataurl: http://acoderleex.github.io/2016/05/19/zujianhua/
tags: featured
image: /assets/article_images/2014-08-29-welcome-to-jekyll/desktop.JPG
---


        组件化不是插件化，插件化是在［运行时］，而组件化是在［编译时］。换句话说，插件化是基于多 APK 的，而组件化本质上还是只有一个 APK。
代码实现上核心思路要紧记一句话：开发时是 application，发版时是 library。


        组件化并不是新话题，其实很早很早以前我们开始为项目解耦的时候就讨论过的。但那时候我们说的是功能组件化。比如很多公司都常见的，网络请求模块、登录注册模块单独拿出来，交给一个团队开发，而在用的时候只需要接入对应模块的功能就可以了。


来看一段 gradle 代码：

if (isDebug.toBoolean()) {
    apply plugin: 'com.android.application'
} else {
    apply plugin: 'com.android.library'
}
非常好理解，我们在开发的时候，module 如果是一个库，会使用com.android.library插件，如果是一个应用，则使用com.android.application插件，我们通过一个开关来控制这个状态的切换。
然后因为我们需要在 library 和 application 之间切换，manifest文件也需要提供两套。





遇见的问题

解决 Application 冲突

有些人喜欢将 application 单例，写一个静态的对象，然后在代码里面需要context的时候用这个全局单例。这样的情况我送大家一个工具类(其实是从冯老师代码里偷来的)：Common

public class App {
    public static final Application INSTANCE;

    static {
        Application app = null;
        try {
            app = (Application) Class.forName("android.app.AppGlobals").getMethod("getInitialApplication").invoke(null);
            if (app == null)
                throw new IllegalStateException("Static initialization of Applications must be on main thread.");
        } catch (final Exception e) {
            LogUtils.e("Failed to get current application from AppGlobals." + e.getMessage());
            try {
                app = (Application) Class.forName("android.app.ActivityThread").getMethod("currentApplication").invoke(null);
            } catch (final Exception ex) {
                LogUtils.e("Failed to get current application from ActivityThread." + e.getMessage());
            }
        } finally {
            INSTANCE = app;
        }
    }
}

重复依赖


重复依赖问题其实在开发中经常会遇到，比如你 compile 了一个A，然后在这个库里面又 compile 了一个B，然后你的工程中又 compile 了一个同样的B，就依赖了两次。

默认情况下，如果是 aar 依赖，gradle 会自动帮我们找出新版本的库而抛弃旧版本的重复依赖。但是如果你使用的是 project 依赖，gradle 并不会去去重，最后打包就会出现代码中有重复的类了。

一种是 将 compile 改为 provided，只在最终的项目中 compile 对应的代码；



* 在gradle配置里面，增加两个task
两个task分别创建ListActivity任务和创建Model任务

// gradle createModel -Pmodel=wishlist
task createModel (type: JavaExec, dependsOn: classes) {
    if(project.hasProperty('model')){
        args(model)
    }
    main = 'com.templateutil.CreateModel'
    classpath = sourceSets.main.runtimeClasspath
}
// gradle createListActivity -Pmodel=wishlist
task createListActivity (type: JavaExec, dependsOn: classes) {
    if(project.hasProperty('model')){
        args(model)
    }
    main = 'com.templateutil.CreateListActivity'
    classpath = sourceSets.main.runtimeClasspath
}
* 然后就可以使用了
harry$ gradle createListActivity -Pmodel=wishlist
