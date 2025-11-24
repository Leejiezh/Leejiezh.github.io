---
title: mvn命令报没有配置jdk环境
top: false
cover: false
toc: true
mathjax: true
date: 2024-06-06 14:47:45
password:
summary:
tags: maven
categories: 后端
---


# mvn 命令报没有配置jdk环境

jdk环境变量配置没问题，并且java -version 也正常显示。

执行mvn -v 报错误：

```
/d/Program Files/maven/apache-maven-3.9.5/bin/mvn: line 93: cd: C:\Program Files\Java\jdk-19;: No such file or directory
The JAVA_HOME environment variable is not defined correctly,
this environment variable is needed to run this program.
```

**解决方法：**

1. 命令行设置*JAVA_HOME*

```java
//自己本机jdk安装目录（C:\Program Files\Java\jdk-19）
setx JAVA_HOME "C:\Program Files\Java\jdk-19"
```

2. 更新*Path*变量

```java
//maven的安装目录（D:\Program Files\maven\apache-maven-3.9.5）
setx PATH "%JAVA_HOME%\bin;D:\Program Files\maven\apache-maven-3.9.5\bin;%PATH%"
```

3. *关闭并重新打开命令提示符*，然后运行以下命令验证配置：

```java
java -version
mvn -version
```

