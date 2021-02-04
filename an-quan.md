# Spark安全性（Spark Security）

[文档链接](http://spark.apache.org/docs/2.2.0/security.html)

Spark当前支持通过共享密钥（shared secret）进行认证。可以通过配置参数_spark.authenticate_开启认证。这个参数控制Spark的通信协议是否适用共享密钥。这个认证是一个基本的握手，以保证双方有相同的共享密钥，允许进行通信。如果共享密钥不一致，将不允许通信。共享密钥的生成如下：

对于YARN上部署的Spark，配置spark.authenticate为true会自动处理生成河分发共享密钥。每个应用将会使用一个唯一的共享密钥。

对一其它方式的Spark部署，应该在每个节点上配置Spark的参数_spark.authenticate.secret_。这个密钥由所有的Master/Workers和应用使用。

## 1、Web UI的加密

Spark UI可以通过spark.ui.filters设置来使用Java servelt过滤器（javax servlet filters）加密，也可以通过[SSL设置](http://spark.apache.org/docs/2.2.0/security.html#ssl-configuration)使用https/SSL加密。

### 1.1、认证Authentication

一个用户如果有不该被其他用户看到的数据，就可能需要对UI加密。

TO BE CONTINUED!!!

