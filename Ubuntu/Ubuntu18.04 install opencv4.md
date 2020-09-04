[引用地址](https://www.linuxidc.com/Linux/2019-05/158419.htm)

### 安装依赖 ###

```shell
$ sudo apt install -y build-essential cmake git pkg-config libgtk2.0-dev libopenexr-dev 
$ sudo apt install -y gfortran libblas-dev liblapack-dev libeigen3-dev 
$ sudo apt install -y Python-dev python-numpy libtbb2 libtbb-dev libjpeg-dev libpng-dev libtiff-dev libdc1394-22-dev libjasper-dev 
$ sudo apt install -y libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev
$ sudo apt install -y libavcodec-dev libavformat-dev libswscale-dev libavutil-dev libavresample-dev libxvidcore-dev libx264-dev
$ sudo apt install vtk7
```

官网下载最新版本的[opencv](https://github.com/opencv/opencv/archive/4.0.1.zip)和[opencv_contrib](https://github.com/opencv/opencv_contrib/archive/4.0.1.zip)源码压缩包。并解压到同一文件夹下。

```
$ unzip opencv.zip
$ unzip opencv_contrib.zip
```

### 编译安装 

进入解压出的`opencv`目录，创建`build`目录，按需配置`cmake`参数并执行，最后`make`，再`make install`。
根据需要配置需要编译的模块，如下例子所示：

```shell
$ cd opencv
$ mkdir build
$ cd build
$ cmake -D CMAKE_BUILD_TYPE=Release \
    -D CMAKE_INSTALL_PREFIX=/usr/local \
    -D OPENCV_EXTRA_MODULES_PATH=../../opencv_contrib/modules \
    -D OPENCV_GENERATE_PKGCONFIG=YES \
    -D WITH_1394=OFF ..
$ make -j8
$ sudo make install
$ sudo ldconfig
```

对于OPENCV_EXTRA_MODULES_PATH选项会安装opencv的额外库文件,有可能会出下载文件,若长时间没有下载速度,可尝试手动下载,并修改对应的

最方便的办法是终端翻墙

```   shell
$ gedit opencv_contrib/modules/faceCMakeLists.txt
```

中的下载curl为本地地址,并重新编译



然后添加环境变量`PKG_CONFIG_PATH`到`~/.bashrc`

```shell
export PKG_CONFIG_PATH=/usr/local/lib/pkgconfig
```

### 验证安装

通过编译一个例子程序来验证安装成功。

```shell
$ cd ..
$ cd opencv/samples/cpp/example_cmake  #解压下的opencv文件夹下
$ cmake .
$ make
$ ./opencv_example
```

如果连接有摄像头，会看到窗口有摄像头的内容。



### clion

对于clion初次需要修改`CMakeLists.txt`文件并添加以下代码,并重新加载CMakeLists

```c++
find_package(OpenCV REQUIRED)
target_link_libraries(opencv_test ${OpenCV_LIBS} )
// opencv_test 为项目名称
```
