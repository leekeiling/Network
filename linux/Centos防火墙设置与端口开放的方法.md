## Centos防火墙设置与端口开放的方法

1. iptables

   ```
   开启防火墙（重启后永久生效）：chkconfig iptables on
   关闭防火墙（重启后永久生效）：chkconfig iptables off
   开启防火墙：service iptables start
   关闭防火墙: service iptables stop
   重启防火墙：service iptables restarted
   ```

2. 查看端口

   ```
   etc/init.d/iptables status
   ```

3. 保存并重启防火墙

   ```
   /etc/init.d/iptables restart
   ```


