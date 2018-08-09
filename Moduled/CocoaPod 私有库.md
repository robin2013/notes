#CocoaPod 私有库

###创建私有Spec仓

1.在git服务器上创建一个代码库, 并将代码库添加到本地:

```
$ pod repo add HomePageModule https://git.coding.net/RobinCui/HomePageModuleSpec.git
```

查看在 Finder 目录 `~/.cocoapods/repos`， 可以发现增加了一个 HomePageModule 的储存库

###创建代码库
- 在git服务器上创建私有代码库, 创建时添加 `MIT License` 和 `README`,  并`clone`到本地
- 进入代码根目录, 添加代码文件
- 创建`.podspec `

```
pod spec create ModeuleA
```
编辑`ModeuleA . podspec `文件

- 创建`.swift-version`文件用来知道swift版本,如:
 ```
 $ echo "3.0" > .swift-version
 ```
- 检查类库完整性:
 ```
 $ pod lib lint
 ```
 
 ###给仓库打标签
 验证成功后，将仓库提交到远程，然后给仓库打上标签并将标签也推送到远程。

标签相当于将你的仓库的一个压缩包，用于稳定存储当前版本。标签号与``ModeuleA.podspec 中的`s.version = "1.0.0`"的版本号一致` 1.0.0`

```
创建标签
$ git tag -a 1.0.0 -m '标签说明' 
推送到远程
$ git push origin --tags
```

###发布.podspec

```
pod repo push HomePageModuleSpec ModeuleA.podspec
```

###私人pod库的使用
使用私人pod库的需要在Podflie中添加这句话，指明私有版本库地址

```
source 'https://git.coding.net/RobinCui/HomePageModuleSpec.git'
```

若有还使用了公有的pod库，需要把公有库地址也带上

```
source 'https://github.com/CocoaPods/Specs.git'
```