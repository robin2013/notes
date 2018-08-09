#UIViewController的生命周期

##前言
UIViewController的生命周期函数:

- init(initWithNibName)  初始化
- loadView 加载自定义的视图(如果是从xib文件加载, 通常不需要添加此方法; 如果添加该方法, 系统则不会自动创建根view)
- viewDidLoad 视图加载完成, 可以进行布局,创建其他控件和获取数据
- viewWillAppear 视图即将出现
- viewDidAppear 视图已经显示完成
- viewWillDisappear 视图即将消失
- viewDidDisappear 视图已经消失
- viewWillLayoutSubviews 在view的layoutSubviews方法执行前调用
- viewDidLayoutSubviews 在view的layoutSubviews方法执行后调用
- didReceiveMemoryWarning 系统内存警告
- dealloc 销毁视图

###创建视图
从xib加载:
init-> viewDidLoad-> viewWillAppear-> viewDidAppear

从自定义视图加载:

init-> loadView -> viewDidLoad-> viewWillAppear->viewWillLayoutSubviews->(view)layoutSubviews -> viewDidLayoutSubviews -> viewDidAppear

代码:

```
//ThirdView.m

@implementation ThirdView
- (void)layoutSubviews {
    [super layoutSubviews];
    NSLog(@"ThirdView layoutSubviews");
}
@end
```

```
//ThirdViewController.m

@implementation ThirdViewController
- (id)init {
    if (self = [super init]) {
        NSLog(@"ThirdViewController init");
        return self;
    }
    return nil;
}
- (void)viewDidLoad {
    [super viewDidLoad];
    NSLog(@"ThirdViewController viewDidLoad");

}

- (void)loadView {
    NSLog(@"ThirdViewController loadView");
    ThirdView *view = [ThirdView new];
    self.view = view;
}

- (void)viewWillAppear:(BOOL)animated {
    [super viewWillAppear:animated];
    NSLog(@"ThirdViewController viewWillAppear");
}

- (void)viewDidAppear:(BOOL)animated {
    [super viewDidAppear:animated];
    NSLog(@"ThirdViewController viewDidAppear");
}

- (void)viewDidDisappear:(BOOL)animated {
    [super viewDidDisappear:animated];
    NSLog(@"ThirdViewController viewDidDisappear");
}

- (void)viewWillDisappear:(BOOL)animated {
    [super viewWillDisappear:animated];
    NSLog(@"ThirdViewController viewWillDisappear");
}

- (void)viewWillLayoutSubviews {
    [super viewWillLayoutSubviews];
    NSLog(@"ThirdViewController viewWillLayoutSubviews");

}

- (void)viewDidLayoutSubviews {
    [super viewDidLayoutSubviews];
    NSLog(@"ThirdViewController viewDidLayoutSubviews");
}
@end
```

输出:

```
2018-08-09 20:16:01.577931+0800 LifeCycle[42337:3300297] ThirdViewController init
2018-08-09 20:16:01.581844+0800 LifeCycle[42337:3300297] ThirdViewController loadView
2018-08-09 20:16:01.582142+0800 LifeCycle[42337:3300297] ThirdViewController viewDidLoad
2018-08-09 20:16:01.582462+0800 LifeCycle[42337:3300297] ThirdViewController viewWillAppear
2018-08-09 20:16:01.611263+0800 LifeCycle[42337:3300297] ThirdViewController viewWillLayoutSubviews
2018-08-09 20:16:01.611392+0800 LifeCycle[42337:3300297] ThirdView layoutSubviews
2018-08-09 20:16:01.611473+0800 LifeCycle[42337:3300297] ThirdViewController viewDidLayoutSubviews
2018-08-09 20:16:02.114443+0800 LifeCycle[42337:3300297] ThirdViewController viewDidAppear

```

###视图显示逻辑
Controller之间的显示隐藏逻辑
代码:

```
//ViewController.m

- (void)viewWillAppear:(BOOL)animated {
    [super viewWillAppear:animated];
    NSLog(@"ViewController viewWillAppear");
}

- (void)viewDidAppear:(BOOL)animated {
    [super viewDidAppear:animated];
    NSLog(@"ViewController viewDidAppear");
}

- (void)viewDidDisappear:(BOOL)animated {
    [super viewDidDisappear:animated];
    NSLog(@"ViewController viewDidDisappear");

}

- (void)viewWillDisappear:(BOOL)animated {
    [super viewWillDisappear:animated];
    NSLog(@"ViewController viewWillDisappear");
}
```

```
//SecondViewController.m

@implementation SecondViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    NSLog(@"SecondViewController viewDidLoad");
}

- (void)viewWillAppear:(BOOL)animated {
    [super viewWillAppear:animated];
    NSLog(@"SecondViewController viewWillAppear");
}

- (void)viewDidAppear:(BOOL)animated {
    [super viewDidAppear:animated];
    NSLog(@"SecondViewController viewDidAppear");
}

- (void)viewDidDisappear:(BOOL)animated {
    [super viewDidDisappear:animated];
    NSLog(@"SecondViewController viewDidDisappear");
    
}

- (void)viewWillDisappear:(BOOL)animated {
    [super viewWillDisappear:animated];
    NSLog(@"SecondViewController viewWillDisappear");
}

@end
```

ViewController push SecondViewController输出:

```
2018-08-09 20:20:52.184795+0800 LifeCycle[42496:3307472] ViewController viewWillDisappear
2018-08-09 20:20:52.184965+0800 LifeCycle[42496:3307472] SecondViewController viewWillAppear
2018-08-09 20:20:52.711445+0800 LifeCycle[42496:3307472] ViewController viewDidDisappear
2018-08-09 20:20:52.711620+0800 LifeCycle[42496:3307472] SecondViewController viewDidAppear

```

SecondViewController pop back to ViewController输出:

```
2018-08-09 20:23:37.193745+0800 LifeCycle[42496:3307472] SecondViewController viewWillDisappear
2018-08-09 20:23:37.193930+0800 LifeCycle[42496:3307472] ViewController viewWillAppear
2018-08-09 20:23:37.698928+0800 LifeCycle[42496:3307472] SecondViewController viewDidDisappear
2018-08-09 20:23:37.699078+0800 LifeCycle[42496:3307472] ViewController viewDidAppear
```

总结:

页面间push的显示逻辑总是先触发Disappear, 然后触发Appear

### 其他

- 收到`didReceiveMemoryWarning`方法调用时,吗我们需要把不是位于顶级视图的view全部释放掉(有些业务上必须留存的视图除外), 以减轻内存压力; 恢复页面时, Controller会重新走`loadView`流程, 此时需要注意业务逻辑是否重复

- 除了`init`方法, 其他方法在生命周期内都可能多以调用

- `loadView` 实现该方法, 却没有给`self.view`赋值的情况下, 不会产生根view

[demo地址](https://github.com/robin2013/ViewControllerLifeCycle)