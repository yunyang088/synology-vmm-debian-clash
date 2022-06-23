# 群晖 VMM 安装 Debian 和 Clash 的流程记录

# 0x00 前言

## 折腾目的
在群晖 VMM 虚拟的 Debian 上安装 Clash Premium 用作家庭旁路由，接管需要代理的设备

## 环境说明
用淘汰机器组装起来的黑群晖，配置为 `i5-6500/16G`，系统版本为 `DSM 7.1 Up1`，以下使用的平台均为 `linux-amd64`

#  0x01 前期准备

## Debian
按D大的说法直接用 testing 的镜像就可以，附上下载地址：[debian-testing-amd64-netinst.iso](https://cdimage.debian.org/cdimage/weekly-builds/amd64/iso-cd/debian-testing-amd64-netinst.iso)

## Clash Premium
可以直接用 latest，附上下载地址：[clash-linux-amd64-latest.gz](https://release.dreamacro.workers.dev/latest/clash-linux-amd64-latest.gz) 

# 0x02 Debian

## 安装系统

VMM 里面映像导入 ISO，等待群晖处理完毕

虚拟机新建，选择 Linux，配置 CPU 核数，内存大小等等，此处 CPU 一定要选择 `启用 CPU 兼容模式`，否则很有可能在安装系统的过程中卡在最后的 `installing GRUB driver`，会无限的 `Looking for other operating systems`

Debian 安装过程中基本都按默认就行，地区选择 `China` 后，在 apt 安装界面可以选择镜像源，这边选择比较熟悉的就行，不建议安装 `Desktop` 相关的内容，毕竟只是用作 Clash 的载体，操作均可使用命令行完成

## 调整 ssh，安装 sudo

安装成功后会进入 Debian 的登录界面，初次建议使用 `root` 登录

启用 `root` 的 `ssh` 登录，`nano /etc/ssh/sshd_config`，修改内容为：
```ini
PermitRootLogin yes
```

安装 `sudo`（Debian 并没有默认安装 sudo）:
```bash
$ apt -y install sudo
```
当然足够自信的话全程用 root 操作也可以，sudo 也大可不装（不是

立即配置 ssh 的原因是我在尝试取消 cpu兼容模式后，vnc 进系统直接卡在了 grub 的界面， ssh 可以登录普通用户但是没有配置 sudo 也没开 root 的访问权限，等于陷入了死胡同

第一时间配置 ssh 后，关机并建立快照，留一条后路

## 调整网络配置
编辑 `/etc/sysctl.conf`，开启 ipv4 流量转发，增加或取消注释 `net.ipv4.ip_forward` 修改为 `1`
```ini
net.ipv4.ip_forward=1
```
保存配置，关机，建快照
```bash
$ sysctl --system
$ shutdown -h now
```

# 0x03 Clash Premium 

>以下操作均以 root 操作（步骤参考 [官方页面说明](https://github.com/Dreamacro/clash/wiki/clash-as-a-daemon)）
>配置的策略为：dns fake-ip，tcp-转发，udp-tun
>auto-redir 为 6.15 后新编译的功能，务必确认版本，文章上方的给的链接可用
>下面给出 `config.yaml` 里相关配置的参考：

## Config 相关
```yaml
auto-redir:
  enable: true
  auto-route: true
tun:
  enable: true
  stack: system
  auto-route: true
  auto-detect-interface: false
  dns-hijack:
    - any:53
dns:
  enable: true
  ipv6: false
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
  listen: 0.0.0.0:53
  nameserver:
  - 223.5.5.5
  - 119.29.29.29
  fallback:
  - https://1.1.1.1/dns-query
  - https://dns.google/dns-query
  - https://doh.pub/dns-query
```

## 配置守护进程
将 Clash 二进制文件复制到 `/usr/local/bin`，并将配置文件放到 `/etc/clash`
```bash
$ mv clash-linux-amd64-latest /usr/bin/clash
$ chmod +x /usr/bin/clash
$ cp config.yaml /etc/clash/
$ cp Country.mmdb /etc/clash/
```

配置守护进程，创建系统配置 `/etc/systemd/system/clash.service`
```ini
[Unit]
Description=Clash daemon
After=network.target

[Service]
Type=simple
Restart=always
ExecStart=/usr/local/bin/clash -d /etc/clash

[Install]
WantedBy=multi-user.target
```

**启用** Clash 守护进程
```bash
$ systemctl enable clash
```

**启动** Clash 守护进程
```bash
$ systemctl start clash
``` 

其他相关命令
```bash
$ systemctl status clash #查看服务状态
$ systemctl restart clash #重启服务
$ systemctl stop clash #关闭服务
$ journalctl -u clash -f #查看clash log
```

# 0x04 使用

如果 Clash 启动成功没报错的话，就可以尝试把家里中的设备网关和 DNS 都改为 Debian 虚拟机的 ip，能上网的话就大功告成啦

**别忘了建快照！**
