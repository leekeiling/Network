## http_load

http_load以并行复用方式运行，用于测试web服务器的吞吐量和负载。

命令格式：

http_load -p 并发访问进程数 -s 访问时间 需要访问的URL文件

参数：

-p: 并发的用户进程数

-f: 总计的访问次数

-r：每秒的访问频率

-s：总计的访问时间



测试结果主要指标是 

fetches/sec（每秒的响应请求）、msecs/connect（每个连接的相应时间）