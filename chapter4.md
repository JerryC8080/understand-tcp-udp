## 状态流转

无论客户端还是服务器，在双方 TCP 通讯的过程中，都会有着一个「状态」的概念，状态会随着 TCP 通讯的不同阶段而变化。

### TCP 状态流转图
![TCP 状态流转图.png](http://om6ayrafu.bkt.clouddn.com/post/understand-tcp-udp/DB900F916ECD267746706FEA8DF682CD.png)

### 各种状态表示的意思

**CLOSED**：表示初始状态

**LISTEN**：表示服务器端的某个 socket 处于监听状态，可以接受连接

**SYN_SENT**：在服务端监听后，客户端 socket 执行 CONNECT 连接时，客户端发送 SYN 报文，此时客户端就进入 SYN_SENT 状态，等待服务端确认。

**SYN_RCVD**：表示服务端接收到了 SYN 报文。

**ESTABLISHED**：表示连接已经建立了。

**FIN_WAIT_1**：其中一方请求终止连接，等待对方的 FIN 报文。

**FIN_WAIT_2**：在 **FIN_WAIT_2** 之后， 当对方回应 ACK 报文之后，进入该状态。

**TIME_WAIT**：表示收到了对方的 FIN 报文，并发送出了 ACK 报文，就等 2MSL 之后即可回到 CLOSED 状态。

**CLOSING**：一种罕见状态，发生在发送 FIN 报文之后，本应是先收到 ACK 报文，却先收到对方的 FIN 报文，那么就从 FIN_WAIT_1 的状态进入 CLOSING 状态。

**CLOSE_WAIT**：表示等待关闭，在 ESTABLISHED 过渡到 LAST_ACK 的一个过渡阶段，该阶段需要考虑是否还有数据发送给对方，如果没有，就可以关闭连接，发送 FIN 报文，然后进入 LAST_ACK 状态。

**LAST_ACK**：被动关闭一方发送 FIN 报文之后，最后等待对方的 ACK 报文所处的状态。

**CLOSED**：当收到 ACK 保温后，就可以进入 CLOSED 状态了。

# 参考
[《后台开发 核心技术与应用实践》](https://book.douban.com/subject/26850616/)    
[《计算机网络》](https://book.douban.com/subject/2970300/)