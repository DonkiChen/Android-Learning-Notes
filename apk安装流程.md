# apk安装流程
___

## 打包流程

![apk打包流程](https://user-gold-cdn.xitu.io/2019/5/6/16a8c918da070cc0?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

### 编译阶段

1. aapt工具编译项目中的资源文件, 生成R.java, resource.arsc. xml文件编译成二进制, assets 和 raw 目录下的资源并不会被编译，会被原封不动的打包到 apk 压缩包中.
    * R.java: 定义了各个资源ID常量, 十六进制表示的int值, 前8位代表packageId, 01 = system, 7f = app 之后的8位代表资源类型, 最后16位代表资源id
    * resource.arsc: 文件索引表, 在给定资源ID和设备信息下能快速找到文件
2. 处理aidl文件, aidl工具解析文件生成相应的Java代码
3. 源代码经过编译器生成.class文件, 连同第三方库中的.class文件一起用dx工具生成dex文件, 如果有multidex, 这里也会有多个dex文件. dx工具将Java字节码转换成Dalvik字节码, 压缩常量池, 消除冗余信息

### 打包阶段

1. 通过apkbuilder将编译过后的资源文件, dex文件, assets文件等一起打包生成apk
2. 通过jarsigner对apk进行签名, 签名之后会生成 MATE-INF 文件夹, MATE-INF 文件夹保存着和签名相关的各个文件, 如下:
    * CERT.SF：生成每个文件相对的密钥
    * MANIFEST.MF：数字签名信息
    * xxx.SF：这是 JAR 文件的签名文件
    * xxx.DSA：对输出文件的签名和公钥
3. 通过zipalign对apk中的资源文件进行对齐操作, 加快资源的访问速度

## 安装流程 PackageManagerService(PMS)

1. 创建存储apk的路径, 即 /data/app/packageName
2. 将apk拷贝到目标路径, 改名为 base.apk
3. 将apk中的动态链接库 .so文件 拷贝到 目录下
4. 解析apk文件, 解析清单文件
5. 验证签名
6. dex优化, dex2ota, 将dex文件转换为oat文件
7. 继续扫描解析apk, 保存信息到PMS, 生成 data文件夹, /data/data/packageName
8. 更新系统设置中的应用信息
9. 发送成功广播