# Google V8编译指南（Android）

## 准备

> V8源码仓库：[https://github.com/TsinStudio/v8-5.5.1](https://github.com/TsinStudio/v8-5.5.1)

|编译环境| |
|:----:|:----:|
| 操作系统 | MacOS |
| 编译工具 |XCode + XCodeCommandTools + NDK r10+|
| python | 2.7 |
| 依赖python第三方库 |urllib3, certifi|

下载V8第三方库：

```python
python download_deps.py
```

安装Clang编译器（可选）：

```python
python tools/clang/scripts/update.py --if-needed
```
---

## 使用GYP构建V8

直接在根目录执行：

```shell
make android_arm.release -j 16 android_ndk_root=... 
```

生成的V8共享库位于`out/android_arm.release/lib.target`下。