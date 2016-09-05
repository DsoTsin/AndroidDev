# Android 7 Nougat 新特性调研：DayDream VR

> DayDreamVR 是Android 7引进来的新特性，这个版本提供了App的VR模式，同时定义了VR头盔以及控制器的一些标准，可以让手机制造商提供兼容该平台的VR设备。

## DayDream VR设置

![daydream](images/an_daydream.png)

当前Android VR的开发目前需要**Nexus 6P**来支持，目前市面上暂时没有手机能满足`DayDreamVR`的要求。

![dev-kit](images/an_daydream_devkit.png)

不过我们可以使用**另外一台手机**作为**DayDreamVR的手柄**，在Nexus 6P需要打开DayDreamVR的**开发者选项**通过蓝牙和手柄配对即可使用。

![paint](images/an_daydream_paint.png)

为了提高渲染的性能，Android N为App引入了VR模式，在该模式下，图像渲染变得更加直接，延迟也更低。

![option](images/an_daydream_option.png)

---

## 开发入手

目前，我们可以使用Google VR SDK来进行Android上的VR APP开发。



---

## 参考

1. [Designing & Developing for the Daydream Controller - Google I/O 2016][1]
2. [Set up a Daydream Development Kit][2]


[1]:https://www.youtube.com/watch?v=l9OfmWnqR0M
[2]:https://developers.google.com/vr/concepts/dev-kit-setup