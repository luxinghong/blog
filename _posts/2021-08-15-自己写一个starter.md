---
layout: post
title: "自己写一个starter"
subtitle: ""
author: ""
header-style: text
tags:
  - springboot
  -	starter
---

一个 starter 就是一个提供特定功能的库，它主要做了几件事：

- 添加了一些依赖
- 定义了一些属性（在application.properties 或 application.yml 中使用），可以理解为一些功能的开关，或者是初始值
- 根据某些条件动态的定义一些bean，这些 bean 就是提供功能的对象，给客户端使用



目标：实现一个starter，提供 json 序列化功能，有两种序列化方式可供选择：fastjson 和 gson

使用场景1： 由用户在属性文件中选择使用哪一种

使用场景2： 由用户添加某个具体实现的依赖来选择使用哪一种





### 使用场景1： 由用户在属性文件中选择使用哪一种

1. 新建一个maven项目，命名为 json-spring-boot-starter（所有第三方库都命名为xxx-spring-boot-starter，spring 自己的 starter 命名为 spring-boot-starter-xxx）

   pom 文件如下，重点是引入的依赖：spring-boot-autoconfigure、fastjson、gson

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
       <modelVersion>4.0.0</modelVersion>
       <groupId>com.example</groupId>
       <artifactId>json-spring-boot-starter</artifactId>
       <version>0.0.1-SNAPSHOT</version>
       <name>json-spring-boot-starter</name>
       <description>Json Serializer Starter</description>
   
       <properties>
           <java.version>11</java.version>
       </properties>
   
       <dependencyManagement>
           <dependencies>
               <dependency>
                   <groupId>org.springframework.boot</groupId>
                   <artifactId>spring-boot-dependencies</artifactId>
                   <version>2.5.2</version>
                   <type>pom</type>
                   <scope>import</scope>
               </dependency>
           </dependencies>
       </dependencyManagement>
   
       <dependencies>
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-autoconfigure</artifactId>
           </dependency>
   
           <dependency>
               <groupId>com.alibaba</groupId>
               <artifactId>fastjson</artifactId>
               <version>1.2.76</version>
           </dependency>
           <dependency>
               <groupId>com.google.code.gson</groupId>
               <artifactId>gson</artifactId>
           </dependency>
   
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-test</artifactId>
               <scope>test</scope>
           </dependency>
       </dependencies>
   
       <build>
           <plugins>
               <plugin>
                   <groupId>org.apache.maven.plugins</groupId>
                   <artifactId>maven-compiler-plugin</artifactId>
                   <version>3.8.1</version>
                   <configuration>
                       <source>11</source>
                       <target>11</target>
                   </configuration>
               </plugin>
           </plugins>
       </build>
   </project>
   ```



2. 定义一个 JsonSerializer 接口，提供两种实现

   ```java
   public interface JsonSerializer {
       String serialize(Object obj);
   }
   ```

   ```java
   public class FastjsonSerializer implements JsonSerializer {
       @Override
       public String serialize(Object obj) {
           return "Fastjson: " + JSON.toJSONString(obj);
       }
   }
   ```

   ```java
   public class GsonSerializer implements JsonSerializer {
       @Override
       public String serialize(Object obj) {
           Gson gson = new Gson();
           return "Gson: " + gson.toJson(obj);
       }
   }
   ```

   ​	

3.  定义一个 JsonTemplate，这是最终用户要使用的工具类

   ```java
   public class JsonTemplate {
       private final JsonSerializer serializer;
   
       public JsonTemplate(JsonSerializer serializer) {
           this.serializer = serializer;
       }
   
       public String serialize(Object obj) {
           return this.serializer.serialize(obj);
       }
   }
   ```

   ​	

4.  实现自动配置，这里根据属性来选择，属性名定义为 json.serializer.type，用到`@ConditionalOnProperty` 注解。没有指定属性时使用 fastjson（`matchIfMissing = true`）

   ```java
   @Configuration
   public class JsonSerializerAutoConfiguration {
       @Bean
       public JsonTemplate jsonTemplate(JsonSerializer serializer) {
           return new JsonTemplate(serializer);
       }
   
       @Bean
       @ConditionalOnProperty(
               prefix = "json.serializer",
               name = "type",
               havingValue = "fastjson",
               matchIfMissing = true)
       public JsonSerializer fastjsonSerializer() {
           return new FastjsonSerializer();
       }
   
       @Bean
       @ConditionalOnProperty(prefix = "json.serializer", name = "type", havingValue = "gson")
       public JsonSerializer gsonSerializer() {
           return new GsonSerializer();
       }
   }
   ```

   ​	

   5. 在 `resources` 下创建一个文件 `META-INF/spring.factories`

   ```tex
   org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
   com.example.json.JsonSerializerAutoConfiguration
   ```

mvn install 到本地，接下来写客户端验证一下

```xml
		<dependency>
            <groupId>com.example</groupId>
            <artifactId>json-spring-boot-starter</artifactId>
            <version>0.0.1-SNAPSHOT</version>
        </dependency>
```

```java
@SpringBootApplication
@Slf4j
public class StarterClientApplication implements CommandLineRunner {

    @Autowired private JsonTemplate jsonTemplate;

    public static void main(String[] args) {
        SpringApplication.run(StarterClientApplication.class, args);
    }

    @Override
    public void run(String... args) throws Exception {
        Person person = new Person();
        person.setName("张三");
        person.setAge(30);

        log.info(jsonTemplate.serialize(person));
    }
}
```

没有指定属性时，输出为：

```java
Fastjson: {"age":30,"name":"张三"}
```

指定属性为json时：

```xml
json:
  serializer:
    type: gson
```

```java
Gson: {"name":"张三","age":30}
```



#### 为属性文件加上智能提示

1. 为属性创建一个类，打上 `@ConfigurationProperties` 注解

   ```java
   @ConfigurationProperties(prefix = "json.serializer")
   public class JsonSerializerProperties {
   
       /** 序列化类型：fastjson 和 gson */
       private Type type;
   
       public Type getType() {
           return type;
       }
   
       public void setType(Type type) {
           this.type = type;
       }
   
       public enum Type {
           FASTJSON,
           GSON
       }
   }
   ```

2. 改造自动配置类

   ```java
   @Configuration
   @EnableConfigurationProperties(JsonSerializerProperties.class)
   public class JsonSerializerAutoConfiguration {
       @Bean
       public JsonTemplate jsonTemplate(JsonSerializerProperties serializerProperties) {
           JsonSerializerProperties.Type type = serializerProperties.getType();
           JsonSerializer serializer =
                   (type == null || JsonSerializerProperties.Type.FASTJSON.equals(type))
                           ? new FastjsonSerializer()
                           : new GsonSerializer();
           return new JsonTemplate(serializer);
       }
   }
   ```

3. 加入`spring-boot-configuration-processor`依赖，注意一定不要搞错了，之前我没注意写成 `spring-boot-autoconfigure-processor`，害我调半天

   ```xml
   		<dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-configuration-processor</artifactId>
               <optional>true</optional>
           </dependency>
   ```

   这个注解会在 META-INF 下生成一个元数据文件 `spring-configuration-metadata.json`

   ```json
   {
     "groups": [
       {
         "name": "json.serializer",
         "type": "com.example.json.JsonSerializerProperties",
         "sourceType": "com.example.json.JsonSerializerProperties"
       }
     ],
     "properties": [
       {
         "name": "json.serializer.type",
         "type": "com.example.json.JsonSerializerProperties$Type",
         "description": "序列化类型：fastjson 和 gson",
         "sourceType": "com.example.json.JsonSerializerProperties"
       }
     ],
     "hints": []
   }
   ```

   description 来自字段上的注释

   ```java
   /** 序列化类型：fastjson 和 gson */
       private Type type;
   ```

   可以手动编辑该json文件，加上 `defaultValue`



搞定，看看客户端使用效果

![](/home/head/图片/Screenshot_20210815_014235.png)

![](/home/head/图片/Screenshot_20210815_014550.png)









### 使用场景2：由用户添加某个具体实现的依赖来选择使用哪一种

1. 改造自动配置类，用到 `@ConditionalOnClass`注解。注意这里bean定义的顺序，JsonTemplate一定要放在最后，因为它依赖于JsonSerializer

   ```java
   @Configuration
   @EnableConfigurationProperties(JsonSerializerProperties.class)
   public class JsonSerializerAutoConfiguration {
   
       @Bean
       @ConditionalOnClass(JSON.class)
       @Primary
       public JsonSerializer fastjsonSerializer() {
           return new FastjsonSerializer();
       }
   
       @Bean
       @ConditionalOnClass(Gson.class)
       public JsonSerializer gsonSerializer() {
           return new GsonSerializer();
       }
   
       @Bean
       public JsonTemplate jsonTemplate(JsonSerializer serializer) {
           return new JsonTemplate(serializer);
       }
   }
   ```

2. 将 fastjson 和 gson 依赖设置成可选项

   ```xml
   		<dependency>
               <groupId>com.alibaba</groupId>
               <artifactId>fastjson</artifactId>
               <version>1.2.76</version>
               <optional>true</optional>
           </dependency>
           <dependency>
               <groupId>com.google.code.gson</groupId>
               <artifactId>gson</artifactId>
               <optional>true</optional>
           </dependency>
   ```



测试一把，客户端先添加fastjson的依赖

```xml
		<dependency>
            <groupId>com.example</groupId>
            <artifactId>json-spring-boot-starter</artifactId>
            <version>0.0.1-SNAPSHOT</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.76</version>
        </dependency>
```

输出为：

```java
Fastjson: {"age":30,"name":"张三"}
```

换成gson

```xml
		<dependency>
            <groupId>com.example</groupId>
            <artifactId>json-spring-boot-starter</artifactId>
            <version>0.0.1-SNAPSHOT</version>
        </dependency>
        <dependency>
            <groupId>com.google.code.gson</groupId>
            <artifactId>gson</artifactId>
        </dependency>
```

输出为：

```java
Gson: {"name":"张三","age":30}
```



如果两个依赖都没有添加，会报错提示：

```java
Action:

Consider defining a bean of type 'com.example.json.JsonSerializer' in your configuration.
```

这个信息也可以自定义成更友好的提示信息，这一part暂时不研究了，结束！