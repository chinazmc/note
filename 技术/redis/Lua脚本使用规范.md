
云数据库Redis实例支持Lua相关命令，通过Lua脚本可高效地处理CAS（check-and-set）命令，进一步提升Redis的性能，同时可以轻松实现以前较难实现或者不能高效实现的模式。本文介绍通过Redis使用Lua脚本的基本语法与使用规范。

## 注意事项

[数据管理服务DMS](https://help.aliyun.com/document_detail/47550.htm)控制台目前暂不支持使用Lua脚本等相关命令，请通过客户端或Redis-cli连接Redis实例使用Lua脚本。

## 基本语法

命令

语法

说明

EVAL

`EVAL script numkeys [key [key ...]] [arg [arg ...]]`

执行给定的脚本和参数，并返回结果。

参数说明：

-   script：Lua脚本。
-   numkeys：指定KEYS[]参数的数量，非负整数。
-   KEYS[]：传入的Redis键参数。
-   ARGV[]：传入的脚本参数。KEYS[]与ARGV[]的索引均从1开始。

**说明**

-   与SCRIPT LOAD命令一样，EVAL命令也会将Lua脚本缓存至Redis。
-   混用或滥用KEYS[]与ARGV[]可能会导致Redis产生不符合预期的行为，尤其在集群模式下，详情请参见[集群中Lua脚本的限制](https://help.aliyun.com/document_detail/92942.html#section-8f7-qgv-dlv)。

EVALSHA

`EVALSHA sha1 numkeys key [key ...] arg [arg ...]`

给定脚本的SHA1校验和，Redis将再次执行脚本。

使用EVALSHA命令时，若sha1值对应的脚本未缓存至Redis中，Redis会返回NOSCRIPT错误，请通过EVAL或SCRIPT LOAD命令将目标脚本缓存至Redis中后进行重试，详情请参见[处理NOSCRIPT错误](https://help.aliyun.com/document_detail/92942.html#section-82q-id4-ycg)。

SCRIPT LOAD

`SCRIPT LOAD script`

将给定的script脚本缓存在Redis中，并返回该脚本的SHA1校验和。

SCRIPT EXISTS

`SCRIPT EXISTS script [script ...]`

给定一个（或多个）脚本的SHA1，返回每个SHA1对应的脚本是否已缓存在当前Redis服务。脚本已存在则返回1，不存在则返回0。

SCRIPT KILL

`SCRIPT KILL`

停止正在运行的Lua脚本。

SCRIPT FLUSH

`SCRIPT FLUSH`

清空当前Redis服务器中的所有Lua脚本缓存。

更多关于Redis命令的介绍，请参见[Redis官网](http://www.redis.cn/commands.html)。

以下为部分命令的示例，本文在执行以下命令前执行了`SET foo value_test`。

-   EVAL命令示例：
    
    ```javascript
    EVAL "return redis.call('GET', KEYS[1])" 1 foo
    ```
    
    返回示例：
    
    ```javascript
    "value_test"
    ```
    
-   SCRIPT LOAD命令示例：
    
    ```javascript
    SCRIPT LOAD "return redis.call('GET', KEYS[1])"
    ```
    
    返回示例：
    
    ```javascript
    "620cd258c2c9c88c9d10db67812ccf663d96bdc6"
    ```
    
-   EVALSHA命令示例：
    
    ```undefined
    EVALSHA 620cd258c2c9c88c9d10db67812ccf663d96bdc6 1 foo
    ```
    
    返回示例：
    
    ```javascript
    "value_test"
    ```
    
-   SCRIPT EXISTS命令示例：
    
    ```sql
    SCRIPT EXISTS 620cd258c2c9c88c9d10db67812ccf663d96bdc6 ffffffffffffffffffffffffffffffffffffffff
    ```
    
    返回示例：
    
    ```sql
    1) (integer) 1
    2) (integer) 0
    ```
    

## 优化内存、网络开销

现象：

在Redis中缓存了大量功能重复的脚本，占用大量内存空间甚至引发内存溢出（Out of Memory），错误示例如下。

```javascript
EVAL "return redis.call('set', 'k1', 'v1')" 0
EVAL "return redis.call('set', 'k2', 'v2')" 0
```

解决方案：

-   请避免将参数作为常量写在Lua脚本中，以减少内存空间的浪费。
    
    ```python
    # 与错误示例实现相同功能但仅需缓存一次脚本。
    EVAL "return redis.call('set', KEYS[1], ARGV[1])" 1 k1 v1
    EVAL "return redis.call('set', KEYS[1], ARGV[1])" 1 k2 v2
    ```
    
-   更加建议采用如下写法，在减少内存的同时，降低网络开销。
    
    ```python
    SCRIPT LOAD "return redis.call('set', KEYS[1], ARGV[1])"    # 执行后，Redis将返回"55b22c0d0cedf3866879ce7c854970626dcef0c3"
    EVALSHA 55b22c0d0cedf3866879ce7c854970626dcef0c3 1 k1 v1
    EVALSHA 55b22c0d0cedf3866879ce7c854970626dcef0c3 1 k2 v2
    ```
    

## 清理Lua脚本的内存占用

现象：

由于Lua脚本缓存将计入Redis的内存使用量中，并会导致used_memory升高，当Redis的内存使用量接近甚至超过maxmemory时，可能引发内存溢出（Out Of Memory），报错示例如下。

```sql
-OOM command not allowed when used memory > 'maxmemory'.
```

解决方案：

通过客户端执行SCRIPT FLUSH命令清除Lua脚本缓存，但与FLUSHALL不同，SCRIPT FLUSH命令为同步操作。若Redis缓存的Lua脚本过多，SCRIPT FLUSH命令会阻塞Redis较长时间，可能导致实例不可用，请谨慎处理，建议在业务低峰期执行该操作。

**说明** 在控制台上单击清除数据只能清除数据，无法清除Lua脚本缓存。

同时，请避免编写过大的Lua脚本，防止占用过多的内存；避免在Lua脚本中大批量写入数据，否则会导致内存使用急剧升高，甚至造成实例OOM。在业务允许的情况下，建议开启[数据逐出](https://help.aliyun.com/document_detail/38679.htm)（云数据库Redis默认开启，模式为volatile-lru）节省内存空间。但无论是否开启数据逐出，Redis均不会逐出Lua脚本缓存。

## 处理NOSCRIPT错误

现象：

使用EVALSHA命令时，若sha1值对应的脚本未缓存至Redis中，Redis会返回NOSCRIPT错误，报错示例如下。

```go
(error) NOSCRIPT No matching script. Please use EVAL.
```

解决方案：

请通过EVAL命令或SCRIPT LOAD命令将目标脚本缓存至Redis中后进行重试。但由于Redis不保证Lua脚本的持久化、复制能力，Redis在部分场景下仍会清除Lua脚本缓存（例如实例迁移、变配等），这要求您的客户端需具备处理该错误的能力，详情请参见[脚本缓存、持久化与复制](https://help.aliyun.com/document_detail/92942.html#section-ot9-rbm-ofj)。

以下为一种处理NOSCRIPT错误的Python Demo示例，该demo利用Lua脚本实现了字符串prepend操作。

**说明** 您可以考虑通过Python的redis-py解决该类错误，redis-py提供了封装Redis Lua的一些底层逻辑判断（例如NOSCRIPT错误的catch）的Script类。

```python
import redis
import hashlib

# strin是一个Lua脚本的字符串，函数以字符串的格式返回strin的sha1值。
def calcSha1(strin):
    sha1_obj = hashlib.sha1()
    sha1_obj.update(strin.encode('utf-8'))
    sha1_val = sha1_obj.hexdigest()
    return sha1_val

class MyRedis(redis.Redis):

    def __init__(self, host="localhost", port=6379, password=None, decode_responses=False):
        redis.Redis.__init__(self, host=host, port=port, password=password, decode_responses=decode_responses)

    def prepend_inLua(self, key, value):
        script_content = """\
        local suffix = redis.call("get", KEYS[1])
        local prefix = ARGV[1]
        local new_value = prefix..suffix
        return redis.call("set", KEYS[1], new_value)
        """
        script_sha1 = calcSha1(script_content)
        if self.script_exists(script_sha1)[0] == True:      # 检查Redis是否已缓存该脚本。
            return self.evalsha(script_sha1, 1, key, value) # 如果已缓存，则用EVALSHA执行脚本
        else:
            return self.eval(script_content, 1, key, value) # 否则用EVAL执行脚本，注意EVAL有将脚本缓存到Redis的作用。这里也可以考虑采用SCRIPT LOAD与EVALSHA的方式。

r = MyRedis(host="r-******.redis.rds.aliyuncs.com", password="***:***", port=6379, decode_responses=True)

print(r.prepend_inLua("k", "v"))
print(r.get("k"))
            
```

## 处理Lua脚本超时

-   现象：
    
    由于Lua脚本在Tair中是原子执行的，Lua慢请求可能会导致Tair阻塞。单个Lua脚本阻塞Tair最多5秒，5秒后Tair会给所有其他命令返回如下BUSY error报错，直到脚本执行结束。
    
    ```sql
    BUSY Redis is busy running a script. You can only call SCRIPT KILL or SHUTDOWN NOSAVE.
    ```
    
    解决方案：
    
    您可以通过SCRIPT KILL命令终止Lua脚本或等待Lua脚本执行结束。
    
    **说明**
    
    -   SCRIPT KILL命令在执行慢Lua脚本的前5秒不会生效（Tair阻塞中）。
    -   建议您编写Lua脚本时预估脚本的执行时间，同时检查死循环等问题，避免过长时间阻塞Redis导致服务不可用，必要时请拆分Lua脚本。
    
-   现象：
    
    若当前Lua脚本已执行写命令，则SCRIPT KILL命令将无法生效，报错示例如下。
    
    ```bash
    (error) UNKILLABLE Sorry the script already executed write commands against the dataset. You can either wait the script termination or kill the server in a hard way using the SHUTDOWN NOSAVE command.
    ```
    
    解决方案：
    
    请在控制台的实例列表中单击对应实例重启。
    

## 脚本缓存、持久化与复制

现象：

在不重启、不调用SCRIPT FLUSH命令的情况下，Redis会一直缓存执行过的Lua脚本。但在部分情况下（例如实例迁移、变配、版本升级、切换等等），Redis无法保证Lua脚本的持久化，也无法保证Lua脚本能够被同步至其他节点。

解决方案：

由于Redis不保证Lua脚本的持久化、复制能力，请您在本地存储所有Lua脚本，在必要时通过EVAL或SCRIPT LOAD命令将Lua脚本重新缓存至Redis中，避免实例重启、HA切换等操作时Redis中Lua脚本被清空而带来的NOSCRIPT错误。

## 集群中Lua脚本的限制

Redis Cluster对使用Lua脚本增加了一些限制，云数据库Redis集群版在这个基础上存在如下额外限制：

**说明** 如果发现无法执行Eval的相关命令，例如提示`ERR command eval not support for normal user`，请将实例的小版本升级至最新。具体操作，请参见[升级小版本](https://help.aliyun.com/document_detail/56450.htm#concept-itn-f44-tdb "云数据库Redis会不断地对内核进行深度优化，用于丰富云产品功能或修复已知缺陷，提升服务稳定性。您可以在控制台上一键将内核升级至最新版本。")。

-   所有key必须在一个slot上，否则返回错误信息：
    
    -ERR eval/evalsha command keys must be in same slot\r\n
    
    **说明** 您可以通过CLUSTER KEYSLOT命令获取目标key的哈希槽（hash slot）。
    
-   对单个节点执行SCRIPT LOAD命令时，不保证将该Lua脚本存入至其他节点中。
-   不支持发布订阅命令，包括PSUBSCRIBE、PUBSUB、PUBLISH、PUNSUBSCRIBE、SUBSCRIBE和UNSUBSCRIBE。
-   不支持UNPACK函数。

若您能够在代码中确保所有操作都在相同slot（如果不能保障这一点，执行会出错），且希望打破Redis集群的Lua限制，可以在控制台将script_check_enable修改为0，则后端不会对脚本进行校验，但仍需要使用KEYS数组至少传递一个key，供代理节点执行路由转发。具体操作，请参见[设置实例参数](https://help.aliyun.com/document_detail/43885.htm#concept-q1w-kxn-tdb "支持自定义部分参数的值，不同的引擎版本和架构支持的参数有所区别，本文为您介绍各参数的设置方法。")。

**代理模式（Proxy）执行Lua的额外限制**

-   所有key都应该由KEYS数组来传递，redis.call/pcall中调用的Redis命令，key的位置必须是KEYS array（不能使用Lua变量替换KEYS），否则直接返回错误信息：
    
    -ERR bad lua script for redis cluster, all the keys that the script uses should be passed using the KEYS array\r\n
    
    正确与错误命令示例如下：
    
    ```python
    # 本示例的准备工作需执行如下命令
    SET foo foo_value
    SET {foo}bar bar_value
    
    # 正确示例
    EVAL "return redis.call('mget', KEYS[1], KEYS[2])" 2 foo {foo}bar
    
    # 错误示例
    EVAL "return redis.call('mget', KEYS[1], '{foo}bar')" 1 foo
    EVAL "return redis.call('mget', KEYS[1], ARGV[1])" 1 foo {foo}bar
    ```
    
-   调用必须要带有key，否则直接返回错误信息：
    
    -ERR for redis cluster, eval/evalsha number of keys can't be negative or zero\r\n
    
    正确与错误命令示例如下：
    
    ```python
    # 正确示例
    EVAL "return redis.call('get', KEYS[1])" 1 foo
    
    # 错误示例
    EVAL "return redis.call('get', 'foo')" 0
    ```
    
-   不支持在MULTI、EXEC事务中使用EVAL、EVALSHA、SCRIPT系列命令。
-   不支持在Lua中执行跨Redis节点的命令，例如KEYS、SCAN等。
    
    为了保证Lua执行的原子性，Proxy会根据KEYS参数将Lua发送到一个Redis节点执行并获取结果，从而导致该结果与全局结果不一致。
    

**说明**

若您需要使用代理模式下受限的部分功能，您可以尝试开通使用云数据库Redis集群版的直连模式。但是由于云数据库Redis集群版在迁移、变配时都会通过proxy代理迁移数据，直连模式下不符合代理模式的Lua脚本会迁移、变配失败。

建议您在直连模式下使用Lua脚本时应尽可能符合代理模式下的限制规范，避免后续Lua脚本迁移、变配失败。

# Reference
https://help.aliyun.com/document_detail/92942.html