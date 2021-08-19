### Structured Streaming

#### ForeachWriter

`ForeachWriter` 中，在 `open` 方法中创建了 redis 连接池，但是 redis 连接池在 `close` 方法中总是（不能调用连接池的 close 方法，也不能做其他使用）报错： `redis.clients.jedis.exceptions.JedisConnectionException: no reachable node in cluster`， 难道是 `close` 之前关闭了连接池吗？

#### checkpoint

一个 executor 时的 `checkpoint` 在改了 executor 数量之后再次执行，则会报错——checkpoint 中文件找不到之类的。难道 checkpoint 也记录了 executor 信息？