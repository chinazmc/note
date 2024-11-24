![[Pasted image 20240229194101.png]]

![[Pasted image 20240229194402.png]]

![[Pasted image 20240229194451.png]]

![[Pasted image 20240229194625.png]]

![[Pasted image 20240229194847.png]]
一拖n的话对cpu使用率比多主分片的话会变大4倍多。

![[Pasted image 20240229195322.png]]

![[Pasted image 20240229195505.png]]
可能被refresh，被merge,在flush

使用队列的方式写入会好些
![[Pasted image 20240229195650.png]]

管道的方式减少频繁更新，es对写入过程的refresh过程cpu很容易飙升




