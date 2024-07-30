**网络结构：**公网->Nginx->后端websocket服务

**报错日志：**`2024/07/24 14:03:08 [crit] 708160#0: accept4() failed (24: Too many open files)`

**解决方法：**

1. **查看限制数量**

   ```shell
   # 下面已经是修改过的配置
   ulimit -a
   real-time non-blocking time  (microseconds, -R) unlimited
   core file size              (blocks, -c) 0
   data seg size               (kbytes, -d) unlimited
   scheduling priority                 (-e) 0
   file size                   (blocks, -f) unlimited
   pending signals                     (-i) 30297
   max locked memory           (kbytes, -l) 974046
   max memory size             (kbytes, -m) unlimited
   open files                          (-n) 65535    # 主要看这里打开的句柄数量上限(文件数)
   pipe size                (512 bytes, -p) 8
   POSIX message queues         (bytes, -q) 819200
   real-time priority                  (-r) 0
   stack size                  (kbytes, -s) 8192
   cpu time                   (seconds, -t) unlimited
   max user processes                  (-u) 30297
   virtual memory              (kbytes, -v) unlimited
   file locks                          (-x) unlimited
   ```

2. **查看当前系统的连接数量**

   > 1. **TIME_WAIT**：表示连接已经关闭，但TCP还没有完全删除连接，因为需要确保所有数据包都已经被正确接收和处理。这是一种等待状态，通常持续两倍的最大报文段寿命（2 * MSL）。
   >
   > 2. **SYN_SENT**：表示客户端已经发送了一个SYN（同步序列号）请求来启动一个连接，并在等待服务器的SYN-ACK响应。
   >
   > 3. **FIN_WAIT**：表示连接的一端已经发送了FIN（结束）请求来关闭连接，并在等待对方的确认。
   >
   > 4. **SYN_RECV**：表示服务器接收到一个SYN请求，并发送了一个SYN-ACK响应，等待客户端的ACK确认以完成连接的三次握手。
   >
   > 5. **FIN_WAIT2**：表示连接的一端已经发送了FIN请求并接收到对方的确认，但还在等待对方发送自己的FIN请求以完全关闭连接。
   >
   > 6. **CLOSE_WAIT**：表示连接的一端已经接收到对方的FIN请求，并发送了ACK确认，但还没有发送自己的FIN请求来关闭连接。
   >
   > 7. **ESTABLISHED**：表示连接已经建立，数据可以在双方之间自由传输。
   >
   > 	8.	**LAST_ACK**：表示连接的一端已经发送了自己的FIN请求，并接收到对方的ACK确认，现在在等待对方的FIN请求的最后确认。

   ```shell
   netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}' 
   TIME_WAIT 187
   SYN_SENT 5
   FIN_WAIT1 5
   SYN_RECV 1
   FIN_WAIT2 2
   CLOSE_WAIT 630
   ESTABLISHED 1389
   LAST_ACK 2
   ```

3. **统计Linux文件打开数指令**：`lsof -n|awk '{print $2}'|sort|uniq -c|sort -nr|more`

4. **永久修改ulimit限制**

   > *：代表全局
   > soft：代表软件
   > hard：代表硬件
   > nproc：是代表最大进程数
   > nofile：是代表最大文件打开数

   ```shell
   vim /etc/security/limits.conf
   * soft nproc 65535
   * hard nproc 65535
   * soft nofile 65535 
   * hard nofile 65535
   ```

5. **修改nginx打开文件限制**

   ```shell
   #Syntax:	worker_rlimit_nofile number;
   #Default:	—
   #Context:	main
   #更改工作进程的最大打开文件数 ( RLIMIT_NOFILE ) 限制。用于在不重新启动主进程的情况下增加限制。
   vim nginx.conf
   # 添加配置
   worker_rlimit_nofile 65535;
   ```

   