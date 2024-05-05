# FFmpeg

Git地址：[git@github.com:FFmpeg/FFmpeg.git](git@github.com:FFmpeg/FFmpeg.git)

* 使用`./configure -h`查看所有编译选项。
* 查看`ffbuild/config.log`排查编译错误。

编译命令如下：
```sh
./configure \
    --prefix=$PREFIX \
    --arch=$ARCH \
    --cpu=$CPU \
    --enable-cross-compile \
    --target-os=android \
    --cc=$CC \
    --cxx=$CXX \
    --ar=$AR \
    --nm=$NM \
    --ranlib=$RANLIB \
    --strip=$STRIP \
    --sysroot=$SYSROOT \
    --extra-cflags="-Os -fpic $OPTIMIZE_CFLAGS $SSL_CFLAGS" \
    --extra-ldflags="$ADDI_LDFLAGS $SSL_LDFLAGS" \
    $COMMON_FF_CFG_FLAGS
    
  make clean
  make -j8
  make install
```
**但不知为何指定`--as=$AS`选项后编译过程会卡住。**

**如果要支持https协议需要依赖OpenSSL**

编译脚本：[https://github.com/rookieeeeeee/MediaDemo/blob/master/build_ffmpeg.sh](https://github.com/rookieeeeeee/MediaDemo/blob/master/build_ffmpeg.sh)

# OpenSSL
Git地址：[git@github.com:openssl/openssl.git](git@github.com:openssl/openssl.git)

* 需要设置环境变量：`ANDROID_NDK_ROOT`指向NDK目录
* 需要指定编译目标平台：`android-arm`,`android-arm64`,`android-x86`,`android-x86_64`
* 需要设置环境变量`CC`,`CXX`,`LINK`,`LD`,`AR`,`RANLIB`,`STRIP`,`OBJDUMP`

```sh
./Configure "$PLATFORM" \
    no-shared \
    --prefix="$PREFIX" \
    -D__ANDROID_API__=$API

  make clean
  make
  make install
```

编译脚本：[https://github.com/rookieeeeeee/MediaDemo/blob/master/build_openssl.sh](https://github.com/rookieeeeeee/MediaDemo/blob/master/build_openssl.sh)