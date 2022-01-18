---
layout: post
title: "Springboot中校验Enum"
subtitle: ""
author: ""
header-style: text
tags:
  - springboot
---



# 方式一：全局异常处理 + @JsonCreator

直接监听 `HttpMessageNotReadableException` 异常，在全局异常处理器中判断、整理异常信息，简单粗暴，如下所示：

```java
@ExceptionHandler(HttpMessageNotReadableException.class)
    public Result<String> handleHttpMessageNotReadableException(HttpMessageNotReadableException e) {
        String errorMsg;
        if (e.getCause() instanceof ValueInstantiationException) {
            ValueInstantiationException ex = ((ValueInstantiationException) e.getCause());
            Reference path = ex.getPath().get(0);
            Class<?> rawClass = ex.getType().getRawClass();
            String rightValue = "";
            if (rawClass.isEnum()) {
                rightValue = "值应为：" + Arrays.toString(rawClass.getEnumConstants());
            }
            String errorValue = ex.getOriginalMessage().split("problem: ")[1];
            errorMsg = String.format("%s参数(%s) 错误！ %s", path.getFieldName(), errorValue.substring(errorValue.lastIndexOf(".") + 1), rightValue);
        } else {
            errorMsg = e.getCause().getLocalizedMessage();
        }

        return new Result<>(1, errorMsg, "");
  }
```



返回结果类似如下：

```json
{
    "code": 1,
    "message": "searchScope参数(PRIVAT) 错误！ 值应为：[ALL, PLATFORM, PRIVATE]",
    "content": ""
}
```



结合 @JsonCreator 可忽略大小写，空值转默认值等操作，如下所示：

```java
public enum SearchScopeQry {
    ALL("全部"), PLATFORM("平台"), PRIVATE("私有");
    private String scope;

    SearchScopeQry(String scope) {
        this.scope = scope;
    }

    @JsonCreator
    public static SearchScopeQry getSearchScope(String scope) {
        if (StringUtils.isEmpty(scope)) {
            return ALL;
        }
        return SearchScopeQry.valueOf(scope.toUpperCase());
    }
}
```

------

<br/><br/><br/>







# 方式二：自定义校验器

主要思路是自定一个枚举校验注解，结合 `@Constraint` 使用自定义的校验器进行校验，校验不通过会抛出 `MethodArgumentNotValidException` 异常，网上有很多资料，这里不再详细说明。