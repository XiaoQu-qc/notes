### 1
dao，pojo，mapper文件是一一对应的,dao负责pojo类的所有数据库操作（dao的接口中一个方法对应一个sql），一个mapper文件又对应一个dao
### 2为什么用 DTO？
当后端需要将数据响应给前端时，直接响应实体模型（如实体对象）是不合适的，因为实体对象可能包含 敏感字段 或 不必要的数据。
总之，DTO 可以提炼出前端所需的字段 (接口示例约定)，避免过度暴露数据
前端传给后端，后端传给前端封装的对象并不是pojo，而是对应的dto
比如pojo User 有字段有password,总不能直接用User类传递吧，不过后续的数据库操作可能还是要转成pojo，因为pojo才是和数据库表对应的
### 3.一次会话或者说请求就是一个线程
在这种情况下，可以使用ThreadLocal变量来隔离不同线程的值
### 4.静态资源
请求静态资源时要在Webmvcconfiguuration中配置静态资源映射，否则路由器会将静态资源url看成动态请求，从而走controller的路由匹配规则，进而找不到匹配得到接口方法
### 5.mapper和数据库表
实体类，mapper类，数据库表是一一对应的，比如Dish，DishMapper,数据库中也会有个表Dish，这样不仅符合逻辑，并且便于理解和管理，
DishMapper传递的参数通常也是Dish实例
