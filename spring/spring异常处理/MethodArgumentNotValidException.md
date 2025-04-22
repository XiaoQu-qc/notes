
### 1验证触发时机：
表单提交：在处理用户提交的表单时，通常会在控制器方法中对请求参数或请求体进行验证。
API 请求参数验证：当使用 Spring MVC 或其他框架处理 HTTP 请求时，可以在控制器方法的参数上应用这些注解，验证请求参数或请求体中的数据。
### 2验证失败时：
如果验证失败，Spring 框架会生成一个 MethodArgumentNotValidException 异常并抛出。
在这个异常中，包含了每个字段的验证错误信息，包括 @NotBlank 和 @Size 注解中指定的 message 内容。
### 3错误消息显示：
默认行为：如果没有自定义异常处理逻辑，默认会返回一个 HTTP 400（Bad Request）响应，响应体中包含错误信息。这些消息通常是标准格式，不一定直接显示注解中的 message。
自定义异常处理：在实际项目中，通常会使用自定义的全局异常处理器（如 @ControllerAdvice）来捕获这些异常，并将错误信息格式化为统一的响应格式返回给前端。这时，message 属性的内容会被用作具体的错误提示信息。
例如，在前面讨论的 ExceptionControllerAdvice 示例中，当捕获到 MethodArgumentNotValidException 时，会提取错误信息（包括 message 内容）并返回给前端。

```
public class PointDto {
    @NotBlank(message = "点位名称不能为空")
    @Size(max = 50, message = "点位名称请勿超过50个字")
    private String name;

    // 其他字段和方法
}
@PostMapping("/points")
public ResponseEntity<?> createPoint(@Valid @RequestBody PointDto pointDto) {
    // 处理逻辑
}
```

### 4.总结来说
当验证失败是，默认会返回一个 HTTP 400（Bad Request）响应，但一般定义一个类@ControllerAdvice（注明是异常处理器） ExceptionControllerAdvice，其中为每一个异常类定义对应的异常处理方法
