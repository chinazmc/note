## 为什么会锁表
首先我们要知道，在`InnoDB`事务中，锁是在需要的时候才加上的，但并不是不需要了就立刻释放，而是要等到事务结束时才释放。这个就是**两阶段锁协议**。

然后，在`MySQL5.5`版本中引入了`MDL(Metadata Lock)`，当对一个表做增删改查操作的时候，加`MDL`读锁；当要对表做结构变更操作的时候，加`MDL`写锁。

我们可以简单的尝试一下下面的情况。

![DDL锁等待图](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/17/172c0f58142db6f6~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)

`Session A`开启一个事务，执行了一个简单的查询语句。此时，`Session B`，执行另一个查询语句，可以成功。接着，`Session C`执行了一个`DDL`操作，加了个字段，因为`Session A`的事务没有提交，而且`Session A`持有`MDL`读锁，`Session C`获取不到`MDL`写锁，所以`Session C`堵塞等待`MDL`写锁。又由于`MDL`写锁获取优先级高于`MDL`读锁，因此`Session D`这个时候也获取不到`MDL`读锁，等待`Session C`获取到`MDL`写锁之后它才能获取到`MDL`读锁。

我们发现，DDL操作之前如果存在长事务，一直不提交，DDL操作就会一直被堵塞，还会间接的影响后面其他的查询，导致所有的查询都被堵塞。


## 早期DDL原理

再谈如何对加大表加索引前，先谈一下MySQL DDL操作为什么会锁表？对于这个问题，需要先了解一下MySQL5.6.7之前的早期DDL原理。

早期DDL操作分为`copy table`和`inplace`两种方式。

copy table 方式

1. 创建与原表相同的**临时表**，并在临时表上执行DDL语句
2. **锁原表，不允许DML（数据操作语言），允许查询**
3. 将原表中数据逐行拷贝至临时表（过程没有排序）
4. 原表升级锁，禁止读写，即原表暂停服务
5. rename操作，将临时表重命名原表

inplace 方式（fast index creation，仅支持索引的创建跟删除）

1. 创建**frm**（表结构定义文件）临时文件
2. **锁原表，不允许DML（数据操作语言），允许查询**
3. 根据聚集索引顺序构建新的索引项，按照顺序插入新的索引页
4. 原表升级锁，禁止读写，即原表暂停服务
5. rename操作，替换原表的frm文件

### 早期`copy` VS `inplace` 方式？

inplace 方式相对于 copy 方式来说，inplace 不会生成临时表，不会发生数据拷贝，所以**减少了I/O资源占用**。

inplace 只适用于**索引的创建与删除**，不适用于其他类的DDL语句。

不论是早期copy还是早期inplace方式的DDL，都会进行**锁表操作，不允许DML操作，仅允许查询**。

知道了DDL的机制，下面就了解一下“如何对大表进行加索引操作”！

## 方案一：“影子策略”

此方法来自《高性能MySQL》一书中的方案。

### 方案思路

1. 创建一张与原表（tb）结构相同的新表（tb_new）
2. 在新表上创建索引
3. 重命名原表为其他表名（tb => tb_tmp），新表重命名为原表名（tb_new => tb），此时新表（tb）承担业务
4. 为原表（tb_tmp）新增索引
5. 交换表，新表改回最初的名称（tb => tb_new），原表改回最初的名称（tb_tmp => tb），原表（tb）重新承担业务
6. 把新表数据导入原表（即把新表承担业务期间产生的数据和到原表中）

### 如何实践

SQL实现：

# 以下sql对应上面六步

create table tb_new like tb;

alter table tb_new add index idx_col_name (col_name);

rename table tb to tb_tmp, tb_new to tb;

alter table tb_tmp add index idx_col_name (col_name);

rename table tb to tb_new, tb_tmp => tb;

insert into tb (col_name1, col_name2) select col_name1, col_name2 from tb_new;

“影子策略”有哪些问题？

步骤3之后，新表改为原表名后（tb）开始承担业务，步骤3到结束之前这段时间的新产生的数据都是存在新表中的，但是如果有业务对老数据进行修改或删除操作，那将无法实现，所以步骤3到结束这段时间可能会产生数据（更新和删除）丢失。

## 方案二：pt-online-schema-change

PERCONA提供若干维护MySQL的小工具，其中 pt-online-schema-change（简称pt-osc）便可用来相对安全地对大表进行DDL操作。

pt-online-schema-change 方案利用三个触发器（DELETE\UPDATE\INSERT触发器）解决了“影子策略”存在的问题，让新老表数据同步时发生的数据变动也能得到同步。

### 工作原理

1. 创建一张与原表结构相同的新表
2. 对新表进行DDL操作（如加索引）
3. 在原表上创建3个触发器（DELETE\\UPDATE\\INSERT），用来原表复制到新表时（步骤4）的数据改动时的同步
4. 将原表数据以数据块（chunk）的形式复制到新表
5. 表交换，原表重命名为old表，新表重命名原表名
6. 删除旧表，删除触发器

### 如何使用

见[使用 pt-online-schema-change 工具不锁表在线修改 MySQL 表结构](https://link.segmentfault.com/?enc=uCyDRmNcJ0NL484LKsP62Q%3D%3D.olrJShREJ%2B5B0P0VSxo2y6985sbJGsimUoLag96EXKYwo4KnP%2FqcQYWwcyfp2re658%2Fl0OX5ciYJKT7kI5a%2B%2By3rbUvTwkoLFKyKPSQKzsvx6VkygVzF7MQVtPCjEgdk2v2d3OoFG6%2F%2BaJ%2BsKMiOXDkwq6kqQdItrAUPgTuppeM%3D)一文

### 问题疑惑

见[pt-online-schema-change的原理解析与应用说明-问题解答](https://link.segmentfault.com/?enc=6jk0%2FteSMFr2OAV7jwTC9A%3D%3D.Vq37n48z2%2F5vQ6N7wv57Mx5vpyOhjnIBaGLv5VGd76S04P3c4TRdcl3AVPsMyJJlbacnNc2KfmQuj7rw3Q0rpA%3D%3D)

## 方案三：ONLINE DDL

MySQL5.6.7 之前由于DDL实现机制的局限性，常用“影子策略”和 pt-online-schema-change 方案进行DDL操作，保证相对安全性。在 MySQL5.6.7 版本中新推出了 Online DDL 特性，支持“无锁”DDL。5.7版本已趋于成熟，所以在5.7之后可以直接利用 ONLINE DDL特性。

对于 ONLINE DDL 下的 inplace 方式，分为了 `rebuild table` 和 `no-rebuild table`。

### Online DDL执行阶段

大致可分为三个阶段：初始化、执行和提交

#### Initialization阶段

此阶段会使用MDL读锁，禁止其他并发线程修改表结构  
服务器将考虑存储引擎能力、语句中指定的操作以及用户指定的ALGORITHM 和 LOCK选项，确定操作期间允许的并发数

#### Execution阶段

此阶段分为两个步骤 Prepared and Executed  
此阶段是否需要MDL写锁取决于Initialization阶段评估的因素，如果需要MDL写锁的话，仅在Prepared过程会短暂的使用MDL写锁  
其中最耗时的是Excuted过程

#### Commit Table Definition阶段

此阶段会将MDL读锁升级到MDL写锁，此阶段一般较快，因此独占锁的时间也较短  
用新的表定义替换旧的表定义(如果rebuild table)

### ONLINE DDL 过程

1. 获取对应要操作表的 MDL（metadata lock）写锁
2. MDL写锁 降级成 MDL读锁
3. 真正做DDL操作
4. MDL读锁 升级成 MDL写锁
5. 释放MDL锁

在第3步时，DDL操作时是不会进行锁表的，可以进行DML操作。但可能在拿DML写锁时锁住，见文章[MySQL · 源码阅读 · 白话Online DDL](https://link.segmentfault.com/?enc=XK99N2v5vE4UlMkitzE66A%3D%3D.wWOXxSyC9JKDVwI8rdxwKcjP6IrKFuhM8bIW0mu87NeS%2F2Ild5f%2FUHQk03A5E8RA)

## `pt-osc`执行过程

1. 创建一个和原表表结构一样的临时表(`_tablename_new`)，执行`alter`修改临时表表结构。
2. 在原表上创建3个与`insert` `delete` `update`对应的触发器，用于`copy`数据的过程中，在原表的更新操作，更新到新表。
3. 从原表拷贝数据到临时表，拷贝过程中在原表进行的写操作都会更新到新建的临时表。
4. `rename`原数据表为`old`表，把新表`rename`为原表名，并将`old`表删除。
5. 删除触发器。

这里面创建、删除触发器和`rename`表的时候都会尝试获取`DML`写锁，如果获取不到会等待。就是我们看到的`Waiting for table metadata lock`。

所以，这些时间段如果长时间获取不到锁，就会一直堵塞，还是会出现问题的。

## `Online DDL`执行过程

1. 拿`MDL`写锁
2. 降级成`MDL`读锁
3. 真正做`DDL`
4. 升级成`MDL`写锁
5. 释放`MDL`锁

`1、4`如果没有锁冲突，执行时间非常短。第3步占用了`DDL`绝大部分时间，这期间这个表可以正常读写数据，因此称为**online**。

但是，如果拿锁的时候没拿到，或者升级`MDL`写锁不能成功，就会等待，我们又会看到`Waiting for table metadata lock`，然后就接着的一系列问题了。

## 总结

加个索引，说难也难，说不难也不难。如果数据量大，又存在长事务，加索引的过程又有用户访问，`Online DDL`和`pt-osc`都不能保证对业务没有影响。但是如果我们`SQL`的执行时间比较短，或者我们加索引的时候，对应的业务没有多少请求。那么我们就可以很快的加完索引。

加字段也是类似的过程，但是如果我们能保证没有慢`SQL`，那么就不会存在长事务，那么执行时间就会很快，对用户就可以做到几乎没有影响。至于选择`Online DDL`还是`pt-osc`就要看他们的一些限制以及自己的场景需求了。感兴趣的同学，自己尝试一下。



# Reference
https://segmentfault.com/a/1190000040570831
https://juejin.cn/post/6844904193531052040