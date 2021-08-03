# 内网部署vsftpd服务的注意事项
由于N1服务器运行于网关模式，在AC2100的局域网环境下 192.168.0.1/24，为了能够在路由器内网外网均正常使用FTP服务，需要有如下设置：
 - FTP运行于PASV模式下（默认），需要在AC2100上设置FTP的指令和数据的端口映射，如当前设置为FTP指令端口4000，PASV模式的数据端口为50000-50005，需要将这些端口**一一配置端口映射**。<br>鉴于当前路由器上的Socat在配置端口段的映射会产生错误，因此应该在Openwrt的防火墙模块配置端口映射；当配置的内外网端口段长度相同时，根据Openwrt的规则，会从小到大产生一一映射的关系。<br>
> 例如配置的内外网端口段分别为内网：100-200，外网：150-250，则会产生100→150，101→151，...的映射
  
 - 在配置端口映射时，有的路由器不支持端口段的映射，因此只能够一一配置，考虑到工作量，应该尽量减少PASV数据端口段的长度
 - 在配置完端口映射后，需要设置FTP回传地址。默认情况下由于vsftpd的回传地址是本机LAN地址，但访问时是通过路由器WAN地址进行访问，二者不一致，为此需要特别配置该参数。此外为了保证路由器下的设备还能正常通过LAN地址访问FTP服务，需要将FTP安全检查关闭<br>
> 5.pasv_promiscuous选项参数说明：
> 
> 此选项激活时，将关闭PASV模式的安全检查。该检查确保数据连接和控制连接是来自同一个IP地址。小心打开此选项。此选项唯一合理的用法是存在于由安全隧道方案构成的组织中。默认值为NO。
> 
> 合理的用法是：在一些安全隧道配置环境下，或者更好地支持FXP时(才启用它)。
> 
> ————————————————
> 
> 版权声明：本文为CSDN博主「穿越清华」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
> 
> 原文链接：https://blog.csdn.net/qq_15127715/article/details/69055099
 - 为了实现这两个功能，需要修改相关设置。但luci并没有提供这些选项，所以只能手动修改相关配置文件<br>

> N1的Openwrt使用vsftpd-alt，配合luci-app-vsftpd，运行时仍然为vsftpd进程，与一般的vsftpd一致。每当vsftpd服务重启，会在/var/run/vsftpd路径下生成vsftpd.conf文件，该文件语法与一般的vsftpd相同；
> 
> luci网页端的设置依靠Openwrt内置的lua语句模块，在/etc/config/vsftpd下生效，这里的设置只针对luci服务，擅自添加luci中没有的变量将不会生效
> 
> 自行添加参数需要修改/usr/sbin/vsftpd_prepare文件，仿照已有语句的语法编写即可

总体来说，需要在/usr/sbin/vsftpd_prepare文件的`# listen`块增加如下代码：

    output_field listen pasv_address "pasv_address" "(WAN ip address on AC2100)"
    output_const "pasv_promiscuous" YES
    !(https://raw.githubusercontent.com/iky1905/vsftpd-alt-on-LAN/main/%E6%8D%95%E8%8E%B7.PNG?token=AOOIP6N6AM737WY622I7XXTBBFAJ6)
 - 每更换一次网络环境，AC2100的WAN地址就不一样，尤其是在校园网更换为PPPoE模式后，该地址更是不能保持固定，为了不用重复配置，最好的方法是编写一个脚本，能够让N1获取AC2100的WAN地址，并将其写入相应的配置文件中，这样就避免了每更换一次WAN地址就要修改一次配置的问题。一个可能的思路如下：
> 1. 在AC2100上运行脚本，每拨号成功，就将WAN地址写到一个文件里吗，这可以使用ifconfig eth0.1(在当前使用的固件中，该网卡代表路由器的WAN网卡)配合正则表达式完成，最终获取ip地址的字符串
> 2. 在N1上运行脚本，以SCP协议从AC2100下载包含WAN地址的文件到N1上。但目前遇到了问题，**SCP协议是基于ssh的，在首次连接是需要确认相应的指纹，这就要求N1能够回应“是否接受该指纹”的请求，一般的命令行并不能这么做；如果该脚本在N1开机后执行，由于该执行主体并没有接受过该指纹，于是这个动作就被中断了，但如果在开机后在ssh的命令行输入同样的代码，并且此前认为确认过该指纹，那么命令将会正常执行，从AC2100获取该文件**
> 3. 接下来的操作还没有想好，但是理论上仅使用echo加正则表达式应该可以做到**在正确的位置写入正确的地址**
 - 上述操作仅针对N1中的配置文件，至于用户如何知道每次拨号的WAN地址并实现从路由器的外网访问N1，不在本文的讨论范围内
