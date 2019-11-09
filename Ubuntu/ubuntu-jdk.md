## Ubuntu 18.04 安装JDK环境

在oracle官网安装jdk,选择`Linux x86` 平台
https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html
解压然后放在/usr/local/目录下,并配置环境变量.

```shell
vim /etc/profile
```
假设jdk解压后文件名为`jdk1.8`
则添加以下内容,并`source`

```shell
export JAVA_HOME=/usr/local/jdk1.8
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
export PATH=.:${JAVA_HOME}/bin:$PATH
```
输入`java -version`观察是否安装成功
为了使其他文件检测到,需要添加连接
如下所示:

```shell
sudo update-alternatives --install /usr/bin/java  java  /usr/local/jdk1.8/bin/java 300
sudo update-alternatives --install /usr/bin/javac  javac  /usr/local/jdk1.8/bin/javac 300
```
