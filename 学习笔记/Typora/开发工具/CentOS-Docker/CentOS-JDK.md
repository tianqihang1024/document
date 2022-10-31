# CentOS8-JDK

## 1 下载

-   最新版本下载地址
    [http://www.oracle.com/technet...](https://link.segmentfault.com/?enc=9yuOf4qlT1%2BjFlSGgSK%2BwA%3D%3D.XE4HFq5BRpGoiHstkjJLd4oFLmYzIDx0YE8Uk%2BcdNze%2BIAwVur9Sc%2BMGBP5V2U469GPG1waquQ754YBCfRY4jAExRzE1Gp32T3NLG4oKQQY%3D)
-   历史版本下载地址
    [http://www.oracle.com/technet...](https://link.segmentfault.com/?enc=61lcMc5NvQUPElkxBz1AIg%3D%3D.lZ%2B4nW%2FKpeGtl%2B8B4%2F%2F4OxGk4u3u5jGsAF40d%2BJ6QswJPAqAtLorRkAYaDjspC3p4d2h3ZM0NNdzP3zdcOy3Uz%2BAzEJm9LfF%2FlAuIIiAh8k%3D)

## 2 、解压安装包

```shell
tar -zxvf jdk-8u301-linux-x64.tar.gz
```

## 3 配置环境变量

```shell
vim /etc/profile
```

在文本的最后一行粘贴

```bash
# java environment
export JAVA_HOME=JDK目录
export CLASSPATH=.:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$PATH:$JAVA_HOME/bin
```

## 4 配置生效，测试

```shell
# 刷新配置
source /etc/profile
# 测试
java -version
```