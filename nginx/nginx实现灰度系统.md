公司上线新版本一般通过灰度系统
灰度系统可以把流量划分为多份，一份走新版本代码，一份老版本版本
![[Pasted image 20231125233250.png]]
我们可以在nginx里根据 条件（比如cookie） 里的 version 字段来决定转发请求到哪个服务
在这之前，还需要按照比例来给**流量染色**，也就是返回不同的 cookie![[Pasted image 20231126164830.png]]
