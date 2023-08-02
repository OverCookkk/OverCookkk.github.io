class Account{

​		方法1：转入

​	   方法2：转出

}





每个实体只能操作自己的属性，而跨实体属性变化必须通过领域服务调度，领域服务不能直接修改实体的状态，只能够调用实体的业务方法。

领域服务：专属当前转账







**DDD包含4个层：**

1. Domain：这是定义应用程序的领域和业务逻辑的地方
2. Infrastructure：此层包含独立于我们的应用程序而存在的所有内容：外部库，数据库引擎等。
3. Application：该层用作领域和界面层之间的通道。将请求从接口层发送到域层，由领域层处理请求并返回响应。
4. Interface：该层包含与其他系统交互的所有内容，例如Web服务，RMI接口或Web应用程序以及批处理前端。



以著名的案例: [https://github.com/victorsteven/food-app-server](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fvictorsteven%2Ffood-app-server)为例子

```text
interfaces git:(master) tree
.
|____fileupload
| |____fileformat.go
| |____fileupload.go
|____food_handler.go
|____food_handler_test.go
|____handler_setup_test.go
|____login_handler.go
|____login_handler_test.go
|____middleware
| |____middleware.go
|____user_handler.go
```

interfaces层定义了输入层的相关方法，以使用gin提供http接口为例，这里的handler等为使用gin提供的一些http接口（相当于MVC三层中的controller层），这一层会调用application层。



```text
application git:(master) tree
.
|____food_app.go
|____food_app_test.go
|____user_app.go
|____user_app_test.go
```

application层主要是调用domain层与infrastructure层来实现功能



```text
domain git:(master) tree
.
|____entity
| |____food.go
| |____user.go
|____repository
| |____food_repository.go
| |____user_repository.go
```

domain层主要是定义了entity，以及repository接口；entity里头会包含一些领域逻辑





```text
infrastructure git:(master) tree
.
|____auth
| |____auth.go
| |____redisdb.go
| |____token.go
|____persistence
| |____db.go
| |____food_repository.go
| |____food_repository_test.go
| |____setup_test.go
| |____user_repository.go
| |____user_repository_test.go
|____security
| |____password.go
```

infrastructure层这里提供了针对domain层的repository接口的实现，还有其他一些基础的组件，提供给application层或者interfaces层使用



小结
DDD一般分为interfaces、application、domain、infrastructure这几层；其中domain层不依赖其他层，它定义repository接口，infrastructure层会实现；application层会调用domain、infrastructure层；interfaces层一般调用application层或者infrastructure层。
