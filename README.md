# routeosprojectscript
#1个脚本 2个任务

--------------/system scheduler------------------------------
添加定时任务管理每10分钟检查一次，到点启动 wol 任务，到点停止 wol 任务
-----------------5m---创建一个任务，内容放进去-------------------------------------
:global stime [:totime 8:00:00]
:global etime [:totime 9:10:00]
:global tscheduler [/system scheduler find name=wol]
foreach i in=$tscheduler do={
    :if ([/system clock get time]> $stime) do={
        :if ([/system clock get time]> $etime) do={[/system scheduler disable $i];} else={[/system scheduler enable $i];}} else={
    [/system scheduler disable $i];}}
    
---------------/system scheduler wol---------------------------
创建一个定时任务 名字为 wol  目的 15秒运行一次  wolscript 脚本 
/system scheduler add disabled=yes interval=15s name=wol on-event="/system script run wolscript" policy=\
    ftp,reboot,read,write,policy,test,password,sniff,sensitive,romon start-time=startup
    
---------------配置--------------------------------------------

添加在线检查，标记lan   wifi
/ip firewall mangle
add action=add-src-to-address-list address-list=lan address-list-timeout=3m chain=prerouting src-address=\
    192.168.0.1-192.168.1.240
add action=add-src-to-address-list address-list=wifi address-list-timeout=3m chain=prerouting src-address=\
    172.16.0.1-172.16.1.240
在/ip arp 条目中的comment 写上触发的MAC地址 默认截取前17个字符作为识别
pcinterface  需要唤醒的pc网络 ，网卡

------------创建脚本-"wolscript" 内容放入脚本------------------------------------------

:global pcinterface "vlan1000 192.168.0.1";

:global mac2pc "";
:foreach i in=[/ip arp find interface=$pcinterface ] do={
:if ([:len [/ip arp get $i comment ]]> 16) do={   
:local wifimac [:pick [/ip arp get $i comment ] 0 17 ];
:local wolmac  [/ip arp get $i mac-address ] ;
:local wolip  [/ip arp get $i address ] ;
:set mac2pc (mac2pc .",". $wifimac  ."". $wolmac ."". $wolip)}};

:local strtemp "";
foreach i in=[/ip firewall address-list find list=wifi] do={:set strtemp ($strtemp .",". [/ip firewall address-list get $i address ])};
:local wifilistip [:toarray $strtemp];
:set strtemp "";
foreach i in=[/ip firewall address-list find list=lan] do={:set strtemp ($strtemp .",". [/ip firewall address-list get $i address ])};
:local pclistip [:toarray $strtemp];

:foreach i in=[:toarray $mac2pc] do={
:local wolip [:pick $i 34 50] ;
:local wifimac [:pick $i 0 17] ;
:local wolmac [:pick $i 17 34] ;
:if ([:find $pclistip $wolip] >0) do={:nothing ;
}  else={
:local wifiip [/ip arp get [/ip arp find mac-address=$wifimac] address];
:if ([:len $wifiip]>0 ) do={
:if ([:find $wifilistip $wifiip] > 0 ) do={
:log info "-----------wifiip :$wifiip  WOL ing  PC ip: $wolip ,pc opening  ";
[/tool wol interface=$pcinterface mac=$wolmac];}}}};

-----------------------------
