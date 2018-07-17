# ---
# layout: post
# title: sougou输入法搜索劫持问题调研
# category: android
# keywords: android,smali,hijack
# ---


# 0x01.前言

记于在百度实习的这段时间,这是临时发下来给我的一个case

# 0x02.问题

当使用android搜狗输入法进行搜索的时候,其会有多一层搜索选项.当点击了该层的结果将会被sougou搜索劫持

**现象一**:

![1](http://7xkw0v.com1.z0.glb.clouddn.com/Screenshot_2015-12-22-13-37-00.png)

看一看到搜狗输入法出现了一层劫持搜索

![2](http://7xkw0v.com1.z0.glb.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202016-02-28%20%E4%B8%8B%E5%8D%884.25.43.png)

当点击,则被劫持到sougou搜索

![3](http://7xkw0v.com1.z0.glb.clouddn.com/4E74A8C4168B5103D1A32BC1CEB0A300.jpg)


**现象二**:

当在应用宝搜索的时候,搜狗输入法也会出现劫持框

![4](http://7xkw0v.com1.z0.glb.clouddn.com/Screenshot_2015-12-22-14-10-03(1).png)

但是不进行劫持操作而是直接进行搜索

![5](http://7xkw0v.com1.z0.glb.clouddn.com/Screenshot_2015-12-22-14-10-10.png)

**现象三**

在登陆注册的输入框将不出现劫持


# 0x03.分析

## 0x031.劫持的浏览器列表

如果有浏览器劫持发生,一定是用了包名来做判断的

所以可以用包名去作为分析入口.

![6](http://7xkw0v.com1.z0.glb.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202016-02-28%20%E4%B8%8B%E5%8D%884.55.30.png)

果然出现了

**com/sogou/upd/webserver/WebServerController**

```

    # 创建一个0xf大小的数组并且放于寄存器v0
    new-array v0, v0, [Ljava/lang/String;

    # 接下来就是初始化数组的过程
    const/4 v1, 0x0

    const-string v2, "com.tencent.mtt"

    aput-object v2, v0, v1

    const/4 v1, 0x1

    const-string v2, "com.android.browser"

    aput-object v2, v0, v1

    const/4 v1, 0x2

    const-string v2, "com.UCMobile"

    aput-object v2, v0, v1

    const/4 v1, 0x3

    const-string v2, "sogou.mobile.explorer"

    aput-object v2, v0, v1

    const/4 v1, 0x4

    const-string v2, "com.baidu.browser.apps"

    aput-object v2, v0, v1

    const/4 v1, 0x5

    const-string v2, "com.android.chrome"

    aput-object v2, v0, v1

    const/4 v1, 0x6

    const-string v2, "org.mozilla.firefox"

    aput-object v2, v0, v1

    const/4 v1, 0x7

    const-string v2, "com.ijinshan.browser_fast"

    aput-object v2, v0, v1

    const/16 v1, 0x8

    const-string v2, "com.ijinshan.browser"

    aput-object v2, v0, v1

    const/16 v1, 0x9

    const-string v2, "com.uc.browser"

    aput-object v2, v0, v1

    const/16 v1, 0xa

    const-string v2, "com.qihoo.browser"

    aput-object v2, v0, v1

    const/16 v1, 0xb

    const-string v2, "com.oupeng.mini.android"

    aput-object v2, v0, v1

    const/16 v1, 0xc

    const-string v2, "com.opera.mini.android"

    aput-object v2, v0, v1

    const/16 v1, 0xd

    const-string v2, "com.mx.browser"

    aput-object v2, v0, v1

    const/16 v1, 0xe

    const-string v2, "com.dolphin.browser.xf"

    aput-object v2, v0, v1

    sput-object v0, Lcom/sogou/upd/webserver/WebServerController;->TARGET_EXPLOER_PACKAGENAME:[Ljava/lang/String;

    # 数组名叫 TARGET_EXPLOER_PACKAGENAME,其初始化完毕

```

分析类WebServerController

1.单例:

```

.method public static getInstance(Landroid/content/Context;)Lcom/sogou/upd/webserver/WebServerController;
    .locals 1

    .prologue
    .line 18
    sget-object v0, Lcom/sogou/upd/webserver/WebServerController;->mController:Lcom/sogou/upd/webserver/WebServerController;

    if-nez v0, :cond_0

    .line 19
    new-instance v0, Lcom/sogou/upd/webserver/WebServerController;

    invoke-direct {v0, p0}, Lcom/sogou/upd/webserver/WebServerController;-><init>(Landroid/content/Context;)V

    sput-object v0, Lcom/sogou/upd/webserver/WebServerController;->mController:Lcom/sogou/upd/webserver/WebServerController;

    .line 21
    :cond_0
    sget-object v0, Lcom/sogou/upd/webserver/WebServerController;->mController:Lcom/sogou/upd/webserver/WebServerController;

    return-object v0
.end method

```

2.方法

```

.method public stopServer()V
.method public startServer()V

.method public setPackageName(Ljava/lang/String;)V
.method public getPackageName()Ljava/lang/String;

.method public static checkTargetPackage(Ljava/lang/String;)Z

.....

```

分析一下在startServer方法:

其内跑了一个Thread,对应的run方法为:

```

# virtual methods
.method public run()V
    .locals 3

    .prologue
    .line 45
    iget-object v0, p0, Lcom/sogou/upd/webserver/WebServerController$1;->this$0:Lcom/sogou/upd/webserver/WebServerController;

    new-instance v1, Lcom/sogou/upd/webserver/WebServer;

    iget-object v2, p0, Lcom/sogou/upd/webserver/WebServerController$1;->this$0:Lcom/sogou/upd/webserver/WebServerController;

    # getter for: Lcom/sogou/upd/webserver/WebServerController;->mContext:Landroid/content/Context;
    invoke-static {v2}, Lcom/sogou/upd/webserver/WebServerController;->access$100(Lcom/sogou/upd/webserver/WebServerController;)Landroid/content/Context;

    move-result-object v2

    invoke-direct {v1, v2}, Lcom/sogou/upd/webserver/WebServer;-><init>(Landroid/content/Context;)V

    # setter for: Lcom/sogou/upd/webserver/WebServerController;->mServer:Lcom/sogou/upd/webserver/WebServer;
    invoke-static {v0, v1}, Lcom/sogou/upd/webserver/WebServerController;->access$002(Lcom/sogou/upd/webserver/WebServerController;Lcom/sogou/upd/webserver/WebServer;)Lcom/sogou/upd/webserver/WebServer;

    .line 46
    iget-object v0, p0, Lcom/sogou/upd/webserver/WebServerController$1;->this$0:Lcom/sogou/upd/webserver/WebServerController;

    # getter for: Lcom/sogou/upd/webserver/WebServerController;->mServer:Lcom/sogou/upd/webserver/WebServer;
    invoke-static {v0}, Lcom/sogou/upd/webserver/WebServerController;->access$000(Lcom/sogou/upd/webserver/WebServerController;)Lcom/sogou/upd/webserver/WebServer;

    move-result-object v0

    invoke-virtual {v0}, Lcom/sogou/upd/webserver/WebServer;->startServer()V

    .line 47
    return-void
.end method

```

分析一下WebServer,看了smali源码之后我也是醉了

其内的代码和[How to start an android web server service?](http://stackoverflow.com/questions/11763475/how-to-start-an-android-web-server-service) 这里的代码一致,连变量名都不改

就是开了一个后台的Server

这时候要是有**插桩**就好了,但是可惜的是有保护.搜狗的保护和别的壳有点不太一样,我一插桩他就提示不是正版,之后退出

这个暂且不分析,我们先分析其劫持的部分.这个的左右我估计可能是跑一个设置页还是啥,我看webhandle里面没有什么特别的内容(后门),当然也可能是用来确认当前上层的app是哪个,因为在android5.0以上废除了查询app栈的api,所以其有可能是用来判断当然在哪个app上

3.SogouIME

需要实现是一个输入法,需要一个主IME来做主界面键盘等

查找mainfest可以看到com/sohu/inputmethod/sogou/SogouIME.smali 为搜狗输入法的主IME

寻找到他相关劫持代码,在SogouIME.smali中的一个方法中果然出现了更新WebServerController状态的代码

```

.method public c(Ljava/lang/String;)V
    .locals 1

    .prologue
    .line 12338
    sget-object v0, Lcom/sohu/inputmethod/sogou/SogouAppApplication;->mAppContxet:Landroid/content/Context;

    invoke-static {v0, p1}, Lsogou/mobile/explorer/hotwords/entrance/HotwordsController;->notifyPackageName(Landroid/content/Context;Ljava/lang/String;)V

    .line 12339
    invoke-static {p1}, Lcom/sogou/androidtool/sdk/notification/appfrequency/AppFrequencyHelper;->recordAppUse(Ljava/lang/String;)V

    .line 12340
    invoke-virtual {p0}, Lcom/sohu/inputmethod/sogou/SogouIME;->getApplicationContext()Landroid/content/Context;

    move-result-object v0

    invoke-static {v0}, Lcom/sogou/upd/webserver/WebServerController;->getInstance(Landroid/content/Context;)Lcom/sogou/upd/webserver/WebServerController;

    move-result-object v0

    invoke-virtual {v0}, Lcom/sogou/upd/webserver/WebServerController;->getPackageName()Ljava/lang/String;

    move-result-object v0

    .line 12341
    if-eqz p1, :cond_0

    invoke-virtual {p1, v0}, Ljava/lang/String;->equals(Ljava/lang/Object;)Z

    move-result v0

    if-nez v0, :cond_0

    .line 12342
    sget-boolean v0, Lcom/sogou/upd/webserver/WebServer;->RUNNING:Z

    if-nez v0, :cond_1

    invoke-static {p1}, Lcom/sogou/upd/webserver/WebServerController;->checkTargetPackage(Ljava/lang/String;)Z

    move-result v0

    if-eqz v0, :cond_1

    invoke-virtual {p0}, Lcom/sohu/inputmethod/sogou/SogouIME;->getApplicationContext()Landroid/content/Context;

    move-result-object v0

    invoke-static {v0}, Lcom/sohu/inputmethod/settings/SettingManager;->getInstance(Landroid/content/Context;)Lcom/sohu/inputmethod/settings/SettingManager;

    move-result-object v0

    invoke-virtual {v0}, Lcom/sohu/inputmethod/settings/SettingManager;->bh()Z

    move-result v0

    if-eqz v0, :cond_1

    .line 12344
    invoke-virtual {p0}, Lcom/sohu/inputmethod/sogou/SogouIME;->getApplicationContext()Landroid/content/Context;

    move-result-object v0

    invoke-static {v0}, Lcom/sogou/upd/webserver/WebServerController;->getInstance(Landroid/content/Context;)Lcom/sogou/upd/webserver/WebServerController;

    move-result-object v0

    invoke-virtual {v0, p1}, Lcom/sogou/upd/webserver/WebServerController;->setPackageName(Ljava/lang/String;)V

    .line 12345
    invoke-virtual {p0}, Lcom/sohu/inputmethod/sogou/SogouIME;->getApplicationContext()Landroid/content/Context;

    move-result-object v0

    invoke-static {v0}, Lcom/sogou/upd/webserver/WebServerController;->getInstance(Landroid/content/Context;)Lcom/sogou/upd/webserver/WebServerController;

    move-result-object v0

    invoke-virtual {v0}, Lcom/sogou/upd/webserver/WebServerController;->startServer()V

    .line 12351
    :cond_0
    :goto_0
    return-void

    .line 12346
    :cond_1
    sget-boolean v0, Lcom/sogou/upd/webserver/WebServer;->RUNNING:Z

    if-eqz v0, :cond_0

    .line 12347
    invoke-virtual {p0}, Lcom/sohu/inputmethod/sogou/SogouIME;->getApplicationContext()Landroid/content/Context;

    move-result-object v0

    invoke-static {v0}, Lcom/sogou/upd/webserver/WebServerController;->getInstance(Landroid/content/Context;)Lcom/sogou/upd/webserver/WebServerController;

    move-result-object v0

    invoke-virtual {v0, p1}, Lcom/sogou/upd/webserver/WebServerController;->setPackageName(Ljava/lang/String;)V

    .line 12348
    invoke-virtual {p0}, Lcom/sohu/inputmethod/sogou/SogouIME;->getApplicationContext()Landroid/content/Context;

    move-result-object v0

    invoke-static {v0}, Lcom/sogou/upd/webserver/WebServerController;->getInstance(Landroid/content/Context;)Lcom/sogou/upd/webserver/WebServerController;

    move-result-object v0

    invoke-virtual {v0}, Lcom/sogou/upd/webserver/WebServerController;->stopServer()V

    goto :goto_0
.end method

```


在这一步之前其获取了相关的包名并且存入了WebServerController实例之中,这里调用WebServerController的getPackageName,并调用checkTargtpackage对其检查是否是指定浏览器


如果是指定浏览器将会对其直接进行intent


```


.method public a(Ljava/lang/String;Z)V
    .locals 6

    .prologue
    const/4 v0, 0x0

    .line 24509
    invoke-virtual {p0}, Lcom/sohu/inputmethod/sogou/SogouIME;->i()V

    .line 24510
    invoke-virtual {p0, v0}, Lcom/sohu/inputmethod/sogou/SogouIME;->requestHideSelf(I)V

    .line 24512
    :try_start_0
    invoke-static {}, Lcom/sohu/util/SystemPropertiesReflect;->getSdkVersion()I

    move-result v1

    const/16 v2, 0xe

    if-ge v1, v2, :cond_0

    move p2, v0

    .line 24514
    :cond_0
    if-eqz p2, :cond_2

    .line 24515
    invoke-virtual {p0}, Lcom/sohu/inputmethod/sogou/SogouIME;->getApplicationContext()Landroid/content/Context;

    move-result-object v0

    const/4 v1, 0x1
	
	# 注意这里,实际上实在这里面处理具体跳转
	
    invoke-static {v0, p1, v1}, Lsogou/mobile/explorer/hotwords/entrance/HotwordsController;->openHotwordsViewFromStartUrl(Landroid/content/Context;Ljava/lang/String;Z)V

    .line 24540
    :cond_1
    :goto_0
    return-void

    .line 24517
    :cond_2
    invoke-static {p1}, Landroid/net/Uri;->parse(Ljava/lang/String;)Landroid/net/Uri;

    move-result-object v1

    .line 24518
    new-instance v2, Landroid/content/Intent;

    const-string v3, "android.intent.action.VIEW"

    invoke-direct {v2, v3, v1}, Landroid/content/Intent;-><init>(Ljava/lang/String;Landroid/net/Uri;)V

    .line 24519
    const/high16 v1, 0x10000000

    invoke-virtual {v2, v1}, Landroid/content/Intent;->setFlags(I)Landroid/content/Intent;

    .line 24520
    invoke-virtual {p0}, Lcom/sohu/inputmethod/sogou/SogouIME;->getApplicationContext()Landroid/content/Context;

    move-result-object v1

    invoke-virtual {v1}, Landroid/content/Context;->getPackageManager()Landroid/content/pm/PackageManager;

    move-result-object v1

    const/4 v3, 0x0

    invoke-virtual {v1, v2, v3}, Landroid/content/pm/PackageManager;->queryIntentActivities(Landroid/content/Intent;I)Ljava/util/List;

    move-result-object v3

    .line 24522
    if-eqz v3, :cond_1

    invoke-interface {v3}, Ljava/util/List;->size()I

    move-result v1

    if-lez v1, :cond_1

    .line 24523
    invoke-interface {v3}, Ljava/util/List;->size()I

    move-result v4

    move v1, v0

    .line 24524
    :goto_1
    if-ge v1, v4, :cond_3

    .line 24525
    invoke-interface {v3, v1}, Ljava/util/List;->get(I)Ljava/lang/Object;

    move-result-object v0

    check-cast v0, Landroid/content/pm/ResolveInfo;

    iget-object v0, v0, Landroid/content/pm/ResolveInfo;->activityInfo:Landroid/content/pm/ActivityInfo;

    iget-object v0, v0, Landroid/content/pm/ActivityInfo;->packageName:Ljava/lang/String;

    .line 24526
    sget-object v5, Lcom/sohu/inputmethod/sogou/SogouIME;->f:Ljava/lang/String;

    invoke-virtual {v0, v5}, Ljava/lang/String;->equals(Ljava/lang/Object;)Z

    move-result v5

    if-eqz v5, :cond_4

    .line 24527
    invoke-virtual {v2, v0}, Landroid/content/Intent;->setPackage(Ljava/lang/String;)Landroid/content/Intent;

    .line 24528
    sget-object v0, Lcom/sohu/inputmethod/sogou/SogouIME;->f:Ljava/lang/String;

    const-string v1, "sogou.mobile.explorer"

    invoke-virtual {v0, v1}, Ljava/lang/String;->equals(Ljava/lang/Object;)Z

    move-result v0

    if-eqz v0, :cond_3

    .line 24529
    const-string v0, "sogou.mobile.explorer.stay_browser"

    const/4 v1, 0x1

    invoke-virtual {v2, v0, v1}, Landroid/content/Intent;->putExtra(Ljava/lang/String;Z)Landroid/content/Intent;

    .line 24533
    :cond_3
    invoke-virtual {p0}, Lcom/sohu/inputmethod/sogou/SogouIME;->getApplicationContext()Landroid/content/Context;

    move-result-object v0

    invoke-virtual {v0, v2}, Landroid/content/Context;->startActivity(Landroid/content/Intent;)V
    :try_end_0
    .catch Ljava/lang/Exception; {:try_start_0 .. :try_end_0} :catch_0

    goto :goto_0

    .line 24537
    :catch_0
    move-exception v0

    goto :goto_0

    .line 24524
    :cond_4
    add-int/lit8 v0, v1, 0x1

    move v1, v0

    goto :goto_1
.end method

```

## 0X032. 实际监听逻辑

当我们分析出他是如何监听并且跳转的之后,还有一个小问题,就是劫持出现的逻辑

因为在现象三的情况下他是不出现劫持的.这也和用户体验有关.比如你在进行输入用户名密码的时候,它进行劫持肯定是用户体验不好的.

其实这个问题很简单,**其出现劫持的时机为:通过对android软件EditorInfo.IME_ACTION_SEARCH方式的劫持**

这可以根据每次弹出来的键盘进行验证


# 0x04.总结



结论:非前端劫持,通过对android软件EditorInfo.IME_ACTION_SEARCH方式的劫持,并判断当前是否为指定的浏览器,是则跳转相应搜狗搜索,否则则直接不做处理

画成流程图

![p](http://7xkw0v.com1.z0.glb.clouddn.com/p.png)



