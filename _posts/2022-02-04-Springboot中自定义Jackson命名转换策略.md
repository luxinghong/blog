---
layout: post
title: "Springboot中自定义Jackson命名转换策略"
subtitle: ""
author: ""
header-style: text
tags:
  - jackson
---

这篇文章来源于这么一个需求：前端传递过来的json格式不统一，有下划线格式的、驼峰格式的，需要都能正确反序列化。而序列化则统一采用驼峰格式。

如果反序列化和序列化都采用同一种格式，则直接可以使用内置的 `com.fasterxml.jackson.databind.PropertyNamingStrategy`，它有如下几种预定义的属性命名策略：

- `SNAKE_CASE`
- `UPPER_CAMEL_CASE`
- `LOWER_CAMEL_CASE`
- `LOWER_CASE`
- `KEBAB_CASE`
- `LOWER_DOT_CASE`

在 application.yml 中配置示例如下：

```yaml
spring.jackson.property-naming-strategy: SNAKE_CASE
```



但**由于需求中要求反序列化和序列化的格式不一致**，这时就需要定制一下了。

自定义一个类继承 `PropertyNamingStrategy`，重写 `nameForSetterMethod` 方法：

```java
public class CustomPropertyNamingStrategy extends PropertyNamingStrategy {
    @Override
    public String nameForSetterMethod(MapperConfig<?> config, AnnotatedMethod method, String defaultName) {
        // 驼峰转小写下划线
        return CaseFormat.LOWER_CAMEL.to(CaseFormat.LOWER_UNDERSCORE, defaultName);
    }
}
```

Jackson 反序列化逻辑是这样的：首先为目标类型（即 @RequestBody 后的POJO）生成一个 `BeanDeserializer`，反射获取其字段和对应的getter/setter方法封装成一个 `SettableBeanProperty`（这种情况下是它的子类 `MethodProperty`，当然还有其他类型的子类）。然后构造一个 map，以此为 value，以 `nameForSetterMethod` 返回的值为 key（其中 defaultName 为 POJO 中字段名），这样就把json中的字段名和POJO中的setter方法对应起来了，最终完成反序列化。

默认情况下没有配置自定义的命名策略时，map 中的 key 就是 defaultName，没作任何处理。

而我们的需求json是下划线的，POJO是驼峰的。因此在 `nameForSetterMethod ` 方法中将驼峰格式的defaultName转为了小写下划线格式的作为key。比如POJO中有一个属性为 `userId`，json对象中为 `user_id`，使用此自定义命名策略后，内部的map中就会存在这么一个映射：`"user_id" ->  MethodProperty(field:userId,getter:getUserId,setter:setUserId)`。在反序列化 `user_id` 字段时，就会反射调用POJO的 `setUserId` 方法将值设置进去。

除 `nameForSetterMethod` 外，还有一个 `nameForGetterMethod` 方法。同样的逻辑，`nameForGetterMethod` 是用于构造序列化时map的key的。key 为该方法返回的值，value为对应POJO的getter方法。不同的是，此时的key被用来作为序列化后json对象的字段名。默认也是不作任何处理，直接返回defaultName，即驼峰格式。我们的需求也是序列化返回驼峰格式，因此该方法不用重写。

上述情况都是有 getter/setter 方法的情况，在没有getter/setter时，命名策略用到了另一个方法：`nameForField`。此时Jackson序列化/反序列化时由于没有getter/setter 方法，会直接反射得到field来获取值或设置值，**当然前提是字段是public的**。`nameForField` 的返回值便作为了反序列化时json对象的字段名，和序列化时json对象的字段名。同样，在内部会维护一个map，key 是 `nameForField` 的返回值，value是 `FieldProperty`（SettableBeanProperty  的另一个子类）。

可以查看 `SNAKE_CASE` 对应的实现 `SnakeCaseStrategy`，它对上述三个方法都做了重写，全部将 defaultName 转为了小写下划线的格式。所以它的效果是反序列化和序列化都使用下换线的json格式。

最后，在要应用此命名转换策略的POJO类上加上注解 `@JsonNaming(CustomPropertyNamingStrategy.class)` 即可，它会覆盖全局的命名策略，优先级更高。