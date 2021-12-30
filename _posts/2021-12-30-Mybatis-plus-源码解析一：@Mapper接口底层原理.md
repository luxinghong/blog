---
layout: post
title: "Mybatis-plus 源码解析一：@Mapper接口底层原理"
subtitle: ""
author: ""
header-style: text
tags:
  - mybatis
---



通常我们都是这样使用的：

```java
@Mapper
public interface PersonMapper extends BaseMapper<Person> {
}
```

然后在代码中直接注入 `personMapper` 就可以使用了，这篇文章就来解析下这背后做了什么。



首先，毫无疑问从 `MybatisPlusAutoConfiguration` 看起，里面有一个 `MapperScannerRegistrarNotFoundConfiguration` 内部配置类，**当没有使用 `@MapperScan` 、只使用 @Mapper 时**，这个配置类会生效，它 import 了一个 `AutoConfiguredMapperScannerRegistrar` 类，这个类实现了 `ImportBeanDefinitionRegistrar` 接口，先从这个接口讲起。



# `ImportBeanDefinitionRegistrar`

作用：**手动注册 `beanDefinition`**

示例：

1. 首先，定义一个普通的POJO：

   ```java
   public class Person {
       private Integer id;
       private String name;
   
       public Integer getId() {
           return id;
       }
   
       public void setId(Integer id) {
           this.id = id;
       }
   
       public String getName() {
           return name;
       }
   
       public void setName(String name) {
           this.name = name;
       }
   
       @Override
       public String toString() {
           return "Person{" +
                   "id=" + id +
                   ", name='" + name + '\'' +
                   '}';
       }
   }
   ```

2. 写一个普通的类，实现 `ImportBeanDefinitionRegistrar` 接口

   ```java
   public class Config implements ImportBeanDefinitionRegistrar{
       @Override
       public void registerBeanDefinitions(
               AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
           RootBeanDefinition beanDefinition = new RootBeanDefinition();
           beanDefinition.setBeanClass(Person.class);
           MutablePropertyValues propertyValues = new MutablePropertyValues();
           propertyValues.addPropertyValue("id", 1);
           propertyValues.addPropertyValue("name", "xxx");
           beanDefinition.setPropertyValues(propertyValues);
           registry.registerBeanDefinition("person", beanDefinition);
       }
   }
   ```

   注册了一个名为 `person`，beanClass 为 Person.class，id=1，name=xxx 的 beanDefinition。

3. 在启动类上加上 `@Import(Config.class)`

4. 然后就可以直接拿到这个bean了

   ```java
   @Autowired
       private Person person;
   ```

可看到 Person 就是一个普通的POJO，没有打任何注解，但是已经被容器实例化为一个bean了。

所以 `ImportBeanDefinitionRegistrar` 的作用就是手动的把 bean 纳入spring容器的管理，**和 `@Import` 配合使用，更多被用来扫描自定义注解。**Spring框架内部用的非常多。

自定义注解扫描示例：

1. 自定义一个注解 

   ```java
   @Target(ElementType.TYPE)
   @Retention(RetentionPolicy.RUNTIME)
   public @interface MyComponent {
   }
   ```

2. 加到 Person 类上

3. 改一下配置类

   ```java
   @Override
       public void registerBeanDefinitions(
               AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
           ClassPathScanningCandidateComponentProvider scanner =
                   new ClassPathScanningCandidateComponentProvider(false);
           scanner.addIncludeFilter(new AnnotationTypeFilter(MyComponent.class));
           Set<BeanDefinition> candidateComponents =
                   scanner.findCandidateComponents("com.example.demo");
           for (BeanDefinition candidateComponent : candidateComponents) {
               registry.registerBeanDefinition(
                       candidateComponent.getBeanClassName(), candidateComponent);
           }
       }
   ```

   使用了一个 `ClassPathScanningCandidateComponentProvider` 类，从名字大致可看出是基于类路径的组件扫描，接着设置了它的 `includeFilter` 为我们的自定义注解 `MyComponent`，然后调用 `findCandidateComponents` 方法，传入一个包路径，便可得到一个 BeanDefinition 集合。最后把这些 beanDefinition 注册到容器中。

4. 最后直接使用 `@Autowired` 注入 person 即可。这样就完成了使用自定义注解向Spring容器注册bean的目的。当然，此时得到的 person 的属性都是空的，因为我们没有做任何属性的设置，这样的bean肯定是不能直接拿来使用的，接着往下看。

------

<br/><br/><br/>





# 未完待续！！！