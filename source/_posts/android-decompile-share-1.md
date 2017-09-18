title: Android反编译分享
date: 2017-05-02 14:56:03
tags:
  - Android
  - 反编译
---

> 声明：本文仅供学习使用，如有侵犯，请通知我，24小时内删除！
> 前言：反编译工作特点：1.体力活  2.需要耐心  3.坚持不懈  4.熟能生巧

纲目

- 反编译作用
- 反编译工具
- Smali简介
- 代码插桩
- 实战例子（如何获取网易云音乐接口，查看整个反编译流程）

<!-- more -->

## 反编译作用

对于公司而言

- 了解行业内对手特点功能的实现原理。
- 修改应用，添加作弊插件（常见于游戏公司）。

对于个人而言

- 集成广告二次打包发布市场，挣广告费（最常见）。
- 修改系统应用，添加功能，制作特色刷机包。
- 单纯为了学习（我）。

## 反编译工具

主要工具： Apktool，dex2jar，JD-GUI

额外工具： ida，HexEdit

### Apktool

> 功能：将Apk反编译得到资源文件及`smali`代码，将Jar反编译得到`smali`代码，以及回编译。其中将`dex`文件反编译成`smali`的工具为`baksmali.jar`，回编译`smali`成`dex`的为`smali.jar`

- github: [https://github.com/iBotPeaches/Apktool](https://github.com/iBotPeaches/Apktool)

- website: [https://ibotpeaches.github.io/Apktool](https://ibotpeaches.github.io/Apktool)

- usage: [https://ibotpeaches.github.io/Apktool/documentation/](https://ibotpeaches.github.io/Apktool/documentation/)

- smali syntax: [http://www.netmite.com/android/mydroid/dalvik/docs/dalvik-bytecode.html](http://www.netmite.com/android/mydroid/dalvik/docs/dalvik-bytecode.html)

- smali/baksmali github : [https://github.com/JesusFreke/smali.git](https://github.com/JesusFreke/smali.git) 

常用命令

  ```shell
  # 反编译
  apktool d xxx.apk

  # 回编译
  apktool b xxx

  # 参数
  # -r,--no-res             忽略资源文件
  # -s,--no-src             忽略代码文件
  ```

### dex2jar

> 将dex文件反编译成`*.class`集合的jar文件，之后可以使用`JD-GUI`工具查看

- github: [https://github.com/pxb1988/dex2jar](https://github.com/pxb1988/dex2jar)

常用命令: sh d2j-dex2jar.sh classes.dex

### JD-GUI

> 用于查看`*.class`集合的jar文件，如第三方sdk的jar包，或者dex2jar转换得到的jar文件

![image](/images/android-decompile-tools/jd-gui.jpg)

## Smali简介

> 摘自[http://blog.csdn.net/wdaming1986/article/details/8299996](http://blog.csdn.net/wdaming1986/article/details/8299996)
> Davlik字节码中，寄存器都是32位的，能够支持任何类型，64位类型（Long/Double）用2个寄存器表示；
> Dalvik字节码有两种类型：原始类型；引用类型（包括对象和数组）

原始类型

|标记|类型|
|---|---|
|V|void， 只用于返回值|
|Z|boolean|
|B|byte|
|S|short|
|C|char|
|I|int|
|J|long(64bits)|
|F|float|
|D|double(64bits)|

对象类型

> Lpackage/className;  

- L：表示这是一个对象类型
- package：该对象所在的包名
- className： 类的simpleName

数组的表示形式：
[I  :表示一个整形的一维数组，相当于java的int[];
对于多维数组，只要增加[ 就行了，[[I = int[][];注：每一维最多255个； 

对象数组的表示形式：
[Ljava/lang/String    表示一个String的对象数组；

方法的表示形式：
Lpackage/className;—>methodName(III)Z  详解如下：
Lpackage/className  表示类型
methodName   表示方法名
III   表示参数（这里表示为3个整型参数）
说明：方法的参数是一个接一个的，中间没有隔开；
          
字段的表示形式：
Lpackage/className;—>fieldName:Ljava/lang/String;
即表示： 包名，字段名和各字段类型

有两种方式指定一个方法中有多少寄存器是可用的：
.registers  指令指定了方法中寄存器的总数
.locals     指令表明了方法中非参寄存器的总数，出现在方法中的第一行

举例：

java源码
  ```java
  package com.decompile;

  import android.os.Bundle;
  import android.support.v7.app.AppCompatActivity;
  import android.util.Log;

  public class MainActivity extends AppCompatActivity {

      @Override
      protected void onCreate(Bundle savedInstanceState) {
          super.onCreate(savedInstanceState);
          setContentView(R.layout.activity_main);
          
          log("使用私有对象方法，p0是MainActivity.this这个对象实例，p1是第一个参数");

          sLog("使用公有静态方法，p0是第一个参数");
      }

      private void log(Object o) {
          Log.e("private object method", String.valueOf(o));
      }

      public static void sLog(Object o) {
          Log.e("public static method", String.valueOf(o));
      }
      
  }
  ```

smali代码
  ```smali
  .method private log(Ljava/lang/Object;)V
      .locals 2
      .param p1, "o"    # Ljava/lang/Object;

      .prologue
      const-string v0, "private object method"

      invoke-static {p1}, Ljava/lang/String;->valueOf(Ljava/lang/Object;)Ljava/lang/String;

      move-result-object v1

      invoke-static {v0, v1}, Landroid/util/Log;->e(Ljava/lang/String;Ljava/lang/String;)I

      return-void
  .end method

  .method public static sLog(Ljava/lang/Object;)V
      .locals 2
      .param p0, "o"    # Ljava/lang/Object;

      .prologue
      const-string v0, "public static method"

      invoke-static {p0}, Ljava/lang/String;->valueOf(Ljava/lang/Object;)Ljava/lang/String;

      move-result-object v1

      invoke-static {v0, v1}, Landroid/util/Log;->e(Ljava/lang/String;Ljava/lang/String;)I

      return-void
  .end method

  # virtual methods
  .method protected onCreate(Landroid/os/Bundle;)V
      .locals 1
      .param p1, "savedInstanceState"    # Landroid/os/Bundle;

      .prologue
      invoke-super {p0, p1}, Landroid/support/v7/app/AppCompatActivity;->onCreate(Landroid/os/Bundle;)V

      const v0, 0x7f04001b

      invoke-virtual {p0, v0}, Lcom/decompile/MainActivity;->setContentView(I)V

      const-string v0, "\u4f7f\u7528\u79c1\u6709\u5bf9\u8c61\u65b9\u6cd5\uff0cp0\u662fMainActivity.this\u8fd9\u4e2a\u5bf9\u8c61\u5b9e\u4f8b\uff0cp1\u662f\u7b2c\u4e00\u4e2a\u53c2\u6570"

      invoke-direct {p0, v0}, Lcom/decompile/MainActivity;->log(Ljava/lang/Object;)V

      const-string v0, "\u4f7f\u7528\u516c\u6709\u9759\u6001\u65b9\u6cd5\uff0cp0\u662f\u7b2c\u4e00\u4e2a\u53c2\u6570"

      invoke-static {v0}, Lcom/decompile/MainActivity;->sLog(Ljava/lang/Object;)V

      return-void
  .end method
  ```

对象方法调用：
  ```smali
  (invoke-direct|invoke-virtual) {xx, ...params}, Lpackage/className;->methodName(...paramTypes)returnType
  ```

invoke-direct除了用于对象私有方法，还可以用于对象的初始化
invoke-virtual用于对象公有方法

xx：Lpackage/className的实例
params：为参数列表
paramType：参数列表对应类型
returnType：返回值类型

静态方法调用：
  ```smali
  invoke-static {...params}, Lpackage/className;->methodName(...paramTypes)returnType
  ```
invok-static用于调用静态方法
静态方法与对象方法，区别是参数列表第一个参数不是Lpackage/className的实例对象

## 代码插桩

经过上述java和smali代码对照，再查看一下smali语法，由于我们没有java源码，无法调试，只能静态分析代码，然后通过一些log进行跟踪分析别人的代码逻辑，很容易联想到代码插桩。通过代码插桩，我们可以修改第三方应用，实现某些功能。目前市面上基本上通过xposed来实现自动抢红包，实际上，也可以通过反编译修改微信应用达到效果，之前实现过，但不想维护，没玩下去。

代码插桩，需要讲究技巧。在Android Studio和Eclipe中编写好自己的逻辑代码，让其生成apk，然后同时反编译第三方应用和自己的逻辑代码应用，然后对代码进行合并。

如之前弄的微信红包（由于之前没考虑6.0系统运行时权限问题，后面取消了这个微信红包分享文章）。

![image](/images/android-decompile-share-1/android-lucky-money-insertcode.jpg)

由上图可知，我所谓的技巧有

1. 伪造所依赖的类和字段，标上`@Deprecated`注解，表示这部分代码无需进行插桩，只是为了Android Studio能正常编译，生成插桩代码应用
2. 在目标类，编写相关逻辑代码，当然方法名必须清晰，避免以后找不回来
3. 反编译目标应用和插桩代码应用，进行smali代码合并
4. 然后在目标类中的关联方法中调用我们自己写的逻辑方法，达到效果

## 实战例子

> 因为很久之前就反编译获取了网易云音乐接口，例子应用：neteasecloudmusic_v3.3.2.apk

### 抓包

> 使用手机端PacketCapture或者电脑端Charles进行抓包，查看网络请求详情，注意，不一定能抓到包，有些应用做了仿代理访问，抓不了包的。

![image](/images/android-decompile-share-1/android-capture-netease-request.png)![image](/images/android-decompile-share-1/android-capture-netease-response.png)

这个请求为点击启动页面的**立即体验**按钮发出，从图可知，请求参数只有`params`这个加密表单数据，关键字为`params`，另外请求头`Cookie`看上去需要组装`deviceId`等参数。根据最后反编译结果，这里返回结果中的`Set-Cookie`中的cookie字段是用于后续请求的。即这个请求在启动时必须发起，后续请求都需要`MUSIC_A`和`__csrf`这两个cookie

### 反编译

使用Apktool将neteasecloudmusic_v3.3.2.apk反编译成资源文件和smali代码文件

> apktool d neteasecloudmusic_v3.3.2.apk
 
  ```shell
  ➜  org  tree -L 1
  .
  ├── AndroidManifest.xml
  ├── apktool.yml
  ├── assets
  ├── lib
  ├── original
  ├── res
  ├── smali
  ├── smali_classes2
  └── unknown

  7 directories, 2 files
  ```

使用dex2jar将neteasecloudmusic_v3.3.2.apk的dex文件转成jar文件，使用jd-gui查看

> sh dex2jar.sh classes.dex 
> sh dex2jar.sh classes2.dex

![image](/images/android-decompile-share-1/android-jd-gui-netease-string-encrypt.jpg)

### 清理障碍

刚才我们知道`params`是请求的关键字，那么使用`sublime text`编辑器搜索`"params"`，你会发现搜索不到任何结果，上一张图jd-gui的图`System.loadLibrary`方法，会调用`a.c`方法处理字符串。那意思就是网易云音乐将所有字符串都加密了，通过`a.auu.a.c`方法进行解密。

![image](/images/android-decompile-share-1/android-jd-gui-netease-base64.jpg)

打开`a.auu.a`这个类，我们随便拖拖界面，发现有`bad base64`，那么可以猜到这个字符串加密为`base64`加密，当然也有它的一些逻辑。

使用jd-gui能够将dex2jar生成的jar文件，保存成可阅读的java文件，那么导出。接着编写java程序（随便写写，达到目的即可），将所有机密字符串解密。

根据smali语法看混淆代码静态分析，最终解密字符串代码如下：
  ```java
  public static String decode(String str) {
    byte[] bytes = Base64.decode(str, Base64.DEFAULT);
      return new String(xor(bytes));
    }

  private static byte[] xor(byte[] source) {
    int len = source.length;
    int len2 = "Encrypt".length();
    for (int i = 0, j = 0; i < len; i++, j++) {
      j %= len2;
      source[i] = (byte) (source[i] ^ "Encrypt".charAt(j));
    }
    return source;
  }

  public static String encode(String str) {
    return Base64.encodeToString(xor(str.getBytes()), Base64.DEFAULT);
  }
  private static String key = "a.auu.a.c";
  private static String handleLine(String source){
    try{
    while(source.indexOf(key)!=-1){
      int index = source.indexOf(key);
      String end = source.substring(index);
      String gone = source.substring(index, index+end.indexOf(")")+1);
      String encrypt = gone.substring(gone.indexOf("\"")+1, gone.lastIndexOf("\""));
      source = source.replace(gone, "\""+decode(encrypt)+"\"");
    }
    while(source.indexOf("a.c(\"")!=-1){
      int index = source.indexOf("a.c(\"");
      String end = source.substring(index);
      String gone = source.substring(index, index+end.indexOf(")")+1);
      String encrypt = gone.substring(gone.indexOf("\"")+1, gone.lastIndexOf("\""));
      source = source.replace(gone, "\""+decode(encrypt)+"\"");
    }
    }catch(Exception e){
    }
    return source;
  }
    
  private static void process(File file){
    File[] files = file.listFiles();
    for(int i=0;i<files.length;i++){
      File f = files[i];
      if(f.isDirectory()){
        process(f);
      }else{
        handleFile(f);
      }
    }
  }
    
  private static void handleFile(File source){
    String path = source.getPath();
    File file = new File(path+"bak");
    source.renameTo(file);
    try{
      InputStream is = new FileInputStream(file);
      BufferedReader br = new BufferedReader(new InputStreamReader(is));
      String line;
      File dest = new File(path);
      OutputStream os = new FileOutputStream(dest);
      BufferedWriter bw = new BufferedWriter(new OutputStreamWriter(os));
      while((line=br.readLine())!=null){
        bw.write(handleLine(line)); 
        bw.write("\n");
      }
      bw.close();
      br.close();
    }catch(IOException e){
      System.out.println(e.getMessage());
    }
    file.delete();
  }

  public static void main(String[] args){
    process(new File("保存的java文件路径"));
  }
  ```
再次使用`sublime text`搜索`"params"`字段，你会发现有结果，如下：

![image](/images/android-decompile-share-1/android-sublime-text-netease-params.jpg)

从图，我们看到了曙光，因为只要在这个地方通过代码插桩，就能把加密前整个请求参数列表全部打印出来，然后分析每个参数的来源。

插桩代码得到的log（这是提前揭晓的结果，中间还有障碍）
  ```log
  05-03 14:45:58.556 2137-2185/? E/params: {"\/api\/cdns":"","\/api\/v1\/vehiclefm\/visible":"","\/api\/log\/mam\/freq\/get":"","\/api\/android\/version":"{\"carrier\":\"unicom\",\"ismiui\":false}","\/api\/copyright\/pay_fee_message\/config":"","header":{"MUSIC_A":"47e30a9b79cacd30979d5c02186a8611851676624cde1c280b151856c9fd0d5c7c829645fe34e3c88d2e96001ea988b7446253d594d49cc2093469c45a9b7fe670500f1ae8f45856590694c120e20fee05dbee31830798be75c7cd93f1d9b7ac87577e38f566c25a80bec5f4143606251f977bdb2e8a00e70493e93303df938bbf122d59fa1ed6a2","__csrf":"0f2529b2f4466890e77e1a176a39ba7d","deviceId":"MzUyODI0MDYwNzk1OTU5CTAwOmVlOmJkOjQ0OmI3OmQ2CThjM2JhNGY3MjNmNjlhM2YJSEM0NlNXWTAxMjI0","appver":"3.3.2","os":"android","osver":"5.0.2","mobilename":"HTCM8Sw","resolution":"1776x1080","channel":"google","requestId":"1493793958567_865"}}
  05-03 14:45:58.556 2137-2185/? E/params: /api/batch
  05-03 14:45:58.566 2137-2185/? E/params: C561B86F974CBA39FB9AB3222AC3E77103E75A6A409D4B3CC17C4A7671C9FAD03F74A144F4E75C97B372C380BC2A6329450A2E19AC4BA84F673435A636D90439DD4D5E0E685EC201677DF834E5CD93103AFFEEA66517CD5A6593811E02F1A4BF3257B48D3EBBA5B7B7792621CB9A484E5C39FE31174DD5284BDAE946FC9D308912E015DA16629D64D53F3EE0236203C0B1690A2AE6BDC47B8BB040C746DD8AB746E2AA107B1AE8C4BC452B9C82470F6C771CAF0396B00AD43C2884E25DB41A49FEF1ED8294EDC325C712EC63145D211BFF041BF51003503DEA806F0AF6039F2FB4E2098D15592353D76485D102A81D5CA3304EFFF3044027E2855539BAA021073D049A4F894E560E4E95BCDC7E7E98988599215CD80D429C9EEAB3CE7C1B2D9034CB31BE681634B95201F0E181125F1ED5FECFC6D5B2D07CA8B98324994248E9403A5F7A65B34EEE8BE8B6E0316B943DD3B62BFEB042C119BC9C86896CCB6F70306FE1646792E2E4769E8DD453078DBEABD7285E77B6B41F6CF8217ACB795CA0214CBE467637D253A97A30F1D91BCCD13D9EEC5611018645B3F4BEC780FE4EA0B11209EC9CC951504A083DAA6877CD216C815EE33CC33EE0D0F10EF02B3F3C630DBDFB365D6FF61C6EEA398C13BAF5BAA6CE0E34786855FD85DE360A227D0F52FED554FB63B8044E6E21E494E92CAFAA63DB0FB4C5201DB5646F2A03D40AEEA031452D4D0B72B1775DC7BFE04EBD8B1E78E7067B31569460F1CA15890998592E2B0B18A540D3D46F671B82E601688C79A3D30CF201A89520E030E5823B568CDC1DF69493E31B49AA82DA5A1C5F5279BF8425F96A5DE841EEC729B2C81DD3C3E70BE4F336A685E96CE3B47CA3A4AA9E8ACABBCD2D040153E20C6EEA33D7E1ECBE7850B450C1A3DC60C66BD777A3ED4823023EEB3E3342542DDCF4743D12D9086544D78643FEDFE6D5C80666652F90CE073F293CBF179F6046A82C307EDD9B462E468A2F7D57A9BB8EFB173B08E1CB19AE98FB44334BECEA542C07CC5B8E977066262269CBD13E2B457EF60FDC6AA5560152569686C9FE8CA57387CF42E422D3AC2F5C4BD9CE1F2A3D1FD72269DCF4DCA3C200917A3C83B848CFDCFE97274BE7E1DBE65AF9FE0A466F2B218E6E65C50BBFDD2C8A19858AFFB401A0FC9B3942BC19B8869F796BA28AA09D98F64B35953970E25FB4705EC886D68861A2FECAA9AE64
  ```
第一行为加密前的参数
第二行为请求的接口路劲
第三行为加密后的params字段值

### 破解签名

上述说到中间还有障碍，指的是很多第三方apk往往会在so校验应用的签名，或者用应用的签名作为请求参数加密的一些参数，加大接口被调用的难度。当代码插桩添加log后，使用apktool回编，然后签名，你会发现应用打开就crash。因此必须要解决签名问题。

继续分析，从上一张图，可以看出，加密参数调用的是`NeteaseMusicUtils.serialdata`函数
  ```java

  // NeteaseMusicUtils.java
  static
  {
    UtilInterface.load("poison");
    nativeInit(NeteaseMusicApplication.a());
  }

  // UtilInterface.java
  public class UtilInterface
  {
    static
    {
      System.loadLibrary("neutil");
    }
    
    public static native void l(String paramString);
    
    public static void load(String paramString)
    {
      l(paramString);
    }
  }

  // so文件列表
  armeabi
  ├── libFMProcessor.so
  ├── libMP3Encoder.so
  ├── libbitmaps.so
  ├── libblur.so
  ├── libfpGenerate.so
  ├── libgifimage.so
  ├── libijkffmpeg.so
  ├── libijkplayer.so
  ├── libijksdl.so
  ├── libimagepipeline.so
  ├── liblocSDK3.so
  ├── libmemchunk.so
  ├── libndkbitmap.so
  ├── libneutil.so
  ├── libwakeup.so
  ├── libwebp.so
  └── libwebpimage.so
  ```

从代码看来，貌似加载了个`libneutil.so`，然后通过`libpoison.so`文件，只有`libneutil.so`文件，并没有`libpoison.so`这个文件。


通过使用ida调试，目前貌似只能在windows机器上面调试so文件，教程：[使用IDA Pro动态调试so文件](http://www.gimoo.net/t/1512/5677d68a4a1c5.html)

另外`libneutil.so`做了反调试处理，需要跳过反调试，教程：[在JNI_Onload 函数处下断点避开针对IDA Pro的反调试](http://www.gimoo.net/t/1512/5677cda2844d9.html)

在调试过程中，你会发现`libneutil.so`会使用`dlopen`和`dlsym`的方法加载了另外的`libpoison.so`。

![image](/images/android-decompile-share-1/android-ida-netease-neutil.png)

生成目录为：/data/data/com.netease.cloudmusic/.cache/libpoision${pid}.so

如果你要提取这个文件，需要是root手机权限。当然我手机root过，直接拷贝出来，后面用于加密参数。

以上还没有说到如何破解签名，因为破解签名是在libpoison.so里面，libneutil.so仅仅是加了反调试和解密`assets`目录下的`data.db`生成`libpoison.so`

如何破解`libpoison.so`里面的签名验证？

因为获取签名只有java方法`context.getPackageManager().getPackageInfo().signatures`，因此，肯定会从java层传递一个`context`对象到jni，jni又只能反射调用java方法，字符串的定义，在so也是一目了然，可以使用HexEdit工具修改的。因此，我们可以欺骗so，让他获取正确的应用签名，而不是二次打包应用后的签名。

由于libpoison.so还是用了`MessageDigest.getInstance("MD5").update(signature)`这个静态方法对应用签名进行md5校验，因此，直接修改`libpoison.so`里面的`MessageDigest`类引用为自己的java类`com.netease.cloudmusic.Hack`，在`com.netease.cloudmusic.Hack`里面给它正确结果即可，类名长度和MessageDigest长度一样，并保证被so反射调用到的方法都补全。


代码如下：
  ```
  package com.netease.cloudmusic;

  /**
   * Created by 4ndroidev on 16/7/31.
   * cheat the poison.so to load signature from here rather than apk
   */
  public class Hack {

    private static Hack instance = new Hack();

    private Hack() {
    }

    /**
     * how to get netease.signature?
     * <code>
     * PackageManager packageManager = getPackageManager();
     * PackageInfo info = packageManager.getPackageInfo("com.netease.cloudmusic", PackageManager.GET_SIGNATURES);
     * Signature[] signatures = info.signatures;
     * FileOutStream os = new FileOutputStream(new File('netease.signature'));
     * os.write(signatures[0].toByteArray());
     * os.close();
     * </code>
     */
    public static Hack getInstance(String useless) {
      return instance;
    }

    //just for native call
    public void update(byte[] bytes) {
    }

    /**
     * directly return the result of md5(signature); actually you should follow the steps
     * 1. covert assets/netease.signature to byte array variable `signature`
     * 2.
     * <code>
     * MessageDigest md = MessageDigest.getInstance("MD5").update(signature)
     * md.update(signature);
     * byte[] digest = md.digest();
     * </code>
     */
    public byte[] digest() {
      byte[] digest = {-38, 107, 6, -99, -95, -30, -104, 45, -77, -29, -122, 35, 63, 104, -41, 109};
      return digest; // hardcode: digest = md5(signature);
    }

  }
  ```

![image](/images/android-decompile-share-1/android-hexedit-netease-poison.png)

也可以用另外的破解签名方法，可参照另外一篇分享：[Android Apk 签名破解](http://androidev.coding.me/2017/04/18/android-signatures-crack)

### 回编译

由于我们获取了`libpoison.so`，以及破解了`libpoison.so`的签名验证，即可使用`System.loadLibrary("poison");`代替`UtilInterface.load("poison");`，避免回编重签导致奔溃问题。

回编译，由于插桩log，我们便得一个个请求，同理，可以找出返回结果，添加log。

### 静态分析

虽然打出了每个请求参数的log，事实还有很多工作，例如参数的来源，这些需要看混淆代码获得，至于怎么快速定位，那就是通过完整的类名+方法名，或者类名+属性名进行搜索。

只要熟悉smali，是可以直接将smali逆回java，体力活，只要耐心即可，示例：
  ```smali

  // smali

  .method private static a(Ljava/lang/String;Ljava/lang/String;Ljava/util/Map;Ljava/lang/String;)Ljava/util/Map;
      .locals 7
      .annotation system Ldalvik/annotation/Signature;
          value = {
              "(",
              "Ljava/lang/String;",
              "Ljava/lang/String;",
              "Ljava/util/Map",
              "<",
              "Ljava/lang/String;",
              "Ljava/lang/String;",
              ">;",
              "Ljava/lang/String;",
              ")",
              "Ljava/util/Map",
              "<",
              "Ljava/lang/String;",
              "Ljava/lang/String;",
              ">;"
          }
      .end annotation

      .prologue
      const/4 v0, 0x0

      if-nez p2, :cond_0

      new-instance v1, Ljava/util/HashMap;

      invoke-direct {v1}, Ljava/util/HashMap;-><init>()V

      :goto_0
      if-eqz p0, :cond_1

      :try_start_0
      const-string v2, "Yw=="

      invoke-static/range {v2 .. v2}, La/auu/a;->c(Ljava/lang/String;)Ljava/lang/String;

      move-result-object v2

      invoke-virtual {p0, v2}, Ljava/lang/String;->split(Ljava/lang/String;)[Ljava/lang/String;

      move-result-object v2

      array-length v3, v2

      :goto_1
      if-ge v0, v3, :cond_1

      aget-object v4, v2, v0

      const-string v5, "eA=="

      invoke-static/range {v5 .. v5}, La/auu/a;->c(Ljava/lang/String;)Ljava/lang/String;

      move-result-object v5

      invoke-virtual {v4, v5}, Ljava/lang/String;->split(Ljava/lang/String;)[Ljava/lang/String;

      move-result-object v4

      const/4 v5, 0x0

      aget-object v5, v4, v5

      const/4 v6, 0x1

      aget-object v4, v4, v6

      invoke-interface {v1, v5, v4}, Ljava/util/Map;->put(Ljava/lang/Object;Ljava/lang/Object;)Ljava/lang/Object;
      :try_end_0
      .catch Lorg/json/JSONException; {:try_start_0 .. :try_end_0} :catch_0

      add-int/lit8 v0, v0, 0x1

      goto :goto_1

      :cond_0
      new-instance v1, Ljava/util/HashMap;

      invoke-direct {v1, p2}, Ljava/util/HashMap;-><init>(Ljava/util/Map;)V

      goto :goto_0

      :cond_1
      :try_start_1
      new-instance v2, Lorg/json/JSONObject;

      invoke-direct {v2}, Lorg/json/JSONObject;-><init>()V

      invoke-interface {v1}, Ljava/util/Map;->keySet()Ljava/util/Set;

      move-result-object v0

      invoke-interface {v0}, Ljava/util/Set;->iterator()Ljava/util/Iterator;

      move-result-object v3

      :goto_2
      invoke-interface {v3}, Ljava/util/Iterator;->hasNext()Z

      move-result v0

      if-eqz v0, :cond_2

      invoke-interface {v3}, Ljava/util/Iterator;->next()Ljava/lang/Object;

      move-result-object v0

      check-cast v0, Ljava/lang/String;

      invoke-interface {v1, v0}, Ljava/util/Map;->get(Ljava/lang/Object;)Ljava/lang/Object;

      move-result-object v4

      invoke-virtual {v2, v0, v4}, Lorg/json/JSONObject;->put(Ljava/lang/String;Ljava/lang/Object;)Lorg/json/JSONObject;
      :try_end_1
      .catch Lorg/json/JSONException; {:try_start_1 .. :try_end_1} :catch_0

      goto :goto_2

      :catch_0
      move-exception v0

      invoke-virtual {v0}, Lorg/json/JSONException;->printStackTrace()V

      :goto_3
      return-object v1

      :cond_2
      :try_start_2
      new-instance v3, Lorg/json/JSONObject;

      invoke-direct {v3}, Lorg/json/JSONObject;-><init>()V

      invoke-static {}, Lcom/netease/cloudmusic/i/f;->f()Ljava/util/List;

      move-result-object v0

      invoke-interface {v0}, Ljava/util/List;->iterator()Ljava/util/Iterator;

      move-result-object v4

      :goto_4
      invoke-interface {v4}, Ljava/util/Iterator;->hasNext()Z

      move-result v0

      if-eqz v0, :cond_3

      invoke-interface {v4}, Ljava/util/Iterator;->next()Ljava/lang/Object;

      move-result-object v0

      check-cast v0, Lorg/apache/http/cookie/Cookie;

      invoke-interface {v0}, Lorg/apache/http/cookie/Cookie;->getName()Ljava/lang/String;

      move-result-object v5

      invoke-interface {v0}, Lorg/apache/http/cookie/Cookie;->getValue()Ljava/lang/String;

      move-result-object v0

      invoke-virtual {v3, v5, v0}, Lorg/json/JSONObject;->put(Ljava/lang/String;Ljava/lang/Object;)Lorg/json/JSONObject;

      goto :goto_4

      :cond_3
      const-string v0, "NwsSBxwDAAwK"

      invoke-static/range {v0 .. v0}, La/auu/a;->c(Ljava/lang/String;)Ljava/lang/String;

      move-result-object v0

      invoke-virtual {v3, v0, p3}, Lorg/json/JSONObject;->put(Ljava/lang/String;Ljava/lang/Object;)Lorg/json/JSONObject;

      const-string v0, "LQsCFhwC"

      invoke-static/range {v0 .. v0}, La/auu/a;->c(Ljava/lang/String;)Ljava/lang/String;

      move-result-object v0

      invoke-virtual {v2, v0, v3}, Lorg/json/JSONObject;->put(Ljava/lang/String;Ljava/lang/Object;)Lorg/json/JSONObject;

      invoke-interface {v1}, Ljava/util/Map;->clear()V

      const-string v0, "NQ8RExQD"

      invoke-static/range {v0 .. v0}, La/auu/a;->c(Ljava/lang/String;)Ljava/lang/String;

      move-result-object v0

      invoke-virtual {v2}, Lorg/json/JSONObject;->toString()Ljava/lang/String;

      move-result-object v2

      invoke-static {p1, v2}, Lcom/netease/cloudmusic/utils/NeteaseMusicUtils;->serialdata(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;

      move-result-object v2

      invoke-interface {v1, v0, v2}, Ljava/util/Map;->put(Ljava/lang/Object;Ljava/lang/Object;)Ljava/lang/Object;
      :try_end_2
      .catch Lorg/json/JSONException; {:try_start_2 .. :try_end_2} :catch_0

      goto :goto_3
  .end method
  ```

  ```java

  //java

  private static Map<String, String> a(String query, String path, Map<String, String> params, String requestId){
    Map<String, Object> body;
    
    if (params == null) {
        body = new HashMap<>();
    } else {
        body = new HashMap<>(params);
    }
    try {
      if (query != null) {
        String[] kvs = query.split("&");
        for (int i = 0, len = kvs.length; i < len; i++) {
          String[] kv = kvs[i].split("=");
          body.put(kv[0], kv[1]);
        }
      }
      JSONObject paramObject = new JSONObject();
      Iterator<String> paramIterator = body.keySet().iterator();
      while (paramIterator.hasNext()) {
        String key = paramIterator.next();
        paramObject.put(key, body.get(key));
      }
      Iterator<Cookie> cookieIterator = f().iterator();
      JSONObject header = new JSONObject();
      while (cookieIterator.hasNext()) {
        Cookie cookie = cookieIterator.next();
        header.put(cookie.getName(), cookie.getValue());
      }
      header.put("requestId", requestId);
      paramObject.put("header", header);
      body.clear();
      body.put("params", NeteaseMusicUtils.serialdata(path, paramObject.toString()));
    } catch (JSONException e) {
    }
    return body;
  }
  ```

### 编写程序

当你获取了网易云音乐的接口后，肯定是开始练手，编写程序业务。我自己搞了一下，效果图如下：

![image](/images/android-decompile-share-1/android-netease-screenshot.jpg)

