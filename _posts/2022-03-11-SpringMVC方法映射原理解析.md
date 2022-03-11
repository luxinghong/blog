---
layout: post
title: "SpringMVC方法映射原理解析"
subtitle: ""
author: ""
header-style: text
tags:
  - Spring
---



# HandlerMapping接口

package：spring-webmvc。

顶级接口。

用于定义请求和处理对象之间的映射关系。

主要方法：

```java
HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception;
```

主要实现类：`HandlerMapping（接口）-> AbstractHandlerMapping(抽象类) -> AbstractHandlerMethodMapping(抽象类) ->...省略... -> RequestMappingHandlerMapping(实现类)`。





# `WebMvcAutoConfiguration`自动配置类

声明了一个 `RequestMappingHandlerMapping` bean

```java
@Bean
@Primary
@Override
public RequestMappingHandlerMapping requestMappingHandlerMapping(
      @Qualifier("mvcContentNegotiationManager") ContentNegotiationManager contentNegotiationManager,
      @Qualifier("mvcConversionService") FormattingConversionService conversionService,
      @Qualifier("mvcResourceUrlProvider") ResourceUrlProvider resourceUrlProvider) {
   // Must be @Primary for MvcUriComponentsBuilder to work
   return super.requestMappingHandlerMapping(contentNegotiationManager, conversionService,
         resourceUrlProvider);
}
```







# RequestMappingHandlerMapping

`RequestMappingHandlerMapping` 同时实现了 `InitializingBean` 接口，看它的 `afterPropertiesSet` 方法：

```java
@Override
@SuppressWarnings("deprecation")
public void afterPropertiesSet() {
   //省略，一些属性设置

   super.afterPropertiesSet();
}
```

看它的父类 `AbstractHandlerMethodMapping`

```java
@Override
public void afterPropertiesSet() {
   initHandlerMethods();
}

/**
 * Scan beans in the ApplicationContext, detect and register handler methods.
 * @see #getCandidateBeanNames()
 * @see #processCandidateBean
 * @see #handlerMethodsInitialized
 */
protected void initHandlerMethods() {
   for (String beanName : getCandidateBeanNames()) {
      if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {
         processCandidateBean(beanName);
      }
   }
   handlerMethodsInitialized(getHandlerMethods());
}
```

进 `processCandidateBean` 方法：

```java
protected void processCandidateBean(String beanName) {
   Class<?> beanType = null;
   try {
      beanType = obtainApplicationContext().getType(beanName);
   }
   catch (Throwable ex) {
      // An unresolvable bean type, probably from a lazy bean - let's ignore it.
      if (logger.isTraceEnabled()) {
         logger.trace("Could not resolve type for bean '" + beanName + "'", ex);
      }
   }
   if (beanType != null && isHandler(beanType)) {
      detectHandlerMethods(beanName);
   }
}
```

当 bean isHandler 时，检测该bean内的方法。

`isHandler` 就是看有没有打了 `@Controller` 或 `@RequestMapping`  注解：

```java
@Override
protected boolean isHandler(Class<?> beanType) {
   return (AnnotatedElementUtils.hasAnnotation(beanType, Controller.class) ||
         AnnotatedElementUtils.hasAnnotation(beanType, RequestMapping.class));
}
```

接着看 `detectHandlerMethods`：

```java
protected void detectHandlerMethods(Object handler) {
   Class<?> handlerType = (handler instanceof String ?
         obtainApplicationContext().getType((String) handler) : handler.getClass());

   if (handlerType != null) {
      Class<?> userType = ClassUtils.getUserClass(handlerType);
      Map<Method, T> methods = MethodIntrospector.selectMethods(userType,
            (MethodIntrospector.MetadataLookup<T>) method -> {
               try {
                  return getMappingForMethod(method, userType);
               }
               catch (Throwable ex) {
                  throw new IllegalStateException("Invalid mapping on handler class [" +
                        userType.getName() + "]: " + method, ex);
               }
            });
      
       //省略
       
       
      methods.forEach((method, mapping) -> {
         Method invocableMethod = AopUtils.selectInvocableMethod(method, userType);
         registerHandlerMethod(handler, invocableMethod, mapping);
      });
   }
}
```

`MethodIntrospector.selectMethods` 会根据第二个参数返回的metadata(这里是RequestMappingInfo)，当metadata不为null时，将该method加入到map<Method,RequestMappingInfo>中返回。`getMappingForMethod` 方法的主要逻辑就是拿到method上RequestMapping注解的信息：

```java
private RequestMappingInfo createRequestMappingInfo(AnnotatedElement element) {
   RequestMapping requestMapping = AnnotatedElementUtils.findMergedAnnotation(element, RequestMapping.class);
   RequestCondition<?> condition = (element instanceof Class ?
         getCustomTypeCondition((Class<?>) element) : getCustomMethodCondition((Method) element));
   return (requestMapping != null ? createRequestMappingInfo(requestMapping, condition) : null);
}
```

所以最终只会返回打了RequestMapping注解的方法，因为类如`GetMapping`类的注解其实也是加了@RequestMapping的复合注解。

接下来就进入到了 `registerHandlerMethod`，注册mapping关系：

```java
protected void registerHandlerMethod(Object handler, Method method, T mapping) {
   this.mappingRegistry.register(mapping, handler, method);
}
```

```java
public void register(T mapping, Object handler, Method method) {
   // 省略
      this.registry.put(mapping,
            new MappingRegistration<>(mapping, handlerMethod, directPaths, name, corsConfig != null));
   // 省略
}
```

registry 就是一个map：

```java
private final Map<T, MappingRegistration<T>> registry = new HashMap<>();
```

至此，在 RequestMappingHandlerMapping 中维护了一个RequestMappingInfo和Method关系的map。







# 请求匹配

springmvc本质就是一个servlet，直接看DIspatcherServlet的doDispatch方法：

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
   // 省略

         // Determine handler for the current request.
         mappedHandler = getHandler(processedRequest);
         
   // 省略
}
```

```java
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
   if (this.handlerMappings != null) {
       // 熟悉的 HandlerMapping 接口
      for (HandlerMapping mapping : this.handlerMappings) {
         HandlerExecutionChain handler = mapping.getHandler(request);
         if (handler != null) {
            return handler;
         }
      }
   }
   return null;
}
```

`mapping.getHandler(request)` 调用链为：

`mapping.getHandler(request) -> AbstractHandlerMaping#getHandler -> AbstractHandlerMethodMapping#getHandlerInternal -> AbstractHandlerMethodMapping#lookupHandlerMethod`。

`lookupHandlerMethod` :

```java
protected HandlerMethod lookupHandlerMethod(String lookupPath, HttpServletRequest request) throws Exception {
   // 省略 
   List<T> directPathMatches = this.mappingRegistry.getMappingsByDirectPath(lookupPath);
   // 省略
}
```

又看到熟悉的 `mappingRegistry` 了。