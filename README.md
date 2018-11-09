## 简介
sdk-editor是为了实现对APP引用的第三方SDK进行修改而开发的Gradle插件，插件利用Android Plugin提供的Transform API来干预Build流程，实现对三方SDK中特定类的替换修改，不影响APP运行性能，也不会增加生成的APK体积。
## 适用场景
1）修复SDK中存在的Bug；

2）暴露出SDK某些未提供的接口；

3）扩展SDK功能；

4）其他需要修改SDK已有类的需求；
## 用法
#### 1. 在根项目的build.gradle文件中添加插件依赖：
```gradle
buildscript {
    dependencies {
        classpath 'com.iwhys.sdkeditor:plugin:1.1.0'
    }
}
```
#### 2. 在项目主模块（app module）的build.gradle文件应用插件：
```gradle
apply plugin: 'sdk-editor'
```
#### 3. 找到三方SDK中需要修改的类文件（以下称为Bug类），在app module中新建与Bug类同包名同类名的新类（以下称为Fix类），同时拷贝Bug类的内容到Fix类，给Fix类添加类注解@ReplaceClass，在注解的值中标记该类所在SDK的名字，最后在Fix类中实现要修改的内容即可。

下面以demo模块中libs引用的三方SDK DuappsAd-HW-v1.1.1.6-release.aar为例，我们需要修改SDK中的com.duapps.ad.DuNativeAd类，在其中添加广告请求监听器，修改流程如下：

1）在demo工程的main/java下新建包com/duapps/ad；

2）在1)新建的包com/duapps/ad中新建DuNativeAd类，并拷贝原SDK中DuNativeAd类的内容；

3）在新建的DuNativeAd类中添加注解@ReplaceClass("DuappsAd-HW-v1.1.1.6-release")；

4）在新建的DuNativeAd类中添加需要新增的广告监听器逻辑即可；

5）在终端(Terminal)中执行命令：gradlew clean assembleDebug，能够看到插件替换类的全过程日志，查看最终生成的APK文件，可以发现目标类已经被顺利修改；
```java
@ReplaceClass("DuappsAd-HW-v1.1.1.6-release")
public class DuNativeAd {
    ...
}
```
## 高级用法
如果有多个项目用到了同一个需要修改的SDK，为了在多个项目中共享修复后的代码，我们把修复代码达成一个单独jar/aar包，以便在不同项目之间共享。具体流程如下：

1）在任意项目中新建一个module（如demo中的library_fix）；

2）添加对需要修复的SDK（如：DuappsAd-HW-v1.1.1.6-release.aar）以及插件注解包（com.iwhys.classeditor:domain:1.1.0）的依赖，并把修复代码复制到该module的源码目录；

3）在终端(Terminal)中执行命令：gradlew library_fix:build，即可在该module的build/output目录中看到生成的aar文件；

4）给aar文件(或者其中的jar文件)指定一个名字，如du_hack，拷贝该文件到主module(app module)的libs文件夹，并添加对该文件的依赖；

5）在主module(app module)的build.gradle中指定用于SDK修复的jar/aar包信息即可，格式如下：
```gradle
sdkEditor {
    // 这里是一个数组，有多个用于修复的包，需要用","分隔开
    extraJarNames = ['hack_du']
}
```
## 常见问题
#### 1. 新建与Bug类同包名的Fix类时，编译器提示"Package 'xxx.xxx.xxx' clashes with class of same name"
这种情况是因为包路径中的包和类重名了，我们可以通过把java类转换为kotlin类来修复这个问题。

注意：此时需要添加kotlin相关的支持，且因为kotlin编译器在把类编译为class的时候，默认会把文件名改为：原文件名+kt，因此在kotlin版的Fix类中添加文件命名注解 @file:JvmName("Bug类的名称")
#### 2. 在新建的Fix类中，存在部分形如"a.b.c"的类无法正确的导入，或者导入之后与当前类的成员重名
这种情况我们可以把类改为kotlin版，并利用kotlin提供的 import xxx as yyy功能，对导入的有问题的类进行重命名。
#### 3. 新建的Fix类时，如果其所在包的名字同级已经存在一个同名的类（如已存在类com.a.a，Fix类路径com.a.a.Fix,则IDE提示"Redeclare a")
我们可以通过"高级用法"的笨办法，新建module把已存在类com.a.a和Fix类com.a.a.Fix分别放在不同的module来实现。
## 原理分析