Kubernetes Informer 是 Kubernetes 中的一种机制，用于监视 Kubernetes API Server 中资源对象的变化，并将这些变化通知给客户端。Informer 可以帮助开发者编写自定义控制器（Custom Controller），并在 Kubernetes 集群中实现自动化操作。

Informer 通过轮询 Kubernetes API Server 来获取资源对象的最新状态，并将其缓存到本地内存中。当资源对象发生变化时，Informer 会检测到这些变化，并将其通知给客户端。这样，客户端就可以及时地响应资源对象的变化，从而实现对 Kubernetes 集群的实时监控和管理。

使用 Informer 可以大大简化开发者编写自定义控制器的工作量，同时提高控制器的性能和可靠性。Informer 还支持多种事件处理方式，例如添加、更新、删除等，可以根据不同的业务需求进行灵活配置。