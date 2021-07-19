今日思语：I miss you？ 何解？ 我错过你了？我想你了？

 

当下许多公司都会选择使用springboot作为服务应用开发框架，springboot框架提供了一套自己的打包机制，是通过spring-boot-maven-plugin插件来实现的。

### 1、spring-boot-maven-plugin引入pom

对于新建的一个springboot项目来说，pom中会加入插件：

![img](https://cdn.jsdelivr.net/gh/OneYX/resources@master/images/2021/07/19/20210719210459.png)

通过idea可以看到maven中包含了spring-boot-maven-plugin插件：

![img](https://cdn.jsdelivr.net/gh/OneYX/resources@master/images/2021/07/19/20210719210507.png)

功能说明：

* build-info：生成项目的构建信息文件 build-info.properties
* repackage：这个是默认 goal，在 `mvn package` 执行之后，这个命令再次打包生成可执行的 jar，同时将 `mvn package` 生成的 jar 重命名为 `*.origin`
* run：这个可以用来运行 Spring Boot 应用
* start：这个在 `mvn integration-test` 阶段，进行 `Spring Boot` 应用生命周期的管理
* stop：这个在 `mvn integration-test` 阶段，进行 `Spring Boot` 应用生命周期的管理

spring-boot-maven-plugin插件默认在父工程sprint-boot-starter-parent中被指定为repackage，可以点击sprint-boot-starter-parent进入父pom进行查看，如下图：

![img](https://cdn.jsdelivr.net/gh/OneYX/resources@master/images/2021/07/19/20210719210514.png)

如果需要设置其他属性，需要在当前应用的pom中进行设置。

### 2、执行打包命令

```
mvn clean package
```

或者通过开发工具如idea执行clean和package俩命令：

![img](https://cdn.jsdelivr.net/gh/OneYX/resources@master/images/2021/07/19/20210719210522.png)

执行以上命令时会自动触发spring-boot-maven-plugin插件的repackage目标，完后可以在target目录下看到生成的jar，如下图：

![img](https://cdn.jsdelivr.net/gh/OneYX/resources@master/images/2021/07/19/20210719210528.png)

这里可以看到生成了两个jar相关文件，其中common.jar是spring-boot-maven-plugin插件重新打包后生成的可执行jar，即可以通过java -jar common.jar命令启动。common.jar.original这个则是mvn package打包的原始jar，在spring-boot-maven-plugin插件repackage命令操作时重命名为xxx.original，这个是一个普通的jar，可以被引用在其他服务中。

### 3、jar内部结构

对这两个jar文件解压看看里面的结构差异：

**3.1 common.jar目录结构如下：**

![img](https://cdn.jsdelivr.net/gh/OneYX/resources@master/images/2021/07/19/20210719210535.png)

其中BOOT-INF主要是一些启动信息，包含classes和lib文件，classes文件放的是项目里生成的字节文件class和配置文件，lib文件是项目所需要的jar依赖。

META-INF目录下主要是maven的一些元数据信息，MANIFEST. MF文件内容如下：

[![复制代码](Dloader.path失效.assets/copycode.gif)](javascript:void(0); )

```
Manifest-Version: 1.0
Implementation-Title: java-common-utils
Implementation-Version: 0.0.1-SNAPSHOT
Start-Class: com.common.util.CommonUtilsApplication
Spring-Boot-Classes: BOOT-INF/classes/
Spring-Boot-Lib: BOOT-INF/lib/
Build-Jdk-Spec: 1.8
Spring-Boot-Version: 2.1.9.RELEASE
Created-By: Maven Archiver 3.4.0
Main-Class: org.springframework.boot.loader.JarLauncher
```

[![复制代码](Dloader.path失效.assets/copycode.gif)](javascript:void(0); )

其中Start-Class是项目的主程序入口，即main方法。Springboot-Boot-Classes和Spring-Boot-Lib指向的是生成的BOOT-INF下的对应位置。

Main-Class属性值为org.springframework.boot.loader. JarLauncher，这个值可以通过设置属性layout来控制，如下：

[![复制代码](Dloader.path失效.assets/copycode.gif)](javascript:void(0); )

```
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <!--使用-Dloader.path需要在打包的时候增加<layout>ZIP</layout>，不指定的话-Dloader.path不生效-->
        <layout>ZIP</layout>
        <!-- 指定该jar包启动时的主类[建议] -->
        <mainClass>com.common.util.CommonUtilsApplication</mainClass>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>repackage</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

[![复制代码](Dloader.path失效.assets/copycode.gif)](javascript:void(0); )

设置<layout>ZIP</layout>时Main-Class为org.springframework.boot.loader. PropertiesLauncher，具体layout值对应Main-Class关系如下：

* JAR，即通常的可执行jar

Main-Class: org.springframework.boot.loader. JarLauncher

* WAR，即通常的可执行war，需要的servlet容器依赖位于WEB-INF/lib-provided

Main-Class: org.springframework.boot.loader.warLauncher

* ZIP，即DIR，类似于JAR

Main-Class: org.springframework.boot.loader. PropertiesLauncher

* MODULE，将所有的依赖库打包（scope为provided的除外），但是不打包Spring Boot的任何Launcher
* NONE，将所有的依赖库打包，但是不打包Spring Boot的任何Launcher

common.jar之所以可以使用java -jar运行，和MANIFEST. MF文件里的配置关系密切

**3.2 original jar包结构**

![img](https://cdn.jsdelivr.net/gh/OneYX/resources@master/images/2021/07/19/20210719210547.png)

可以看到通过mvn package构建的jar是一个普通的jar, 包含的都是项目的字节文件和一些配置文件，没有将项目依赖的第三方jar包含进来。再看下MANIFEST. MF文件：

```
Manifest-Version: 1.0
Implementation-Title: java-common-utils
Implementation-Version: 0.0.1-SNAPSHOT
Build-Jdk-Spec: 1.8
Created-By: Maven Archiver 3.4.0
```

其中没有包含Start-Class、Main-Class等信息，这个与可执行jar的该文件存在很多差异，而且目录结构也有很大差异。

一般对使用spring-boot-maven-plugin插件打出的可执行jar不建议作为jar给其他服务引用，因为可能出现访问可执行jar中的一些配置文件找不到的问题。如果想让构建出来的原始jar不被重新打包，可以对spring-boot-maven-plugin插件配置classifier属性，自定义一个可运行jar名称，这样该插件就不会对原始的jar重命名操作了。

[![复制代码](Dloader.path失效.assets/copycode.gif)](javascript:void(0); )

```
<configuration>
    <!-- 指定该jar包启动时的主类[建议] -->
    <mainClass>com.common.util.CommonUtilsApplication</mainClass>
    <!--配置的 classifier 表示可执行 jar 的名字，配置了这个之后，在插件执行 repackage 命令时，
    就不会给 mvn package 所打成的 jar 重命名了,这样就可以被其他项目引用了，classifier命名的为可执行jar-->
    <classifier>myexec</classifier>
</configuration>
```

[![复制代码](Dloader.path失效.assets/copycode.gif)](javascript:void(0); )

效果如下：

![img](https://cdn.jsdelivr.net/gh/OneYX/resources@master/images/2021/07/19/20210719210553.png)

 

以上是对spring-boot-maven-plugin插件的打包机制和jar包结构的一些分析。
