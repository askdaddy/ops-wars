# VPN连接IDC和腾讯云，内网互联
公司项目有部分业务要从自管理的IDC迁移到腾讯云，由于业务间有NFS共享配置文件以及内网Api互访的需求打通2个Lan势在必行。



## 现有网络以及设备的情况
- IDC
  - 192.168.0.0/16
  - Gateway: 192.168.0.1 / 外网ip保密
  - Tinc: 192.168.107.102

- Tcloud
  - 10.10.0.0/16 (VPS)
  - Gateway: 10.10.0.1
  - Tinc: 10.10.10.2 / 外网ip保密，下文使用 <tcloud-tip> 代表

> 以上tinc服务器环境为 Ubuntu18.04.5

## 配置VPN
tinc网络名: apowo

```bash
# 每台机器上的tinc安装
$ sudo apt install -y tinc
$ sudo mkdir -p /etc/tinc/apowo/hosts
```

### @IDC.Tinc 配置

*File: /etc/tinc/apowo/tinc.conf*
```conf
Name=idc  
Interface=tinc0  
Mode=switch  
ConnectTo=tcloud  
```

*File: /etc/tinc/apowo/tinc-up*
```shell
#!/bin/sh  
ip link set $INTERFACE up  
ip addr add 10.28.0.253/24 dev $INTERFACE
```

*File: /etc/tinc/apowo/tinc-down*
```shell
#!/bin/sh
ip link set $INTERFACE down
```

```bash
$ sudo tincd -n apowo -K 4096 
```

*File: /etc/tinc/apowo/hosts/idc*
```txt
Subnet = 192.168.0.0/16

-----BEGIN RSA PUBLIC KEY-----
MIICCgKCAgEApwOzjmzfaWTrJGAplQymJhZfFAH1ilwq0Tr3u6f5+7KNPK9aIWJk
op2EkaYveG5s+wRFF+eDUNDIrAhG2BKeS0qWPSMzUyznVq95nNPGUjMF5qqhcdJj

XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX

YVw7BtHVfuYgjqhv6jAggKvZmvH6KRMZPDjmcJyLBVsIs30W/2QY6MD9+Y7bZW4I
u0s2nigSkVOzUd0N4NiHlOPZHoWcFm4rkSs3h0R6D0P8xA2Xx2mkKBkCAwEAAQ==
-----END RSA PUBLIC KEY-----
```


### @Tcloud.Tinc 配置
*File: /etc/tinc/apowo/tinc.conf*
```conf
Name=tcloud  
Interface=tinc0  
Mode=switch  
ConnectTo=tcloud  
```

*File: /etc/tinc/apowo/tinc-up*
```shell
#!/bin/sh  
ip link set $INTERFACE up  
ip addr add 10.28.0.254/24 dev $INTERFACE
```

*File: /etc/tinc/apowo/tinc-down*
```shell
#!/bin/sh
ip link set $INTERFACE down
```

```bash
$ sudo tincd -n apowo -K 4096 
```

*File: /etc/tinc/apowo/hosts/idc*
```txt
Address = <tcloud-tip>
Subnet = 10.10.0.0/16

-----BEGIN RSA PUBLIC KEY-----
MIICCgKCAgEAzLaCRQ9mcoKfXX2ryT2VoXu7LSiCfRPWxXpAxmTt1GgMW/IHu8ZY
aipjaxpdi+J1aXx88YVhTHwLlFR7CUfeufPV7J0VCj+Bd0Mv6jX6odGBijwmI24Y
ADwbQ3cEHQqfBxxMDCDtMmTquGe1c4Suev/X7Tc0sQiuDwsGFpVdmktt+IXbVYXl

XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX

BGmPKMYvnnmTSvaOuHY6UnYwWgjolSR1ehl0Uo2vVEH7i4JCX2p1ef8N7VjtD9Xg
nVCTUXX72FlQVUaS894NPJ3OueOdeskoMb8/eSyLEaPOddSpaweQBk0CAwEAAQ==
-----END RSA PUBLIC KEY-----
```

### 启动 Tinc
1. 将生成的hosts公钥互相注入到对端的hosts目录中
2. ``` $ sudo systemctl enable tinc@apowo ```

### 验证 VPN的连通性
在两端互ping tinc私网ip 10.28.0.25x


## 配置动态路由
```bash
# 每台机器上的bird安装
$ sudo apt install -y bird
```

### @IDC.Bird 配置
*File: /etc/bird/bird.conf*
```conf
router id 10.28.0.253;

protocol device {
        scan time 10;
}

protocol kernel {
        metric 64;      # Use explicit kernel route metric to avoid collisions
                        # with non-BIRD routes in the kernel routing table
        export all;     # Actually insert routes into the kernel routing table
        persist;        # Don't remove routes on bird shutdown
        scan time 10;   # Scan kernel routing table every 20 seconds
}

protocol direct {
        interface "ens19"; 
}

protocol ospf {
        import all;
        export all;
        area 0.0.0.0 {
                interface "tinc0" {
                        hello 9;
                        retransmit 6;
                        cost 10;
                        transmit delay 5;
                        dead count 60;
                        wait 10;
                        type pointopoint;
                };
        };
}
```



### @Tcloud.Bird 配置
*File: /etc/bird/bird.conf*
```conf
router id 10.28.0.254;

protocol device {
        scan time 10;
}

protocol kernel {
        metric 64;      # Use explicit kernel route metric to avoid collisions
                        # with non-BIRD routes in the kernel routing table
        export all;     # Actually insert routes into the kernel routing table
        persist;        # Don't remove routes on bird shutdown
        scan time 10;   # Scan kernel routing table every 20 seconds
}

protocol direct {
        interface "eth0";
}

protocol ospf {
        import all;
        export all;
        area 0.0.0.0 {
                interface "tinc0" {
                        hello 9;
                        retransmit 6;
                        cost 10;
                        transmit delay 5;
                        dead count 60;
                        wait 10;
                        type pointopoint;
                };
        };
}
```

### 启动bird
```bash
$ sudo systemctl restart bird
```

### 验证
@IDC.Bird

```shell
$ ip r
default via 192.168.0.1 dev ens19 proto static
10.10.10.0/24 via 10.28.0.254 dev tinc0 proto bird metric 64
10.28.0.0/24 dev tinc0 proto kernel scope link src 10.28.0.253
192.168.0.0/16 dev ens19 proto kernel scope link src 192.168.107.102
```
@Tcloud.Bird
```shell
$ ip r
default via 10.10.10.1 dev eth0 proto dhcp src 10.10.10.2 metric 100
10.10.10.0/24 dev eth0 proto kernel scope link src 10.10.10.2
10.28.0.0/24 dev tinc0 proto kernel scope link src 10.28.0.254
192.168.0.0/16 via 10.28.0.253 dev tinc0 proto bird metric 64
```


