![[Pasted image 20220928181842.png]]
 es 是数据库里面的执行时间(event time)， 而 ts 是解析时间(process time)
 id是canal自己维护的一个递增id，但是这个id并不是指每条canal entry都拥有不同的id,多条entry都可能是同一个批次
 
