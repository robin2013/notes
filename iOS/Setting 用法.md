#App Settings.Bundle

###写在前边
  Settings.Bundle 是App的配置项, 用户和测试可以在外围对于App的部分信息进行修改, 是的App的运行环境变的可配置, 对于测试尤其使用.

我们也可以结合Build Configuration针对不同的编译环境进行个性化设置, 当然, 包括移除Setting(有时候Production版本是不需要Setting的)

本文分为三个部分:

- Setting 用法: 详细的数据结构
- 应用场景: 使用注意事项
- 动态配置: 动态添加Setting

## Setting 用法


Settings.Bundle支持六种配置项分别是：

- TextField
- MultiValue
- Title
- Group
- Slider
- ToggleSwitch

###TextField

数据结构:

![](http://pdlfktchy.bkt.clouddn.com/18-8-17/94067429.jpg)

字段:

- Type 类型，默认是 Text Field
- Title  显示的Title
- Identifier 标识符，用来获取配置项的配置内容
- Default Value  默认值
- Autocorrection Style 自动纠错(我很讨厌这个功能)
- Autocapitalization Style 大小写类型(字符 句子 全部) 

效果:
![](http://pdlfktchy.bkt.clouddn.com/18-8-17/12394366.jpg)
###MultiValue

数据结构:

![](http://pdlfktchy.bkt.clouddn.com/18-8-17/86623125.jpg)

字段:

- Type 类型，默认是 Multi Value
- Title  显示的Title
- Identifier 标识符，用来获取配置项的配置内容
- Default Value  默认值
- Values 可供选择的值数组
- Titles 与选择值对应的显示文本

效果:  
![](http://pdlfktchy.bkt.clouddn.com/18-8-17/60716550.jpg)
![](http://pdlfktchy.bkt.clouddn.com/18-8-17/2932381.jpg)

###Title

数据结构1:

![](http://pdlfktchy.bkt.clouddn.com/18-8-17/11124522.jpg)

- Type 类型，默认是Title
- Title  显示的Title
- Identifier 标识符，用来获取配置项的配置内容
- Default Value  默认值

效果:

![](http://pdlfktchy.bkt.clouddn.com/18-8-17/94627942.jpg)

数据结构2:
![](http://pdlfktchy.bkt.clouddn.com/18-8-17/32102128.jpg)

- Values 可供选择的值数组
- Titles 与选择值对应的显示文本

相对于结构1, 多出了Values和Titles两个字段, 此时要注意的是,  Default Value只能取自于Values中的某一个元素, 否则显示为空

效果图:
![](http://pdlfktchy.bkt.clouddn.com/18-8-17/3026091.jpg)

###Group

Group可以认为是一个用在做分组的文字, 就像Grouped TableView的效果

数据结构:

![](http://pdlfktchy.bkt.clouddn.com/18-8-17/56370864.jpg)

- Type 类型，默认是Group
- Title  显示的Title

效果:

![](http://pdlfktchy.bkt.clouddn.com/18-8-17/8074124.jpg)


###ToggleSwitch
选择开关

数据结构:
![](http://pdlfktchy.bkt.clouddn.com/18-8-17/66468050.jpg)


- Type 类型，默认是 Toggle Switch
- Title  显示的Title
- Identifier 标识符，用来获取配置项的配置内容
- Default Value  默认值
- Value for OFF 关闭时对应的值(貌似没用)
- Value for ON 打开时对应的值(貌似没用)

效果图:

![](http://pdlfktchy.bkt.clouddn.com/18-8-17/84491440.jpg)

###Slider

数据结构:
![](http://pdlfktchy.bkt.clouddn.com/18-8-17/64114689.jpg)

- Type 类型，默认是 Slider
- Title  显示的Title
- Identifier 标识符，用来获取配置项的配置内容
- Default Value  默认值
- Minimum Value：最小值
- Maximun Value：最大值
- Min Value Image Filename：最小值端图片
- Max Value Image Filename：最大值端图片

注意:

**Image Filename 不能放在主文件夹中，而是需要放在Settings捆绑包中，才能够通过 Min/Max Value Image Filename 设置使用**, 如:

![](http://pdlfktchy.bkt.clouddn.com/18-8-17/78712980.jpg)

效果图:

![](http://pdlfktchy.bkt.clouddn.com/18-8-17/77013119.jpg)


####关于字段的官方说明:

![](https://apppie.files.wordpress.com/2016/03/2016-03-14_08-11-55.png?w=900)

##应用场景

添加Settings.Bundle的最终目的还是和app进行交互(废话),Settings中的任何选型都可以用代码获得


```

it is important to understand that until the user actually changes the value of
 the setting nothing is actually set. If you check for the setting in your 
 application it will actually return nil unless you set a default value. 

```
需要注意的是, 在用户实际修改了setting里的数据之前 , 这时配置数据并没有真正的写入到app的`NSUserDefaults`中, 此时用`NSUserDefaults`获取的值都是空的. 比如,直接调用以下代码:


```

- (void)testSetting {
    NSUserDefaults *defaults  = [NSUserDefaults standardUserDefaults];
    NSString *text = [defaults valueForKey:@"selecter"];
    if (text) {
        NSLog(text);
    } else {
        NSLog(@"nothing");
    }
}

```

结果将是:

```
2018-08-17 18:01:27.920247+0800 AppSetting[8672:394359] nothing
```
![](http://pdlfktchy.bkt.clouddn.com/18-8-17/26171958.jpg)

也就是说此时的setting并未生效, 但是如果用户在setting中修改过`selecter`(setting中的`选项`)一个值, 获得的数据将不为空:

```
2018-08-17 18:08:22.177436+0800 AppSetting[8795:404984] 1

```

![](http://pdlfktchy.bkt.clouddn.com/18-8-17/19231675.jpg)

对此, 我们的处理方案是, 在`- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions`方法中调用以下函数, 将所有的setting数据注册到app中:

```
- (void)registerDefaultsFromSettingsBundle
{
    NSString *settingsBundle = [[NSBundle mainBundle] pathForResource:@"Settings" ofType:@"bundle"];
    if(!settingsBundle) {
        NSLog(@"Could not find Settings.bundle");
        return;
    }
    NSDictionary *settings = [NSDictionary dictionaryWithContentsOfFile:[settingsBundle stringByAppendingPathComponent:@"Root.plist"]];
    NSArray *preferences = [settings objectForKey:@"PreferenceSpecifiers"];
    NSMutableDictionary *defaultsToRegister = [[NSMutableDictionary alloc] initWithCapacity:[preferences count]];
    for(NSDictionary *prefSpecification in preferences) {
        NSString *key = [prefSpecification objectForKey:@"Key"];
        if(key) {
            [defaultsToRegister setObject:[prefSpecification objectForKey:@"DefaultValue"] forKey:key];
        }
    }
    [[NSUserDefaults standardUserDefaults] registerDefaults:defaultsToRegister];
}

```

[代码](https://github.com/robin2013/AppSetting)

##动态配置
一般情况下, 我的环境配置(Build Configuration)至少分四种:

- Dev 开发
- Release 测试
- Staging 预发布
- Production 生产
然后分别给予不同的宏定义, 以保证我们切换编译环境时, 做到代码零修改, [Build Configuration配置](http://blog.stablekernel.com/ios-build-configurations-and-schemes/)

###1. 创建工程 `DynamicSetting `
###2. 创建相关目录

在工程目录下创建文件夹`Setting`以及子文件夹`Release`和`Staging`(一般生产环境不需要setting, 当然这要依据具体的业务需求), 并添加到工程中(不添加到工程也行, 此处只是为了方便修改)

![](http://pdlfktchy.bkt.clouddn.com/18-8-17/91263887.jpg)

###3. 创建配置
点击工程文件, 选中`info`, 在`Configurations`下点击`+`, 复制一个`Configuration`(我一般复制`Release`, 这样发出去的ipa都去掉日志等不必要的操作 具体依据业务需求修改)
![](http://pdlfktchy.bkt.clouddn.com/18-8-17/68005451.jpg)
###4. 添加Setting.bundle
新建Setting.Bundle, 但不要添加到当前`Target`中, 并分别保存到`Release`和`Staging`文件夹下:

**不要添加到当前`Target`中**

**不要添加到当前`Target`中**

**不要添加到当前`Target`中**

![](http://pdlfktchy.bkt.clouddn.com/18-8-17/97003751.jpg)

创建完成后的工程, 是这个样子的:

![](http://pdlfktchy.bkt.clouddn.com/18-8-17/23088057.jpg)

运行XCode, 我们在模拟器中的`Setting`中找不到对应的`DynamicSetting`(废话, 都没添加到`Target`, 能显示才怪!!!)

###5. 添加拷贝脚本
不添加到`Target`的原因就是 , 我们要用脚本吧`Setting.Bundle`拷贝到ipa中.
选中"Target", 切换到`Build Phases`, 点击`+`, 选中`New Run Script Phase`

![](http://pdlfktchy.bkt.clouddn.com/18-8-17/63079338.jpg),

添加脚本:

```
if [ ${CONFIGURATION} = "Debug" ]; then
cp -r ${PROJECT_DIR}/${PROJECT_NAME}/Setting/Release/Settings.bundle ${BUILT_PRODUCTS_DIR}/${PRODUCT_NAME}.app
fi

if [ ${CONFIGURATION} = "Release" ]; then
cp -r ${PROJECT_DIR}/${PROJECT_NAME}/Setting/Release/Settings.bundle ${BUILT_PRODUCTS_DIR}/${PRODUCT_NAME}.app
fi

if [ ${CONFIGURATION} = "Staging" ]; then
cp -r ${PROJECT_DIR}/${PROJECT_NAME}/Setting/Staging/Settings.bundle ${BUILT_PRODUCTS_DIR}/${PRODUCT_NAME}.app
fi
```
![](http://pdlfktchy.bkt.clouddn.com/18-8-17/63987909.jpg)

###6 修改Setting
####6.1 开发环境设置
修改`Release`文件加下的`.plist`文件 :

![](http://pdlfktchy.bkt.clouddn.com/18-8-17/2011911.jpg)

选中`edit scheme`, 切换`Build Configuration`:

![](http://pdlfktchy.bkt.clouddn.com/18-8-17/17287599.jpg)

运行XCode, 在模拟器上查看 `Setting`:

![](http://pdlfktchy.bkt.clouddn.com/18-8-17/97153383.jpg)

####6.1 预发布环境设置
修改`Staging`文件加下的`.plist`文件 :

![](http://pdlfktchy.bkt.clouddn.com/18-8-17/49135230.jpg)

选中`edit scheme`, 切换`Build Configuration`:

![](http://pdlfktchy.bkt.clouddn.com/18-8-17/44300752.jpg)

运行XCode, 在模拟器上查看 `Setting`:

![](http://pdlfktchy.bkt.clouddn.com/18-8-17/59523899.jpg)



note: **由于开发环境我们没有添加相应脚本, 所以在`Production`下运行app, 是没有`Setting`的**

[代码](https://github.com/robin2013/DynamicSetting)


##参考

[Adding a settings bundle to an iPhone App](https://useyourloaf.com/blog/adding-a-settings-bundle-to-an-iphone-app/)

[Application Settings Bundle Part – I](http://sugartin.info/2012/08/16/application-settings-bundle-part-i/)

[iOS之Settings.Bundle](https://www.jianshu.com/p/9e1b21789076)