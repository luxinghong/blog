---
layout: post
title: "SpringMVC参数映射原理"
subtitle: ""
author: ""
header-style: text
tags:
  - Spring
---



# 方法参数带 @RequestBody 注解

 ```java
 @RequestMapping("/testRb")
     public Person testRb(@RequestBody Person p) {
         return p;
     }
 ```

贴上 @RequestBody 的说明：

> Annotation indicating a method parameter should be bound to the body of the web request. The body of the request is passed through an HttpMessageConverter to resolve the method argument depending on the content type of the request. Optionally, automatic validation can be applied by annotating the argument with @Valid.
> Supported for annotated handler methods.

翻译一下：

1. 该注解指示方法参数应通过请求的body来绑定；

2.   根据请求头的 content-type 选择对应的 HttpMessageConverter 对body中的内容进行转换；
3.   可以通过 @Valid 注解对方法参数进行自动验证.

这就很明显指出了几个重点，请求应该带body，content-type很重要。**所以只要保证带有body体并且content-type能找到对应的HttpMessageConverter就能解析成功。**



接下来做几个测试。

### GET 请求方式，传递的值放到请求参数上

![](/blog/img/20190801170027731.png)

结果是：提示缺少body体

![](/blog/img/20190801170104457.png)

原因： 很明显，请求中没有带body体



### GET方式，请求中带body体

结果： 请求没有body体

![](/blog/img/20190801170633565.png)

原因： 我们已经带了body，为什么还报错？上面说过还有一个重要的因素：content-type。在这种方式下的content-type是multipart/form-data 。 而这个content-type并没有对应的 HttpMessageConverter，所有没有进行body解析，依然认为body体是空的。至于什么是multipart/form-data，可参考 [multipart/form-data](https://www.cnblogs.com/tylerdonet/p/5722858.html) 。

默认的HttpMessageConverter 有 10 个 ，每个converter都有自己支持的MediaType 。

![](/blog/img/20190801171744977.png)

 ![](/blog/img/20190801171319787.png)

 同理，content-type 为 application/x-www-form-urlencoded 时跟上面结果是一样的。





### GET方式，content-type 改为 application/json ，结果正确

![](/blog/img/20190801172556267.png)

原因：因为application/json 的 content-type 对应到了 MappingJackson2HttpMessageConverter ，该converter将body中的内容转为了对应的实体对象。注：这有一个误区，get不能带body？上述证明，get是可以带body的，而且http规范中也没有规定get不可以带body，只是通常不这么做而已。 





### 上述方式改为POST，过程是一致的，只是报错信息变了

![](/blog/img/20190801183257294.png)


先看一段代码：


```java
	if (body == NO_VALUE) {
		if (httpMethod == null || !SUPPORTED_METHODS.contains(httpMethod) ||
				(noContentType && !message.hasBody())) {
			return null;
		}
		throw new HttpMediaTypeNotSupportedException(contentType, this.allSupportedMediaTypes);
	}
```
 在没有解析到body时走的都是这个逻辑，因为 SUPPORTED_METHODS 只包含 POST、PUT、PATCH，所以GET是return null，POST是抛出异常。



### @RequestBody是如何被解析的。

其实我们可以猜想一下，尤其是对于Spring这么优秀的框架，命名一定即规范又易懂。既然是解析方法参数，那么这种类大体应该叫MethodArgumentsResolver之类的，然后全局搜索一下，发现果然有这么个接口  HandlerMethodArgumentResolver ，再看下它的实现类，我们发现了 RequestResponseBodyMethodProcessor  。里面有几个方法：

```java
@Override
	public boolean supportsParameter(MethodParameter parameter) {
		return parameter.hasParameterAnnotation(RequestBody.class);
	}
	
@Override
public boolean supportsReturnType(MethodParameter returnType) {
	return (AnnotatedElementUtils.hasAnnotation(returnType.getContainingClass(), ResponseBody.class) ||
			returnType.hasMethodAnnotation(ResponseBody.class));
	}
```
一目了然，我们看到了 RequestBody 、ResponseBody 两个类，不就是参数上用的注解么。所以这个类表明它是用来处理带 @RequestBody 注解 和 @ResponseBody 的方法的。具体的解析逻辑在这个类的 resolveArgument 方法中。

**上述原理对应源码流程是：`RequestResponseBodyMethodProcessor.resolveArgument() `  ->  ` readWithMessageConverters() `   ->  ` (父类)AbstractMessageConverterMethodArgumentResolver.readWithMessageConverters()` ** . 可自行查阅。

 



# 参数中直接放实体对象

```java
@RequestMapping("/testRb")
    public Person testRb(Person p) {
        return p;
    }
```

先说一下测试结论，测试过程就不再截图。

这种方式不会报错，只是Person p会接收不到值，它的属性全部为null。

能正常接收到的方式有：

1. GET +  查询参数

2. GET + body + ContentType(multipart/form-data)
3. POST + body + ContentType(multipart/form-data)
4. POST + body + ContentType(application/x-www-form-urlencoded)

其他都不行，包括 POST + body + ContentType(application/json)。

这一部分还想多说一点，干脆连springMVC整个的请求流程也简单说一下好了。

先贴一个完整的调用栈，从中我们可以看到很多有意思的信息：

```java
54   testRb:24, DemoSpringApplication (io.lxh.demo)
53   invoke:-1, GeneratedMethodAccessor32 (sun.reflect)
52   invoke:43, DelegatingMethodAccessorImpl (sun.reflect)
51   invoke:498, Method (java.lang.reflect)
50   doInvoke:189, InvocableHandlerMethod (org.springframework.web.method.support)
49   invokeForRequest:138, InvocableHandlerMethod (org.springframework.web.method.support)
48   invokeAndHandle:102, ServletInvocableHandlerMethod (org.springframework.web.servlet.mvc.method.annotation)
47   invokeHandlerMethod:892, RequestMappingHandlerAdapter (org.springframework.web.servlet.mvc.method.annotation)
46   handleInternal:797, RequestMappingHandlerAdapter (org.springframework.web.servlet.mvc.method.annotation)
45   handle:87, AbstractHandlerMethodAdapter (org.springframework.web.servlet.mvc.method)
44   doDispatch:1038, DispatcherServlet (org.springframework.web.servlet)
43   doService:942, DispatcherServlet (org.springframework.web.servlet)
42   processRequest:1005, FrameworkServlet (org.springframework.web.servlet)
41   doGet:897, FrameworkServlet (org.springframework.web.servlet)
40   service:634, HttpServlet (javax.servlet.http)
39   service:882, FrameworkServlet (org.springframework.web.servlet)
38   service:741, HttpServlet (javax.servlet.http)
37   internalDoFilter:231, ApplicationFilterChain (org.apache.catalina.core)
36   doFilter:166, ApplicationFilterChain (org.apache.catalina.core)
35   doFilter:53, WsFilter (org.apache.tomcat.websocket.server)
34   internalDoFilter:193, ApplicationFilterChain (org.apache.catalina.core)
33   doFilter:166, ApplicationFilterChain (org.apache.catalina.core)
32   doFilterInternal:99, RequestContextFilter (org.springframework.web.filter)
31   doFilter:107, OncePerRequestFilter (org.springframework.web.filter)
30   internalDoFilter:193, ApplicationFilterChain (org.apache.catalina.core)
29   doFilter:166, ApplicationFilterChain (org.apache.catalina.core)
28   doFilterInternal:92, FormContentFilter (org.springframework.web.filter)
27   doFilter:107, OncePerRequestFilter (org.springframework.web.filter)
26   internalDoFilter:193, ApplicationFilterChain (org.apache.catalina.core)
25   doFilter:166, ApplicationFilterChain (org.apache.catalina.core)
24   doFilterInternal:93, HiddenHttpMethodFilter (org.springframework.web.filter)
23   doFilter:107, OncePerRequestFilter (org.springframework.web.filter)
22   internalDoFilter:193, ApplicationFilterChain (org.apache.catalina.core)
21   doFilter:166, ApplicationFilterChain (org.apache.catalina.core)
20   doFilterInternal:200, CharacterEncodingFilter (org.springframework.web.filter)
19   doFilter:107, OncePerRequestFilter (org.springframework.web.filter)
18   internalDoFilter:193, ApplicationFilterChain (org.apache.catalina.core)
17   doFilter:166, ApplicationFilterChain (org.apache.catalina.core)
16   invoke:200, StandardWrapperValve (org.apache.catalina.core)
15   invoke:96, StandardContextValve (org.apache.catalina.core)
14   invoke:490, AuthenticatorBase (org.apache.catalina.authenticator)
13   invoke:139, StandardHostValve (org.apache.catalina.core)
12   invoke:92, ErrorReportValve (org.apache.catalina.valves)
11   invoke:74, StandardEngineValve (org.apache.catalina.core)
10   service:343, CoyoteAdapter (org.apache.catalina.connector)
9   service:408, Http11Processor (org.apache.coyote.http11)
8   process:66, AbstractProcessorLight (org.apache.coyote)
7   process:834, AbstractProtocol$ConnectionHandler (org.apache.coyote)
6   doRun:1415, NioEndpoint$SocketProcessor (org.apache.tomcat.util.net)
5   run:49, SocketProcessorBase (org.apache.tomcat.util.net)
4   runWorker:1149, ThreadPoolExecutor (java.util.concurrent)
3   run:624, ThreadPoolExecutor$Worker (java.util.concurrent)
2   run:61, TaskThread$WrappingRunnable (org.apache.tomcat.util.threads)
1   run:748, Thread (java.lang)
```

简单分析一下，这里面的流程分2个部分，一部分是tomcat的：org.apache.catalina；一部分是spring的：org.springframework。

相信大家都知道，所有的web流程都是基于filter 和 servlet 的。

可以看到tomcat的 ApplicationFilterChain 扮演了过滤链的角色，由它来决定filter链的执行。当所有filter执行完毕后开始执行servlet，也就是spring的 FrameworkServlet （39行）。然后到了大家众所周知的 DispatcherServlet （44行）执行 doDispatch 方法 

```java
// Actually invoke the handler.
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
```

从这开始真正的调用handler来处理请求。mv就是ModelAndView。这里先讲2个概念，Handler和HandlerAdapter。

Handler 类型 是一个Object ，就是这么简单粗暴，它是为了保证扩展性，顾名思义它就是一个处理者的抽象，用来处理各种请求。HandlerAdapter 用来判断当前请求应该选哪一个Handler。众所周知的 RequestMappingHandlerAdapter 对应的 Handler 是 HandlerMethod。HandlerMethod 顾名思义就是对应一个方法的处理者。

回到调用栈上就是第47行，RequestMappingHandlerAdapter.invokeHandlerMethod()，开始调用对应的HandlerMethod，也就是 ServletInvocableHandlerMethod (48行)，接着到了父类 InvocableHandlerMethod 调用 invokeForRequest() 开始真正处理请求。（讲了这么多终于到正题了）

invokeForRequest的源码：

```java
public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {
 
		Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
		if (logger.isTraceEnabled()) {
			logger.trace("Arguments: " + Arrays.toString(args));
		}
		return doInvoke(args);
	}
```
我们终于找到了解析方法参数的入口，**getMethodArgumentValues** 

里面的逻辑就大同小异了，找一个MethodArgumentsResolver 进行参数的解析。这里找到的是 ServletModelAttributeMethodProcessor ，我们再看一下它支持的参数类型

```java
@Override
	public boolean supportsParameter(MethodParameter parameter) {
		return (parameter.hasParameterAnnotation(ModelAttribute.class) ||
				(this.annotationNotRequired && !BeanUtils.isSimpleProperty(parameter.getParameterType())));
	}
```

意思是要么是带 `@ModelAttribute`  注解的 或者 不是简单类型的，比如String、Date或者Class对象、int类型等。

接着看它的 resolveArgument 方法。首先它创建了一个WebDataBinder （Special DataBinder for data binding from web request parameters to JavaBean objects. ）即从request参数到JavaBean的绑定器。接下来的流程就很简单了，拿到请求参数反射调用对应字段的set方法完成对象构造。所以这种方式总是会返回一个对象的，只是对象里有没有值而已。

下面对这部分开始测试的几种情况做下说明：

1. GET +  查询参数

   > 对应的是RequestParamMethodArgumentResolver。本质就是Request.getParamValue(name)。所以直接取查询参数即可。

2. ContentType(multipart/form-data)

   > 对于这种ContentType类型，tomcat直接取的是请求体中的内容，所以GET、POST方式都有效。

3. POST + body + ContentType(application/x-www-form-urlencoded)

   >  tomcat只支持这种类型的POST方式。GET方式返回null。

4. 没有提供对json类型的解析





 

# 直接传简单类型的参数，如String、Date等

 String类型的直接传就好了，这里主要说下Date类型。有以下方式可以完成日期字符串参数到Date类型的转换。

-  @InitBinder

  ```java
  @InitBinder
      public void registerCustomDateEditor(WebDataBinder binder) {
          binder.registerCustomEditor(Date.class,new CustomDateEditor(new SimpleDateFormat("yyyy-MM-dd"),false));
      }
  ```

  `CustomDateEditor`  是spring 提供的一个DateEditor。

  

- ConversionService

- 万能解决方案，使用时间戳Timestamp

​        



# 总结

@RequestBody 是取请求体中的内容

@RequestParam 或者 不带任何注解是直接取请求参数中的内容

解析方法参数的接口是 MethodArgumentResolver，参数类型转换用的是MessageConverter 和 WebDataBinder 。