---
title: Spring Boot 构建 RESTful API 最佳实践
date: 2026-04-22 14:00:00
tags: [Java, Spring Boot, RESTful]
categories: 后端
description: 使用 Spring Boot 构建规范、优雅的 RESTful API，涵盖统一响应、异常处理、参数校验等
top_img:
cover:
---

## 前言

RESTful API 是目前最流行的 Web API 设计风格。本文介绍如何使用 Spring Boot 构建规范、优雅的 RESTful API，涵盖统一响应格式、全局异常处理、参数校验等最佳实践。

## 统一响应格式

定义统一的响应结构，让所有接口返回一致的格式。

```java
@Data
public class Result<T> {
    private Integer code;
    private String message;
    private T data;

    public static <T> Result<T> success(T data) {
        Result<T> result = new Result<>();
        result.setCode(200);
        result.setMessage("操作成功");
        result.setData(data);
        return result;
    }

    public static <T> Result<T> error(Integer code, String message) {
        Result<T> result = new Result<>();
        result.setCode(code);
        result.setMessage(message);
        return result;
    }
}
```

## 全局异常处理

使用 `@RestControllerAdvice` 统一捕获异常，避免在每个 Controller 中重复处理。

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    public Result<Void> handleBusiness(BusinessException e) {
        return Result.error(e.getCode(), e.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result<Void> handleValidation(MethodArgumentNotValidException e) {
        String message = e.getBindingResult()
                .getFieldErrors()
                .stream()
                .map(FieldError::getDefaultMessage)
                .collect(Collectors.joining(", "));
        return Result.error(400, message);
    }
}
```

## 参数校验

使用 `spring-boot-starter-validation` 进行参数校验。

```java
@Data
public class UserDTO {
    @NotBlank(message = "用户名不能为空")
    @Length(min = 2, max = 20, message = "用户名长度需在2-20之间")
    private String username;

    @NotBlank(message = "密码不能为空")
    @Pattern(regexp = "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d).{8,}$",
             message = "密码至少8位，含大小写字母和数字")
    private String password;

    @Email(message = "邮箱格式不正确")
    private String email;
}
```

## Controller 层

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @Autowired
    private UserService userService;

    @GetMapping("/{id}")
    public Result<UserVO> getById(@PathVariable Long id) {
        return Result.success(userService.getById(id));
    }

    @PostMapping
    public Result<Void> create(@Valid @RequestBody UserDTO dto) {
        userService.create(dto);
        return Result.success(null);
    }

    @PutMapping("/{id}")
    public Result<Void> update(@PathVariable Long id,
                               @Valid @RequestBody UserDTO dto) {
        userService.update(id, dto);
        return Result.success(null);
    }

    @DeleteMapping("/{id}")
    public Result<Void> delete(@PathVariable Long id) {
        userService.delete(id);
        return Result.success(null);
    }
}
```

## RESTful 设计规范

| HTTP 方法 | 语义 | 示例 |
|-----------|------|------|
| GET | 查询资源 | `GET /api/users/1` |
| POST | 新增资源 | `POST /api/users` |
| PUT | 更新资源（整体） | `PUT /api/users/1` |
| PATCH | 更新资源（部分） | `PATCH /api/users/1` |
| DELETE | 删除资源 | `DELETE /api/users/1` |

### URL 设计原则

- 使用名词复数：`/api/users` 而非 `/api/user`
- 层级表达关系：`/api/users/1/orders`
- 过滤用查询参数：`/api/users?status=active&page=1`
- 版本号放路径中：`/api/v1/users`

## 总结

构建好的 RESTful API 需要注意：统一响应格式、全局异常处理、参数校验、规范的 URL 设计。这些实践能让 API 更健壮、更易维护、更便于前端对接。