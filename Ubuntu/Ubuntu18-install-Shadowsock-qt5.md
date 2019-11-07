[引用地址](https://www.shangyexin.com/2018/04/20/shadowsocks-qt5/)

### Shdowsock-qt5 安装依赖 ###
1. cmake >= 3.1.0
2. qt5-qtbase-gui >= 5.2 (qtbase5 in Debian/Ubuntu)
3. qrencode (libqrencode in Debian/Ubuntu)
4. libQtShadowsocks >= 1.10.0 (libqtshadowsocks in Debian/Ubuntu. DON'T use the trunk code)
5. zbar (libzbar0 or libzbar in Debian/Ubuntu)
6. libappindicator (libappindicator1 in Debian/Ubuntu)

对应安装指令
```shell
sudo apt-get install cmake qtbase5-dev libqrencode-dev libzbar0 libappindicator1 libzbar-dev
```

其中libQtShadowsocks一定要源码安装,Botan在官方源中只有Botan1的版本,也要源码安装

### libQtShadowsocks 安装依赖


1. Qt >= 5.5
2. Botan-2 >= 2.3.0  Or Botan-1.10 (Not recommended)
3. CMake >= 3.1
4. A C++ Compiler that supports C++14 features (i.e. GCC >= 4.9)

为了支持`chacha20-ietf-poly1305`协议必须源码安装Botan-2*



### 源码链接

shadowsocks-qt5: [https://github.com/shadowsocks/shadowsocks-qt5/wiki/Compiling](https://github.com/shadowsocks/shadowsocks-qt5/wiki/Compiling)

libqtshadowsocks:[https://github.com/shadowsocks/libQtShadowsocks/wiki/Compiling](https://github.com/shadowsocks/libQtShadowsocks/wiki/Compiling)

Botan: [https://botan.randombit.net/releases](https://botan.randombit.net/releases)



### 安装步骤

#### 1.  安装Botan2以上版本,如下所示:

```shell
wget https://botan.randombit.net/releases/Botan-2.3.0.tgz
tar xvf Botan-2.3.0.tgz
cd Botan-2.3.0
./configure.py
make -j4
sudo make install
sudo ldconfig
```
#### 2. 安装libqtshadowsocks,如下所示:

   ```shell
mkdir build && cd build
cmake .. -DUSE_BOTAN2=ON #支持chacha20-ietf-poly1305
make -j4
sudo make install
   ```
#### 3. 安装shadowsocks-qt5,如下所示:

```shell
mkdir build && cd build
cmake .. 
make -j4
sudo make install
```





   