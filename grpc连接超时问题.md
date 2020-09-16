# context deadline exceeded

## 第一种 context.WithTimeout

客户端

![image-20200713112137751](未命名/image-20200713112137751.png)



服务端

![image-20200713112209225](未命名/image-20200713112209225.png)



<font color=red size=5x>问题总结</font>

- 客户端用的上下文是context.WithTimeout 超时时间小于服务端的返回时间，造成 ==context deadline exceeded==



## 第二种 context.WithDeadline

客服端

![image-20200713134634544](未命名/image-20200713134634544.png)

服务端

响应时间超过1秒

<font color=red size=5x>结论</font>

- 服务端响应时间超过客户端的等待时间





# Grpc的链接选项配置

![image-20200713133653782](未命名/image-20200713133653782.png)