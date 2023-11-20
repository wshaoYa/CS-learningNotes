# 传统MVC模式

MVC 模式代表 Model-View-Controller（模型-视图-控制器） 模式。这种模式用于应用程序的分层开发。

- **Model（模型）** - 模型代表一个存取数据的对象或 JAVA POJO。它也可以带有逻辑，在数据变化时更新控制器。
- **View（视图）** - 视图代表模型包含的数据的可视化。
- **Controller（控制器）** - 控制器作用于模型和视图上。它控制数据流向模型对象，并在数据变化时更新视图。它使视图与模型分离开。

![img](https://www.runoob.com/wp-content/uploads/2014/08/1200px-ModelViewControllerDiagram2.svg_.png)

但前后端分离模式下，MVC逐渐不再适用流行

# 新兴CLD模式

分为以下几大层

- 协议处理层:  支持各种协议
- Controller:  服务的入口，负责处理路由、参数校验、请求转发
- Logic/Service:  逻辑(服务)层，负责处理业务逻辑。
- DAO/Repository:  负责数据与存储相关功能。

![image-20231113220940000](https://s2.loli.net/2023/11/13/V1KGi5esuSmCIl4.png)