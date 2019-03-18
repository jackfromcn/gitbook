# Mac关闭摄像头



为了防止黑客入侵，我们可以手动执行命令，关掉摄像头设备的访问权限，让黑客去看黑夜吧。

Mac关闭禁用摄像头的方法，打开终端，拷贝如下代码，回车，输入管理员密码，再次拷贝，回车。

```bash
sudo chmod a-r /System/Library/Frameworks/CoreMediaIO.framework/Versions/A/Resources/VDC.plugin/Contents/MacOS/VDC

sudo chmod a-r /System/Library/PrivateFrameworks/CoreMediaIOServicesPrivate.framework/Versions/A/Resources/AVC.plugin/Contents/MacOS/AVC

sudo chmod a-r /System/Library/QuickTime/QuickTimeUSBVDCDigitizer.component/Contents/MacOS/QuickTimeUSBVDCDigitizer

sudo chmod a-r /Library/CoreMediaIO/Plug-Ins/DAL/AppleCamera.plugin/Contents/MacOS/AppleCamera

sudo chmod a-r /Library/CoreMediaIO/Plug-Ins/FCP-DAL/AppleCamera.plugin/Contents/MacOS/AppleCamera
```

打开摄像头方法

```bash
sudo chmod a+r /System/Library/Frameworks/CoreMediaIO.framework/Versions/A/Resources/VDC.plugin/Contents/MacOS/VDC

sudo chmod a+r /System/Library/PrivateFrameworks/CoreMediaIOServicesPrivate.framework/Versions/A/Resources/AVC.plugin/Contents/MacOS/AVC

sudo chmod a+r /System/Library/QuickTime/QuickTimeUSBVDCDigitizer.component/Contents/MacOS/QuickTimeUSBVDCDigitizer

sudo chmod a+r /Library/CoreMediaIO/Plug-Ins/DAL/AppleCamera.plugin/Contents/MacOS/AppleCamera

sudo chmod a+r /Library/CoreMediaIO/Plug-Ins/FCP-DAL/AppleCamera.plugin/Contents/MacOS/AppleCamera
```

检验方法，使用Mac自带应用 Photo Booth 或者QQ视频来校验是否生效。