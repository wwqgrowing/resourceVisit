最近一直都是在用springboot搭建项目，项目一般都是前后端分离，当然也不用担心静态资源访问的问题，恰好有朋友问到springMVC项目中页面中引入css、js等静态资源无法访问的问题，刚好抽时间来把问题给还原一下，顺便记录一下解决问题的过程

## 本文目录
* 场景还原
* 问题分析
* 解决方案

## 场景还原
首先，通过eclipse搭建一个maven项目，转成web工程，项目工程结构如下：
![](https://upload-images.jianshu.io/upload_images/9692838-8dc1741bb2af004c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/840)
在本项目中，将页面、以及静态资源css、images等都放到resources目录下的templates目录下，在页面hellpspring.jsp中引入静态资源，项目使用tomcat启动，然后通过浏览器访问``` http://localhost:8080/resourceVisit/hello ```，效果如下，出现了资源找不到的错误![](https://upload-images.jianshu.io/upload_images/9692838-002c8e48bc19b907.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/840)
在后台控制台会报错，报错日志信息为：![](https://upload-images.jianshu.io/upload_images/9692838-c030a2824b86302a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 问题分析
根据页面报错的信息来看，静态资源访问不到，通过后台控制台的日志来看，我们的css和images等静态资源也会被dispatchServlet拦截处理，这是由于我们在web.xml中配置了
``` 
  <servlet-mapping>
      <servlet-name>dispatcher</servlet-name>
<!-- “/” 对所有链接都拦截，所以静态资源的链接也被拦截了 -->
      <url-pattern>/</url-pattern>
  </servlet-mapping>
``` 
## 解决方案：
######1.通过激活Tomcat的defaultServlet来处理静态文件：
在web.xml中对不需要拦截的都需要配置一下,注意要写在DispatcherServlet的前面， 让defaultServlet先拦截，这个就不会进入Spring了
```
<servlet-mapping>
     <servlet-name>default</servlet-name>
     <url-pattern>*.png</url-pattern>
</servlet-mapping>
<servlet-mapping>
     <servlet-name>default</servlet-name>
     <url-pattern>*.css</url-pattern>
</servlet-mapping>
<servlet-mapping>
      <servlet-name>default</servlet-name>
      <url-pattern>*.js/</url-pattern>
</servlet-mapping>
```
设置好之后，重启tomcat服务
打开浏览器再次访问，```http://localhost:8080/resourceVisit/hello ```
![](https://upload-images.jianshu.io/upload_images/9692838-002c8e48bc19b907.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/840)
发现页面还是同样的错误，不过后台控制台没有日志报错，说明静态资源没有被dispatchServlet拦截，设置生效了，那为什么还是访问不到静态资源呢？

  继续分析，我们查看一下项目部署打包的路径，右击项目选中```properties```，找到```Deployment Assembly```，如下图：![](https://upload-images.jianshu.io/upload_images/9692838-09e2c9be5118532a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/840)
我们发现，项目对应的resource目录的内容都被打包到了```WEB-INF/classess```目录下去了，其中```WEB-INF```这个目录比较特殊，做为WEB应用的安全目录存在。Servlet规范是对此也有要求：
> A special directory exists within the application hierarchy named “WEB-INF”. ```This directory contains all things related to the application that aren’t in the document root of the application```.

即所有与应用相关的，但又不能放到根目录下的文件可以放在这里。
包含的内容大致有以下几类：
   *  web.xml
   * 对于servlet 3.0，支持其web-fragment.xml的声明。
   * classes目录，用于存放所有编译过的应用的class文件
   * lib目录，存放应用依赖的，第三方的jar文件。

对于WEB-INF目录的访问，规范中有如下约束：
>The Web application class loader must load classes from the WEB-INF/classes directory first, and then from library JARs in the WEB-INF/lib directory. ```Also, except for the case where static resources are packaged in JAR files, any >requests from the client to access the resources in WEB-INF/ directory must be >returned with a SC_NOT_FOUND(404) response```.

所以我们在页面中引用css、images会有了如上的错误，我们只需要把静态资源目录放到和WEB-INF同级目录即可，调整后的工程目录如图：![](https://upload-images.jianshu.io/upload_images/9692838-b0d5b5daa451537d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/840)
再次访问，```http://localhost:8080/resourceVisit/hello ```，发现资源可以正常访问，问题解决
![](https://upload-images.jianshu.io/upload_images/9692838-e1967d7999e5a22c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/840)

######2.使用MVC的default-servlet-handler解决
在Spring配置文件 `springContext.xml` 增加以下配置即可：
```
<!-- 配置mvc注解驱动 -->
<mvc:annotation-driven />
<!-- 配置default-servlet-handler -->
<mvc:default-servlet-handler/>
```
注意：使用这种方式的前提和第一种方法一样，要求相关的静态资源文件要放到`WEB-INF`的同级目录下
######3.在spring3.0.4以后版本提供了mvc:resources
只需在srping配置文件`springContext.xml`中配置静态资源的映射即可，配置如下：
```
    <mvc:resources mapping="/images/**" location="/WEB-INF/classes/templates/images/"/>
      <!-- 使用classpath路径 -->
	<mvc:resources mapping="/css/**" location="classpath:/templates/css/"/>
```
`mapping`是指定要处理的映射
`location`对应项目发布的资源文件目录
`classpath`是指 WEB-INF文件夹下的classes目录 
使用此种方式，不要求静态资源放在`WEB-INF`同级目录
注意：如果访问出现错误`警告: No mapping found for HTTP request with URI [/resourceVisit/hello] in DispatcherServlet with name 'dispatcher'`,可能是因为没有配置`<mvc:annotation-driven />`的原因

## 总结
建议项目从开始设计时，把静态资源以及页面文件放到`WEB-INF`的同级目录，这样只需要用`方案1`和`方案2`即可解决静态资源文件的访问问题，如果项目已经定型，不想更换目录的话，用`方案3`的方式去解决静态资源访问问题

参考：
1.  [WEB-INF目录知多少](https://mp.weixin.qq.com/s/ZLzizw5UQ-gDEvnJpMfGbQ)
2. [javaweb应用程序的规范目录结构](https://www.douban.com/group/topic/103418474/)
3. [工程git项目地址](https://github.com/wwqgrowing/resourceVisit.git)






