# record my common notes

## ip forward
```shell
#!/bin/bash
pro='tcp'
NAT_Host='192.168.56.105'
NAT_Port=10680
Dst_Host='192.168.56.106'
Dst_Port=80
iptables -t nat -A PREROUTING -p $pro --dport $NAT_Port -j DNAT --to-destination $Dst_Host:$Dst_Port
iptables -t nat -A POSTROUTING -p $pro --dport $Dst_Port -d $Dst_Host -j SNAT --to $NAT_Host
#echo 1 > /proc/sys/net/ipv4/ip_forward
```
