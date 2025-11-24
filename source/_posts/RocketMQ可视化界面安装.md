---
title: RocketMQ可视化界面安装
top: false
cover: false
toc: true
mathjax: true
date: 2024-06-06 16:04:58
password:
summary:
tags: rocketmq
categories: 后端
---


# RocketMQ可视化界面安装

**起因：**访问rocketmq-externals项目的git地址，下载了源码，在目录中并没有找到rocketmq-console文件夹。
{% asset_img c05b0433185fafe34db4cdcb40dc2e19.png MQ仪表板 %}
git下面文档提示rocketMQ的仪表板转移到了新的项目中，点击[仪表板](https://github.com/apache/rocketmq-externals?tab=readme-ov-file)到新项目地址；

1. 下载源码

2. 进入到项目的resources资源目录下

   > rocketmq-dashboard\src\main\resources

3. 编辑application.yml配置文件，配置控制台的端口号和nameServer地址。
   {% asset_img 9a00bc614ed787fe1cbf8fe8a92e23c2.png 配置文件 %}

4. 在项目根目录下打开git命令窗,执行`mvn clean package -Dmaven.test.skip=true`

   ```java
   admin@DESKTOP-G6MAR8U MINGW64 /d/Program/rocketmq-dashboard (master)
   $ mvn clean package -Dmaven.test.skip=true
   ```

   **注意：**在这里可能会打包失败，

   具体异常：

   ```java
   [ERROR] Failed to execute goal com.github.eirslett:frontend-maven-plugin:1.11.3:yarn (yarn install) on project rocketmq-dashboard: Failed to run task: 'yarn install' failed. org.apache.commons.exec.ExecuteException: Process exited with an error: 1 (Exit value: 1) -> [Help 1]
   [ERROR]
   [ERROR] To see the full stack trace of the errors, re-run Maven with the -e switch.
   [ERROR] Re-run Maven using the -X switch to enable full debug logging.
   [ERROR]
   [ERROR] For more information about the errors and possible solutions, please read the following articles:
   [ERROR] [Help 1] http://cwiki.apache.org/confluence/display/MAVEN/MojoFailureException
   ```

   解决方法：

   打开项目中的pom.xml文件，注释以下两个插件

   ```xml
   <!--    <plugin>
                   <groupId>org.apache.maven.plugins</groupId>
                   <artifactId>maven-checkstyle-plugin</artifactId>
                   <version>2.17</version>
                   <executions>
                       <execution>
                           <id>validate</id>
                           <phase>validate</phase>
                           <configuration>
                               <excludes>src/main/resources</excludes>
                               <configLocation>style/rmq_checkstyle.xml</configLocation>
                               <encoding>UTF-8</encoding>
                               <consoleOutput>true</consoleOutput>
                               <failsOnError>true</failsOnError>
                           </configuration>
                           <goals>
                               <goal>check</goal>
                           </goals>
                       </execution>
                   </executions>
               </plugin>
   			-->
   <!--    <plugin>
                   <groupId>com.github.eirslett</groupId>
                   <artifactId>frontend-maven-plugin</artifactId>
                   <version>1.11.3</version>
                   <configuration>
                       <workingDirectory>frontend</workingDirectory>
                       <installDirectory>target</installDirectory>
                   </configuration>
                   <executions>
                       <execution>
                           <id>install node and yarn</id>
                           <goals>
                               <goal>install-node-and-yarn</goal>
                           </goals>
                           <configuration>
                               <nodeVersion>v16.2.0</nodeVersion>
                               <yarnVersion>v1.22.10</yarnVersion>
                           </configuration>
                       </execution>
   
                       <execution>
                           <id>yarn install</id>
                           <goals>
                               <goal>yarn</goal>
                           </goals>
                           <configuration>
                               <arguments>install</arguments>
                           </configuration>
                       </execution>
                       <execution>
                           <id>yarn build</id>
                           <goals>
                               <goal>yarn</goal>
                           </goals>
                           <configuration>
                               <arguments>build</arguments>
                           </configuration>
                       </execution>
                   </executions>
               </plugin> 
   		-->
   
   ```

   还有另一异常：

   ```java
   [ERROR] Failed to execute goal org.apache.maven.plugins:maven-antrun-plugin:1.8:run (default) on project rocketmq-dashboard: An Ant BuildException has occured: D:\ProgramFiles\RocketMQ\rocketmq-dashboard\frontend\build does not exist.
   [ERROR] around Ant part ...<copy todir="D:\ProgramFiles\RocketMQ\rocketmq-dashboard\target/classes/public">... @ 4:83 in D:\ProgramFiles\RocketMQ\rocketmq-dashboard\target\antrun\build-main.xml
   ```

   解决：在 rocketmq-dashboard\frontend 建一个build文件夹，然后再target目录下，建一个classes文件夹,classes下再建一个public文件夹

5. 执行mvn命令

   ```java
   mvn install -Dmaven.test.skip=true
   ```

6. 执行成功后，target目录下会有一个jar包，然后执行java命令运行就可以了。

7. 访问：127.0.0.1:9998  (jar运行的服务器地址 : xml中配置的端口号）

   {% asset_img bf119b4fad7460459535238ed6ddc43d.png "可视化界面" %}
   
   相关文章链接：
    [https://blog.csdn.net/qq_45515766/article/details/126360526](https://blog.csdn.net/qq_45515766/article/details/126360526)
    [https://blog.csdn.net/xiaoyiny/article/details/132134052](https://blog.csdn.net/xiaoyiny/article/details/132134052)