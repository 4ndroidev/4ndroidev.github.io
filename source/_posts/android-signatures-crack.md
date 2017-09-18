title: Android Apk 签名破解
categories: Android
tags:
  - Android
  - 反编译
date: 2017-04-18 15:10:23
---

阅读本文前，应先了解反编译工具的使用，和smali代码的插桩修改

> 前提：应用厂商为防止二次打包发布，一般会校验签名，甚至使用签名作为网络请求起算参数，防止服务器资源被盗用
> 作用：避免二次打包后签名校验导致无法运行问题，破解签名后，可插桩日志代码，分析应用请求接口
> 破解方法：很简单，伪造正确签名，欺骗程序读取正确签名


步骤：

1. 判断是否使用了签名校验
2. 获取正确签名并保存
3. 欺骗程序获取正确签名

<!-- more -->

# 判断是否使用了签名校验

使用`~/.android/debug.keystore`对应用进行重新签名，安装，检测能否正常使用
  ```shell
  # 重新签名脚本
  apk=xxx.apk
  apk_sign=${apk%.*}_sign.${apk##*.}
  jarsigner -verbose -keystore ~/.android/debug.keystore -signedjar $apk_sign $apk androiddebugkey -storepass android
  adb install -r $apk_sign
  ```


如果还能正常使用，代表没作签名校验，如果不能正常使用，解压apk，`grep "signatures" -r .`找出所有`signatures`的，举例

![image](/images/android-signatures-crack/find-all-signatures.jpg)

可见，应用可能在java和so中进行校验签名，由于欺骗java获取签名，使用`baksmali`和`smali`工具即可，难度在于欺骗so，下文主要讲如何欺骗so获取正确应用签名

# 获取正确签名并保存

安装过程，会调用`PagekManagerService`的`installPackage`方法，接着调用`PackageParser`对安装包进行解释，查看相应源码，我们可以知道，`PackageInfo`的`signatures`数组长度正常情况下为1，即只对`AndroidManifest.xml`签名进行保存，另外签名缓存路径为：`/data/system/packages.xml`
  ```java
  private void logSignature(String path){
    PackageManager packageManager = context.getPackageManager();
    PackageInfo packageInfo = packageManager.getPackageArchiveInfo(path,
                    PackageManager.GET_ACTIVITIES | PackageManager.GET_SIGNATURES);
    Signatures[] signatures = packageInfo.signatures;
    String signature = signatures[0].toCharsString();
    Log.e("signature", signature); //将signature字符串保存下来
  }
  ```

# 欺骗程序获取正确签名

> 通过apktool反编译的apk，smali可以随便改，而so不能随便修改，从获取签名函数可以看出，so获取系统签名，必须往下传`Context`实例，通过反射调用java方法，获取签名。因此，只需要对传给so的`Context`实例进行封装下，即可达到欺骗目的

1 使用ida打开相关so文件，查找`signatures`，由于本文只是为了欺骗，不打算看汇编代码（主要是知识还老师了）

![image](/images/android-signatures-crack/find-so-signatures.jpg)

2 查看`Context`相关反射调用，看so中的字符串常量即可，记录下来，封装时需要用到

![image](/images/android-signatures-crack/context-method.jpg)

上图可见会调用`Context`的`getSystemService`，`getPackageName`和`getPackageManager`方法

3 封装与调用

3.1 封装`Context`对象，由于`Application`集成了`Context`，直接继承`Application`，重写相应方法比较方便，重写获取系统签名方法，需要继承三个类，分别是`Application`，`PackageManager`，`PackageInfo`，代码如下

**FateApplication**继承**Application**
  ```java
  public class FakeApplication extends Application {
    private Context context;

    public FakeApplication(Context context) {
      this.context = context;
    }

    @Override
    public Object getSystemService(String name) {
      return context.getSystemService(name);
    }

    @Override
    public ApplicationInfo getApplicationInfo() {
      return context.getApplicationInfo();
    }

    @Override
    public PackageManager getPackageManager() {
      return new FakePackageManager();
    }

    @Override
    public String getPackageName() {
      return context.getPackageName();
    }
  }
  ```

**FakePackageManager**继承**PackageManager**
  ```java
  public class FakePackageManager extends PackageManager {
      
    @Override
    public PackageInfo getPackageInfo(String packageName, int flags) throws NameNotFoundException {
      return new FakePackageInfo();
    }

    // 此处省略一堆无关紧要的abstract方法的重写
  }
  ```

**FakePackageInfo**继承**PackageInfo**
  ```java
  public class FakePackageInfo extends PackageInfo {

    private final static String SIGNATURE = "${第二步保存的签名字符串}";

    public FakePackageInfo() {
      this.signatures = new Signature[]{new Signature(SIGNATURE)};
    }

  }
  ```

3.2 调用
    
使用`apktool`反编译apk，然后sublime text打开，搜索`loadLibrary`，找到java类，再查找传递context对象native函数，再调用此native函数时，创建`FakeApplication`对象

![image](/images/android-signatures-crack/context-call.jpg)
