title: Spring MVC静态资源处理
date: 2014-09-20 18:00:21
tags: [spring]
---


在前几年的web开发中，url通常是以.do、.action、.xhtml等等作为结尾，现在是Rest的时代，这样的url显得非常ugly。老版本的Spring MVC不能很好的处理静态资源，所以在web.xml中通常配置`DispatcherServlet`的`url-pattern`类似.do、.action这种。因为如果请求映射配置成`/`的话，Spring MVC将拦截所有的请求（当然包括静态资源的请求），交由`Controller`处理，显然静态资源的请求到了Controller那里必然会导致`no handler mapping`的错误。

那么怎么样在配置请求映射为`\`的情况下，让Spring MVC能拦截所有请求，同时将静态资源的请求交给web服务器来处理呢？在Spring3.0的版本中，Spring的团队给出了两种解决方案。

### web.xml中DispatcherServlet配置

```
    <servlet>
        <servlet-name>springmvc</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <load-on-startup>1</load-on-startup>
    </servlet>
    
    <servlet-mapping>
        <servlet-name>springmvc</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
```

通过上面的配置，让Spring MVC拦截所有的请求。

<!--more-->

### 方案一：`<mvc:default-servlet-handler />`

在springmvc-servlet.xml中配置了`<mvc:default-servlet-handler />`之后，将在Spring MVC的context中定义一个`org.springframework.web.servlet.resource.DefaultServletHttpRequestHandler`类，这个类会检查每一个进入DispatcherServlet的url，如果是静态资源的请求，就将该请求转发给web服务器默认的servlet处理，如果是正常的业务请求则交由DispatcherServlet处理。

上文提到web服务器默认的servlet，一般的web服务器默认servlet命名为“default”，因此`DefaultServletHttpRequestHandler`能找到它并将静态资源请求转发给它处理，如果你所使用的web服务器默认的servlet名称不是“default”，可以通过`default-servlet-name`属性来指定：

```
 <mvc:default-servlet-handler default-servlet-name="xxx" />
```

附Tomcat的默认servlet命名，tomcat的默认servlet命名可以在{TOMCAT_HOME_DIR}/conf/web.xml中找到：

```
    <servlet>
        <servlet-name>default</servlet-name>
        <servlet-class>org.apache.catalina.servlets.DefaultServlet</servlet-class>
        <init-param>
            <param-name>debug</param-name>
            <param-value>0</param-value>
        </init-param>
        <init-param>
            <param-name>listings</param-name>
            <param-value>false</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>
```

### 方案二：`<mvc:resources />`

`<mvc:default-servlet-handler />`将静态资源请求交给web服务器默认servlet处理，而`<mvc:resources />`更牛逼，由Spring MVC自行处理静态资源请求，并且提供一些优化的手段。

`<mvc:resources />`允许将静态资源存放在任意的位置，WEB-INF，类路径等等，通过location属性指定静态资源的位置即可，打破了传统web容器静态资源只能放在根路径下的限制。

而且，`<mvc:resources />`根据Google page speed,yahoo yslow等浏览器优化原则对静态资源请求进行优化。比如可以通过cacheSeconds属性指定静态资源的缓存时间，充分利用客户端浏览器的缓存来提升响应能力，在response中根据配置设置响应报文头属性。

在springmvc-servlet.xml中配置如下：

```
    <mvc:resources location="/" mapping="/resources/**"/>
```

以上配置将web根路径映射为resources路径。假如根路径下有images,js目录，images里面有1.jpg,js里面有index.js等文件，则可以通过/resources/images/1.jpg和/resources/js/index.js访问这两个静态资源。

### 实践

在实践中，方案一中的`default-servlet-name`千万不可配置错误，否则将会报错：

```
Caused by:

org.springframework.web.util.NestedServletException: Request processing failed; nested exception is java.lang.IllegalStateException: A RequestDispatcher could not be located for the default servlet 'dsa'
```

如果将`default-servlet-name`和我们web.xml中的`servlet-name`混为一谈的话，将`default-servlet-name`设置为`servlet-name`，将会出现`java.lang.StackOverflowError`的堆栈溢出错误。