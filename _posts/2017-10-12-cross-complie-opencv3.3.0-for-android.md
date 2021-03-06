---
layout:     post
title:      "在Ubuntu16.04上编译OpenCV4Android"
date:       2017-10-12
author:     "winson"
tags:
    - Opencv
    - Android
---

# 在Ubuntu16.04上编译OpenCV4Android

## 0x00 准备源码
直接到github上下载最新版源码

``` shell
git clone https://github.com/opencv/opencv.git
```

## 0x01 编译环境准备

如果木有cmake，就先安装cmake

``` shell
$ sudo apt-get install cmake
```

但是有时候发现这样安装的不是最新版的cmake，opencv3.3的编译貌似需要的cmake版本比ubuntu官方仓库的要新。

> 现在我在16.04上用cmake3.5就没问题~可能之前的仓库cmake比较旧吧。。。

直接cmake官网下载最新版的cmake然后configure然后make & make install，或者用Github上的镜像也行：

```
$ git clone https://github.com/Kitware/CMake.git
$ git checkout -b <new-branch-name> <release-tag>
$ ./configure
$ make
$ sudo apt-get remove cmake
$ sudo make install
```

这样装完后输入`cmake --version`看版本可能会出现cmake找不到的问题，一般为缺少到/usr/bin/cmake的软连接的问题，创建一个就好了:

``` shell
$ ln -s /usr/local/bin/cmake /usr/bin/cmake
```

这样就木有问题了。

安装ant

``` shell
$ sudo apt-get install ant
```

autotools系列工具，这应该大家都有的。

环境变量：ANDROID_NDK， ANDROID_SDK， JAVA_HOME

## 0x02 编译

``` shell
$ cd cd <your-path-to-opencv-source>/platforms/script
$ ./cmake_android_arm.sh
```

这样输出一大段打印，说明你的opencv的配置，其中关于Android的一段:

``` shell
Android:
--     Android ABI:                 armeabi-v7a
--     STL type:                    gnustl_static
--     Native API level:            android-9
--     SDK target:                  android_sdk_target_status-NOTFOUND
--     Android NDK:                 /home/winson/Android/Ndk/android-ndk-r14b (toolchain: arm-linux-androideabi-4.9)
--     android tool:                /home/winson/Android/Sdk/tools/android (Android SDK Tools)
```

你会发现SDK Target那一栏有问题（没错我眼就是这么尖O(∩_∩)O），其原因是opencv编译脚本在获取sdk的时候对新版本的Android SDK的BUG，我们要修改一个文件：[path-to-opencv-source]/cmake/OpenCVDetectAndroidSDK.cmake

```
#get installed targets
  if(ANDROID_TOOLS_Pkg_Revision GREATER 11)
    execute_process(COMMAND ${ANDROID_EXECUTABLE} list target -c
      RESULT_VARIABLE ANDROID_PROCESS
      OUTPUT_VARIABLE ANDROID_SDK_TARGETS
      ERROR_VARIABLE ANDROID_PROCESS_ERRORS
      OUTPUT_STRIP_TRAILING_WHITESPACE
      )
    string(REGEX MATCHALL "[^\n]+" ANDROID_SDK_TARGETS "${ANDROID_SDK_TARGETS}")
  else()
    ...
  endif()
```

找到上面那一段，然后把其中的:

`string(REGEX MATCHALL "[^\n]+" ANDROID_SDK_TARGETS "${ANDROID_SDK_TARGETS}")`

改成:

`string(REGEX MATCHALL "android-[0-9]+" ANDROID_SDK_TARGETS "${ANDROID_SDK_TARGETS}")`

改好了后删掉build_android_arm这个文件夹然后重新来一次`$ ./cmake_android_arm.sh`

这次没问题了。

在build_android_arm这个文件夹下执行`make -j8`即可。

这样默认编出来的是静态库armv7a的版本。

> 貌似opencv源码编译不出动态库版本？我只知道在opencv的cmake脚本里面遇到Android直接禁用动态库编译了

## 0x03 使用opencv_contrib
代码下载:

``` shell
git clone https://github.com/opencv/opencv_contrib.git
```

然后按照上面的步骤正常编译，发现会有错误，大概是说有文件没找到
```
opencv_contrib/modules/xfeatures2d/src/boostdesc.cpp:646:37: fatal error: boostdesc_bgm.i: No such file or directory
```
Google一番发现原来是因为opencv的cmake编译脚本执行过程中会去用curl下载点东西，下载记录在build根目录(即构建目录)的CMakeDownloadLog.txt文件里。打开这个文件会发现用curl去下载一些文件的时候失败了。又Google一番发现是cmake用的curl没有ssl支持访问不了https。于是准备重新安装curl。

> 如果你系统里面的curl版本大于7.40则其已支持ssl，可以用命令验证下。

首先卸载系统的curl

``` shell
sudo apt-get remove curl
```

github下载curl源码

``` shell
https://github.com/curl/curl.git
```

然后开始编译curl

``` shell
./buildconf
./configure --with-ssl
make
sudo make install
```

这里有个坑，这样直接弄的话会出现curl不能用的情况，原因是curl跟其所使用的libcurl版本不匹配导致。可以用如下命令证实：

``` shell
curl --version
```

这样会输出curl以及使用的libcurl使用的版本号，类似于这样(这个是我找网上的，因为我忘记截图了...)：

``` shell
$ curl --version
curl 7.41.0 (x86_64-unknown-linux-gnu) libcurl/7.22.0 OpenSSL/1.0.1 zlib/1.2.3.4 libidn/1.23 librtmp/2.3
```

然后我们看看我们在用哪个curl:

``` shell
$ which curl
/usr/local/bin/curl
```

然后看看这个`curl`链接的是哪个`libcurl.so`:

``` shell
winson@ubuntu-server:~/projects/git_projects/curl$ ldd /usr/local/bin/curl
        linux-vdso.so.1 =>  (0x00007fffb97a8000)
        libcurl.so.4 => /usr/lib/x86_64-linux-gnu/libcurl.so.4 (0x00007f6e2f71f000)
        libz.so.1 => /lib/x86_64-linux-gnu/libz.so.1 (0x00007f6e2f505000)
        libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007f6e2f2e7000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f6e2ef1d000)
        libidn.so.11 => /usr/lib/x86_64-linux-gnu/libidn.so.11 (0x00007f6e2ecea000)
        librtmp.so.1 => /usr/lib/x86_64-linux-gnu/librtmp.so.1 (0x00007f6e2eacd000)
        libssl.so.1.0.0 => /lib/x86_64-linux-gnu/libssl.so.1.0.0 (0x00007f6e2e864000)
        libcrypto.so.1.0.0 => /lib/x86_64-linux-gnu/libcrypto.so.1.0.0 (0x00007f6e2e420000)
        libgssapi_krb5.so.2 => /usr/lib/x86_64-linux-gnu/libgssapi_krb5.so.2 (0x00007f6e2e1d5000)
        liblber-2.4.so.2 => /usr/lib/x86_64-linux-gnu/liblber-2.4.so.2 (0x00007f6e2dfc6000)
        libldap_r-2.4.so.2 => /usr/lib/x86_64-linux-gnu/libldap_r-2.4.so.2 (0x00007f6e2dd75000)
        /lib64/ld-linux-x86-64.so.2 (0x00005557bc047000)
        libgnutls.so.30 => /usr/lib/x86_64-linux-gnu/libgnutls.so.30 (0x00007f6e2da44000)
        libhogweed.so.4 => /usr/lib/x86_64-linux-gnu/libhogweed.so.4 (0x00007f6e2d811000)
        libnettle.so.6 => /usr/lib/x86_64-linux-gnu/libnettle.so.6 (0x00007f6e2d5db000)
        libgmp.so.10 => /usr/lib/x86_64-linux-gnu/libgmp.so.10 (0x00007f6e2d35a000)
        libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007f6e2d156000)
        libkrb5.so.3 => /usr/lib/x86_64-linux-gnu/libkrb5.so.3 (0x00007f6e2ce84000)
        libk5crypto.so.3 => /usr/lib/x86_64-linux-gnu/libk5crypto.so.3 (0x00007f6e2cc54000)
        libcom_err.so.2 => /lib/x86_64-linux-gnu/libcom_err.so.2 (0x00007f6e2ca50000)
        libkrb5support.so.0 => /usr/lib/x86_64-linux-gnu/libkrb5support.so.0 (0x00007f6e2c845000)
        libresolv.so.2 => /lib/x86_64-linux-gnu/libresolv.so.2 (0x00007f6e2c629000)
        libsasl2.so.2 => /usr/lib/x86_64-linux-gnu/libsasl2.so.2 (0x00007f6e2c40e000)
        libgssapi.so.3 => /usr/lib/x86_64-linux-gnu/libgssapi.so.3 (0x00007f6e2c1cd000)
        libp11-kit.so.0 => /usr/lib/x86_64-linux-gnu/libp11-kit.so.0 (0x00007f6e2bf68000)
        libtasn1.so.6 => /usr/lib/x86_64-linux-gnu/libtasn1.so.6 (0x00007f6e2bd55000)
        libkeyutils.so.1 => /lib/x86_64-linux-gnu/libkeyutils.so.1 (0x00007f6e2bb50000)
        libheimntlm.so.0 => /usr/lib/x86_64-linux-gnu/libheimntlm.so.0 (0x00007f6e2b947000)
        libkrb5.so.26 => /usr/lib/x86_64-linux-gnu/libkrb5.so.26 (0x00007f6e2b6bd000)
        libasn1.so.8 => /usr/lib/x86_64-linux-gnu/libasn1.so.8 (0x00007f6e2b41a000)
        libhcrypto.so.4 => /usr/lib/x86_64-linux-gnu/libhcrypto.so.4 (0x00007f6e2b1e7000)
        libroken.so.18 => /usr/lib/x86_64-linux-gnu/libroken.so.18 (0x00007f6e2afd1000)
        libffi.so.6 => /usr/lib/x86_64-linux-gnu/libffi.so.6 (0x00007f6e2adc8000)
        libwind.so.0 => /usr/lib/x86_64-linux-gnu/libwind.so.0 (0x00007f6e2ab9f000)
        libheimbase.so.1 => /usr/lib/x86_64-linux-gnu/libheimbase.so.1 (0x00007f6e2a990000)
        libhx509.so.5 => /usr/lib/x86_64-linux-gnu/libhx509.so.5 (0x00007f6e2a744000)
        libsqlite3.so.0 => /usr/lib/x86_64-linux-gnu/libsqlite3.so.0 (0x00007f6e2a46f000)
        libcrypt.so.1 => /lib/x86_64-linux-gnu/libcrypt.so.1 (0x00007f6e2a237000)
winson@ubuntu-server:~/projects/git_projects/curl$

```

发现其使用的并不是位于/usr/local/lib里面的(curl源码编译的安装位置)，用的是另外一个，估计是系统自带的。
于是又Google一番，找到解决办法：让系统先找/usr/local/lib里面的库。修改/etc/ld.so.conf这个文件，在`include /etc/ld.so.conf.d/*.conf`上面`增加一行include /usr/local/lib`

然后执行命令:

``` shell
$ ldconfig -v
```

再次运行`curl --version`，发现版本一致了。

于是继续opencv的编译。
仍然有错误，似乎刚才的修改没有起作用，但是我们用命令行测试curl确实是成功的。于是继续Google，发现是cmake的坑。

cmake内部有一个curl，默认情况下其使用的就是这个，如果你不改配置的话。

于是重新编译cmake：

``` shell 
$ ./configure --system-curl
make
sudo make install
```

然后重新再编译opencv。终于通过了这次！全部编译成功。

## Reference:

[Building OpenCV4Android from trunk](http://code.opencv.org/projects/opencv/wiki/Building_OpenCV4Android_from_trunk)

[Automatic hunter download fails with "Protocol "https" not supported or disabled in libcurl"](https://github.com/ruslo/hunter/issues/328)

[Libcurl not updated](https://stackoverflow.com/questions/36866583/libcurl-not-updated)

[curl: (48) An unknown option was passed in to libcurl](https://stackoverflow.com/questions/11678085/curl-48-an-unknown-option-was-passed-in-to-libcurl)