# Springboot基础

**<font color="red">一个快速开发框架，快速整合常用的第三方依赖，简化xml配置，全注解化，内置http服务器。</font>**



#### 核心原理

1. 基于SpringMVC无配置文件完全注解化 + 内置tomcat-embed-core + Main函数启动
2. Maven继承依赖关系



#### web.xml配置文件

传统web项目，通过web.xml配置文件加载整个项目流程。

**web.xml详解**

当启动一个web项目是，容器（JBoss、Tomcat等中间件）首先会读取项目web.xml配置文件里的配置，这一步骤没有出错并完成后，项目才能被正常启动起来。

①  web项目启动，容器首先去配置文件web.xml读取两个节点<listener></listener>和<context-param></context-param>。

```JS
 <listener>    
    <listener-class>    
        com.myapp.LogbackConfigListener    
    </listener-class>    
</listener>  
```

<listener></listener>用来注册一个监听类

<listen-class></listen-class>指定一个监听类，该类需要继承ServletContextListener接口。

```js
 <context-param>    
    <param-name>logbackConfigLocation</param-name>    
    <param-value>WEB-INF/logback.xml</param-value>    
</context-param>
```

相当于键值对。

②  紧接着，容器将创建一个<font color = "orange">ServletContext</font>，这个web项目所有部分都将共享这个上下文

③  容器以<context-param></context-param>中的name作为键，value作为值，将其转换成键值对，存入ServletContext。

④  容器创建<listener></listener>中的类实例，根据配置的class类路径<listener-class></listener-class>来创建监听，在监听中会有contextInitialized(ServletContextEvent args)初始化方法，启动Web应用时，系统调用Listener的该方法。

⑤  容器读取<filter></filter>，根据指定的类路径来实例过滤器。

以上工作都是在Web项目还没有完全启动起来的时候就已经完成，如果系统中有servlet，则servlet是在第一次发起请求的时候实例化，而且一般不会被容器销毁，它可以服务于多个用户的请求。

**<font color="red">web.xml的加载顺序：<context-param> -> <listener> -> <filter> -> <servlet></font>**

相同元素则按照出现顺序。

而在spring boot中，内置注解加载整个SpringMVC容器，相当于使用java编写SpringMVC初始化。



#### pom.xml配置文件

```
<parent>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-parent</artifactId>
	<version>2.1.6.RELEASE</version>
	<relativePath/>
</parent>
```

​    配置父工程依赖

```
<dependencies>	
	<dependency>
		...
	</dependency>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-web</artifactId>
		<exclusions>
	        <exclusion>
	            <groupId>org.springframework.boot</groupId>
	            <artifactId>spring-boot-starter-tomcat</artifactId>
	        </exclusion>
	    </exclusions>
	</dependency>
</dependencies>
```

配置子工程依赖

<groupID>和<ArtifactID>统称为“坐标”，groupID分为多个段，第一段为域，第二段为公司名称。groupID是项目组织唯一的标识符，实际对应项目的名称，就是main目录里java的目录结构。ArtifactID是项目的唯一标识符，实际对应项目的名称，是项目根目录的名称。

```js
<build>
	<finalName>demo</finalName>    <!-- 这个标签用于打war包时确定生成war包的名字 -->
	<plugins>
		<plugin>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-maven-plugin</artifactId>
			<configuration>
				<source>1.8</source>
				<target>1.8</target>
			</configuration>
		</plugin>
	</plugins>
</build>
```



#### 目录结构及资源文件访问

1.src/main/java：存放源码

2.src/main/resources/

​		① static/：存放静态文件，如html、css、js、image

​		② templates/：存放动态页面，如jsp、html、tpl

​		③config/：存放配置文件，application.properties、application.yml