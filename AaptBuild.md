# 在WSL2下构建Android ARM AAPT

---

- 下载源码（清华源）
        
        curl https://mirrors.tuna.tsinghua.edu.cn/git/git-repo -o repo
        chmod +x repo
        echo "export REPO_URL='https://mirrors.tuna.tsinghua.edu.cn/git/git-repo'" >> ~/.bashrc
        source ~/.bashrc
        repo init -u https://mirrors.tuna.tsinghua.edu.cn/git/AOSP/platform/manifest -b android-10.0.0_r36 # 下载android10源码
        repo sync -c
        source build/envsetup.sh
        lunch sdk-eng # 设置成sdk构建

- 修改**Aapt**构建文件`Android.bp`
    - 修改`androidfw`的构建文件，让它支持`静态库`构建， `frameworks\base\libs\androidfw\Android.bp`，target.android.static设置成true
    - 修改Aapt构建文件`frameworks\base\tools\aapt\Android.bp`，增加：
        ```gn
        cc_library_shared {
            name: "libaapt_shared",
            
            // 系统不允许链接或者缺失函数符号的库改成静态链接
            static_libs: [
                "libandroidfw",
                "libexpat",
                "libziparchive",
                "libbase",
                "libpng",
            ],

            // 大多数系统自带动态库
            shared_libs: [
                "libz",
                "liblog",
                "libcutils",
                "libutils",
            ],
            cflags: [
                "-Wno-format-y2k",
                "-Wno-error=implicit-fallthrough",
            ],

            srcs: [
                "AaptAssets.cpp",
                "AaptConfig.cpp",
                "AaptUtil.cpp",
                "AaptXml.cpp",
                "ApkBuilder.cpp",
                "Command.cpp",
                "CrunchCache.cpp",
                "FileFinder.cpp",
                "Images.cpp",
                "Package.cpp",
                "pseudolocalize.cpp",
                "Resource.cpp",
                "ResourceFilter.cpp",
                "ResourceIdCache.cpp",
                "ResourceTable.cpp",
                "SourcePos.cpp",
                "StringPool.cpp",
                "WorkQueue.cpp",
                "XMLNode.cpp",
                "ZipEntry.cpp",
                "ZipFile.cpp",
            ],
        }

        cc_binary {
            name: "aapt_andr",

            srcs: ["Main.cpp"],
            use_version_lib: true,
            
            static_libs: [
                "libandroidfw",
                "libexpat",
                "libziparchive",
                "libbase",
                "libpng",
            ],

            shared_libs: [
                "libaapt_shared",
                "libz",
                "liblog",
                "libcutils",
                "libutils",
            ],
        }
        ```
- 构建 `make aapt_andr`
- 构建其它CPU架构的可执行文件