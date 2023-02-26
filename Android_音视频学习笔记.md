# 一、发展方向

1、计算机相关专业本科及以上学历，具有扎实的计算机理论基础，熟悉常⽤数据结构算法。

2、熟悉 iOS/Android ⾳视频采集录制编辑相关框架。

3、熟悉 OpenGL/OpenGLES/DirectX/Metal/Vulkan 其中⼀种图形库开发，较好地掌握图形渲染基础算法和核⼼渲染技术。

4、熟悉 MP4、FLV 等视频格式；熟悉 RTMP、RTSP、HTTP-FLV、HLS、Mpeg-Dash 等协议；熟悉 H.264、HEVC、AV1、 AAC 等⾳视频编码格式。熟悉端侧的软硬解⽅案和实现

5、有短视频编辑模块或直播推流模块的开发经验，有视频剪辑和特效的开发经验。有视频质量客观和主观质量量化和评估优化经验

6、熟悉多线程编程，熟悉多线程的调试分析⽅法。



# 二、Lifecycle与协程、Flow



# 三、targetSdkVersion、compileSdkVersion、minSdkVersion作用与区别

## 1.minSdkVersion

指定app运行的最低设备sdk版本，如minSdkVersion=19 表示该app最低支持Android 4.4（API 19）设备，低于此版本的设备将不能使用该app。



## 2.compileSdkVersion

和编译时有关。比如我们当前compileSdkVersion=28（Andorid 9.0），Android 10 新增了有关5G的api。我们的app想尽早使用5G相关的api，这时只需要将compileSdkVersion=29（Android 10）,就能在编译阶段编译通过。



## 3.targetSdkVersion

直观翻译是“目标版本”，它的作用是兼容旧的api。

举个例子，Android 8.0/8.1 系统规定了透明的activity 不能固定方向，否则抛出异常。假设用了透明的activity，且固定了方向，在Android 8.0版本以下运行正常，当我们运行在Android 8.0/8.1系统上，结果如何会如何呢？很不幸，直接crash。对于我们来说，这结果当然不能接受了。

![img](https://upload-images.jianshu.io/upload_images/19073098-d3593349aad40400.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

从上图可以看出：getApplicationInfo().targetSdkVersion取的值即是在build.gradle里设置的targetSdkVersion值，这里判断targetSdkVersion是否大于Android 8.0（O），如果成立才执行后续逻辑（透明且固定方向），进而抛出异常。在此我们知道，想让这段逻辑不成立，那么我们将targetSdkVersion设置为26（Android 8.0 对应26）以下，即使我们app运行在Andorid 8.0以上的设备，都不会出现问题。这样，我们的app不更新，但是新的Android SDK 里Activity onCreate方法变更了部分逻辑行为，通过targetSdkVersion限制，我们的app依然能够以旧的逻辑运行新的设备上，达到了兼容老版本api的目的。



总结：

**对于同一个API(方法），新版本的系统更改了它的内部实现逻辑，新逻辑可能会影响之前调用此API的App，为了兼容此问题，引入targetSdkVersion。当targetSdkVersion>=xx(某个系统版本）时再生效其新修改的逻辑，否则还是沿用之前的逻辑。换句话说：如果你更改了targetSdkVersion值，说明你已经测试过兼容性问题了。**



``

```java
    /**
     * The minimum SDK version this application targets.  It may run on earlier
     * versions, but it knows how to work with any new behavior added at this
     * version.  Will be {@link android.os.Build.VERSION_CODES#CUR_DEVELOPMENT}
     * if this is a development build and the app is targeting that.  You should
     * compare that this number is >= the SDK version number at which your
     * behavior was introduced.
     */
    public int targetSdkVersion;
```

更新targetSdkVersion时，比如从26（Android 8.0）变更到29（Android 9.0），意味着我们对26～29之间的系统兼容性进行了充分的测试，因此每当我们变更targerSdkVersion时，要充分测试其系统兼容性。

也许你会说，那我可以不更新targetSdkVersion值嘛，一劳永逸，理论上没啥问题。但是，如果你想在新的系统上使用api新的行为，那么就需要更新targetSdkVersion。再者，google会对targetSdkVersion进行一些强关联，比如app上传到google play，必须要求targetSdkVersion>=26（具体的值随着时间的推移，要求不一样）。

![img](https://upload-images.jianshu.io/upload_images/19073098-63025a6b56456d4c.png?imageMogr2/auto-orient/strip|imageView2/2/w/1142/format/webp)

image.png

到这，也许你还会说，为啥Andorid 6.0（包含）以上动态权限不根据targetSdkVersion来适配呢？我们说的targetSdkVersion是向前兼容，兼容之前的app使用的同一api旧的行为逻辑，而动态权限是google在Android 6.0（包含）以上强制要求的，人家就不想给你转圜的余地，有啥办法呢？还是乖乖判断当前系统是否是Android 6.0及其以上的，再决定是否动态申请权限了。。



