# 非用户的短文本搜索以及优化之道
![[Pasted image 20230809104908.png]]
![[Pasted image 20230809104959.png]]
![[Pasted image 20230809105021.png]]
![[Pasted image 20230809105132.png]]
![[Pasted image 20230809105140.png]]
![[Pasted image 20230809105202.png]]
![[Pasted image 20230809105239.png]]
![[Pasted image 20230809105309.png]]
![[Pasted image 20230809105339.png]]
![[Pasted image 20230809105350.png]]
使用copy to 减少查询字段
![[Pasted image 20230809111632.png]]

![[Pasted image 20230809112105.png]]
![[Pasted image 20230809112147.png]]
![[Pasted image 20230809112136.png]]

使用rank feature query干预排序
用这个是否可以替代order，性能能提高多少
![[Pasted image 20230809115622.png]]
![[Pasted image 20230809115639.png]]
![[Pasted image 20230809115917.png]]

![[Pasted image 20230809115957.png]]
![[Pasted image 20230809120129.png]]
![[Pasted image 20230809120208.png]]

后续需要查看es的scroll的原理
![[Pasted image 20230809121950.png]]

![[Pasted image 20230809122131.png]]

学会search after

![[Pasted image 20230809161109.png]]
![[Pasted image 20230809161124.png]]
关闭操作系统的swap

锁定堆内存
禁止通配符删除
仅允许系统索引自动创建
加速分片恢复
加速reblance

# 用户短文本搜索以及优化
![[Pasted image 20230809163133.png]]
![[Pasted image 20230809163211.png]]

建模的第一种方式  ordr/\_doc/{订单号}\_{商品id}?routing={用户id}

指定routing好像是一种不错的方式


还可以将所有商品放到订单的数组中。

这是两种建模方式。

![[Pasted image 20230809164527.png]]

![[Pasted image 20230809164545.png]]

![[Pasted image 20230809164837.png]]

![[Pasted image 20230809164909.png]]

冷热用户需要动态分析，不断更新。

![[Pasted image 20230809165405.png]]
![[Pasted image 20230809165413.png]]
![[Pasted image 20230809165446.png]]

![[Pasted image 20230809170702.png]]

![[Pasted image 20230809170715.png]]




















