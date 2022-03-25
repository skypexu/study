CMake 学习笔记(1) - 外部项目引入时的patch问题
===========================================

## 1. CMake构造项目时允许使用ExternalProject_Add来添加外来项目。例如把leveldb添加进来的指令

```
ExternalProject_Add(leveldb
        DOWNLOAD_DIR ${ExtProj_DOWNLOAD_DIR}/leveldb
        PREFIX ${ExtProj_DIR}/leveldb
        URL "https://github.com/google/leveldb/archive/a53934a3ae1244679f812d998a4f16f2c7f309a6.tar.gz"
        URL_HASH MD5=458aba294697d3fa1367a1916e4a72c0
        BUILD_IN_SOURCE 1
        CONFIGURE_COMMAND ""
        BUILD_COMMAND make -j
        PATCH_COMMAND patch < ${CMAKE_CURRENT_LIST_DIR}/patches/leveldb/leveldb.patch
        INSTALL_COMMAND sh ${CMAKE_CURRENT_LIST_DIR}/install_leveldb.sh ${ExtProj_DIR}/leveldb/src/leveldb ${ExtProj_DIR}/local
```

这里DOWNLOAD_DIR指示下载下来的包应该放在那个目录，PREFIX指解开包的目录应该在哪里。
URL是路径，还可以用URL_HASH来指定校验吗，检验下载的文件是否完好。
一个外部项目的制作通常包含的过程是:download,update,patch,configure,build,install。

外部包下载完后，被重新extract，会覆盖已经修改的文件，这自然包括那些被patch修改了的文件，但是也要注意，如果你的patch生成了新的文件，
要想办法在PATCH_COMMAND中删除掉，否则下次重复上述过程会让patch运行时发现有文件存就会导致错误。

## 2. ExternalProject_Add与git

ExternalProject_Add还允许使用git下载源码，这个时候，DOWNLOAD_DIR不起作用，而PREFIX
指定的位置用来存放git下载的源码，或者可以指定SOURCE_DIR来指定存放source code的位置。
外部项目的制作依然包含过程download,update,patch,configure,build,install。
例如下面的指令把brpc导入到进来:
```
ExternalProject_Add(brpc
        DOWNLOAD_DIR ${ExtProj_DOWNLOAD_DIR}/brpc
        PREFIX ${ExtProj_DIR}/brpc
        GIT_REPOSITORY "https://github.com/apache/incubator-brpc"
        GIT_TAG "1b9e00641cbec1c8803da6a1f7f555398c954cb0"
        PATCH_COMMAND git reset --hard
        COMMAND git clean -d -f
        COMMAND patch -p1 < ${CMAKE_SOURCE_DIR}/thirdparties/brpc/brpc.patch
        CMAKE_ARGS -DCMAKE_PREFIX_PATH=${ExtProj_DIR}/local -DCMAKE_INSTALL_INCLUDEDIR=${ExtProj_DIR}/local/include/brpc
                -DCMAKE_INSTALL_LIBDIR=${ExtProj_DIR}/local/lib
)
```

由于ExternalProject在每次make时都会被执行，那么如果我们有一个patch在上次已经打过了，这次update后再执行patch，patch程序就会报错，
这个过程是自动的，很难阻止。有什么办法回避这个问题呢? 一个方法是，整个过程从头开始，
我们要做的就是在patch之前先执行git reset --hard，然后用git clean把上次patch新生成的文件也删除掉，
然后再用COMMAND patch指令打上patch。

## 3. 总结

包下载方式和git clone方式在打patch时使用了不同的方法。其他还有svn，因为时间有限，没有继续研究。
