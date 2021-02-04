# Spark配置（Spark Configuration）

[文档链接](http://spark.apache.org/docs/2.2.0/configuration.html)

Spark系统配置的三个位置：

* **Spark properties**控制大多数的应用参数，可以通过SparkConf对象配置，或者通过Java系统属性配置。
* **Environment variables**用于设置每个机器的设置，例如，IP地址；通过每个节点的conf/spark-env.sh脚本配置。
* **Logging**通过lo4j.properties设置。



