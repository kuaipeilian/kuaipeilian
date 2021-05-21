---
title: Jenkins实现APP自动打包和上传
date: 2021-05-21 20:00:52
author: yushuailiu
tags:
    - 移动端
    - 自动化
---
### 项目背景
一个基于Jenkins 进行开发的自动构建系统，使用专门的机器负责所有App的构建任务，目的是解决简化打包流程。
目前公司所有APP已接入。系统根据不同项目主要分为两种类型的任务，一种是 iOS/Android 内测分发的构建任务，另一种则是iOS构建好后直接上传到 TestFlight/AppStore 的构建任务。
### 整体流程
<div style="width:400px;height: 400px;margin: auto;"> {% img  /images/dabao.png 400px 400px '"fff text" "alt text"' %}</div>

### iOS内测分发构建任务流程
选择想要构建的任务 -> 选择需要构建的分支 -> 点击开始构建 -> 系统自动拉取项目代码 -> 自动下载相关依赖项目后进行整体项目的集成 -> 编译构建打包 -> 将生成的ipa包上传到OSS服务器存档 -> 上传到蒲公英平台 -> 根据生成的ipa包相关基本信息、下载安装地址与途径以钉钉形式发送

### AppStore上传构建任务流程:
选择想要构建的任务 -> 选择需要构建的分支 -> 点击开始构建 -> 系统自动拉取项目代码 -> 自动下载相关依赖项目后进行整体项目的集成 -> 编译构建打包 -> 将生成的ipa包上传到OSS服务器存档 ->上传到TestFlight/AppStore -> 上传到蒲公英平台 -> 根据生成的ipa包相关基本信息、下载安装地址与途径以钉钉形式发送

### Android内测分发构建任务流程：
选择想要构建的任务 -> 选择需要构建的分支 -> 点击开始构建 -> 系统自动拉取项目代码 -> 自动下载相关依赖项目后进行整体项目的集成 -> 编译构建打包 -> 将生成的apk包加渠道 -> 上传阿里云进行加固 -> 加固包重新签名 -> 上传到OSS服务器存档 -> 上传到蒲公英平台 -> 根据生成的apk包相关基本信息、下载安装地址与途径以钉钉形式发送

### 安装jenkins
先安装JDK，然后下载最新的war包，打开终端，进入到war包所在目录，执行命令：
```
java -jar jenkins.war --httpPort=8080
```
启动之后，浏览器输入地址：http://localhost:8080

安装Jenkins相关插件
```
Git插件
Git 参数化构建插件（Git Parameter）
Gradle 插件 (Gradle plugin) - Android 专用
Xcode 插件 (Xcode integration) - iOS 专用
Jenkins 输出日志文字颜色插件（AnsiColor）
```
### iOS Job自动构建设置
{% img  /images/jenkins-2.png 250 250 '"fff text" "alt text"' %}

### 编译打包之前干了什么
#### 修改版本号以及build号
PlistBuddy 是 Mac 系统中一个用于命令行下读写 plist 文件的工具。可以用来读取或修改 plist 文件的内容
```
/usr/libexec/PlistBuddy -c "Set :CFBundleShortVersionString $APP_VERSION" "${project_infoplist_path}"
bundleShortVersion=$(/usr/libexec/PlistBuddy -c "print CFBundleShortVersionString" "${project_infoplist_path}")
```
build号（一个版本对应一个json文件，内容为

{"BuildNumber": 99, "BuildVersion": "2.0.0"} 每次编译打包前递增build号，然后修改
Info.plist的build号
）
```
/usr/libexec/PlistBuddy -c "Set :CFBundleVersion $BuildNumber" "${project_infoplist_path}"
bundleVersion=$(/usr/libexec/PlistBuddy -c "print CFBundleVersion" "${project_infoplist_path}")
```
#### 编译打包
清理
```
xcodebuild -workspace "${WORK_SPACE}" -scheme "${TARGET_NAME}" -configuration 'Release' clean
```
编译
```
xcodebuild archive -workspace "${WORK_SPACE}" -sdk iphoneos -scheme "${TARGET_NAME}" -archivePath "./build/${XCARCHIVE}" -configuration 'Release' -arch "arm64"（
-archivePath：设置项目的归档路径）
```
导出ipa
```
xcodebuild -exportArchive -archivePath "./build/${XCARCHIVE}" -exportPath "./build/${IPAPATH}" -target "${TARGET_NAME}" -exportOptionsPlist "${EXPORTOPTIONSPLIST}" -allowProvisioningUpdates
-exportArchive：导出ipa
-exportPath：导出ipa文件的路径
-exportOptionsPlist：文件导出时的配置信息
-allowProvisioningUpdates：允许xcodebuild与苹果网站通讯，进行自动签名，证书自动更新，生成
```
#### 上传appstore
Xcode 11 之后已不再提供 Application Loader，改用 xcrun altool 上传 ipa

altool 可以 --validate-app 验证和 --upload-app 上传

验证：
```
/usr/bin/xcrun altool --validate-app -f ${IPANAME} -t ios --apiKey C7RL3QD397 --apiIssuer 69a6de97-9460-47e3-e053-5b8c7c11a4d1 --verbose
apiKey 和 apiIssuer 需要登录开发者网站，打开 用户和访问->密钥->然后新增密钥
成功会提示：No errors validating archive at ... 
```
上传：
```
/usr/bin/xcrun altool --upload-app -f ${IPANAME} -t ios --apiKey C7RL3QD397 --apiIssuer 69a6de97-9460-47e3-e053-5b8c7c11a4d1 --verbose
```
如下图所示，表示上传成功
<div style="width:600px;height: 50px;margin: auto;"> {% img  /images/jenkins-333.png 400px 400px '"fff text" "alt text"' %}</div>

小提示：
每次 TestFlight 发布测试都会询问你的 app 有没有使用加密技术，勾选完后才能测试。这时可以在 Info.plist 添加 ITSAppUsesNonExemptEncryption 为 false 就可以了
<key>ITSAppUsesNonExemptEncryption</key><false/>  下次打包上传，接收到 App Store Connect 邮件 has completed processing 就能自动发布测试了

#### 发送钉钉提醒
{% img  /images/jenkins-444.jpg 400px 400px '"fff text" "alt text"' %}

#### Android Job自动构建设置
 {% img  /images/jenkins-555.png 280 500 '"fff text" "alt text"' %}

#### 渠道包
解压之后，META-INF文件夹下写入文件kpl_channel_{channel}
#### 签名
zipalign 是一种归档对齐工具，可对 Android 应用 (APK) 文件提供重要的优化。其目的是要确保所有未压缩数据的开头均相对于文件开头部分执行特定的对齐。具体来说，它会使 APK 中的所有未压缩数据（例如图片或原始文件）在 4 字节边界上对齐。这样做的好处是可以减少运行应用时消耗的 RAM 容量。
按着有利于系统处理的排列方式，对我们apk中的资源文件进行排列，提高资源的查找速度，从而去提高应用的运行效率
```
zipalign 4 {old_apk_path} {new_apk_path} 对齐
zipalign -c -v 4 {new_apk_path} 检测是否对齐成功
apksigner sign --ks {jks_path} --ks-key-alias {ks_key_alias} --ks-pass pass:{ks_pass} --key-pass pass:{key_pass} --v1-signing-enabled true --v2-signing-enabled true {new_apk_path} 签名
--ks //签名证书路径
--ks-key-alias //生成jks时指定的alias
--ks-pass //密码
--key-pass //签署者的密码，即生成jks时指定alias对应的密码
```
