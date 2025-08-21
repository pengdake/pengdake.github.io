---
layout: single
toc: true
toc_sticky: true
title:  "Keepalived & DRBD Recovery"
categories: [ha, Keepalived, DRBD]
tags: [ha, Keepalived, DRBD]
---

# Keepalived & DRBD 的恢复流程
## 基本架构
主机jmk 和 pi，它们通过 DRBD 进行数据同步，并使用 Keepalived 实现高可用性。DRBD 负责数据的实时复制，而 Keepalived 负责 VIP（虚拟 IP 地址）的漂移和服务的高可用性， drbd主要挂载为nfs服务的底层存储， 而nfs计划作为k3s的storageclass存储之一，为k3s集群提供持久化存储(pvc)功能。
两台主机均为keepalived备节点，通过权重区分主从，pi权重较高，优先成为主节点。具体配置如下
```
# jmk的keepalived配置/etc/keepalived/keepalived.conf
global_defs {
    script_security
}

vrrp_script chk_nfs {
    script "/etc/keepalived/chk_nfs.sh"
    interval  3
    weight -30
    fall   3
    rise   3
}

vrrp_instance VI_1 {
    state BACKUP # 没有 Master 存在，VRRP 会根据优先级选举 Master
    priority 90 # 权重
    interface br0
    nopreempt # 禁止节点抢占已有Master的vip
    virtual_router_id 51
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.31.90/24 dev br0 # vip
    }

    track_script {
        chk_nfs
    }

    notify_master "/etc/keepalived/notify_master.sh"
    notify_backup "/etc/keepalived/notify_backup.sh"
    notify_fault  "/etc/keepalived/notify_fault.sh"
}
``` 
```
# pi的keepalived配置/etc/keepalived/keepalived.conf
global_defs {
    script_security
}

vrrp_script chk_nfs {
    script "/etc/keepalived/chk_nfs.sh"
    interval  3
    weight -30
    fall   3
    rise   3
}

vrrp_instance VI_1 {
    state BACKUP
    priority 100
    interface eth0
    nopreempt
    virtual_router_id 51
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.31.90/24 dev eth0
    }

    track_script {
        chk_nfs
    }

    notify_master "/etc/keepalived/notify_master.sh"
    notify_backup "/etc/keepalived/notify_backup.sh"
    notify_fault  "/etc/keepalived/notify_fault.sh"
}
```
## 场景
### 脑裂
#### 条件
脑裂出现的条件：
* 原 Master 宕机，Backup 升为 Primary 并有写入
* 原 Master 恢复，系统发现双方 UUID 不一致 → Split-Brain
#### 现象
当主节点关机后，vip切换到备节点，同时备节点的keepalived执行notify_master脚本，将drbd角色切换成主，挂载备节点存储，启动nfs服务，实现nfs的高可用。这时备节点drbd的状态为standalone
``` 
# /etc/keepalived/notify_master.sh

drbdadm up nfs
drbdadm primary --force nfs
mount /dev/drbd0 /nfs
systemctl start nfs-server
``` 
```
# /proc/drbd
version: 8.4.11 (api:1/proto:86-101)
srcversion: 2CC17D07553A98E96473D42
  0: cs:WFConnection ro:Primary/Unknown ds:UpToDate/DUnknown C r-----
    ns:0 nr:0 dw:72 dr:2677 al:4 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:f oos:24576

```

当原主节点启动后，由于主节点存储数据不是最新的，如果错误的强制设置当前节点为drbd的主节点，则可能会导致脑裂，脑裂会导致两台节点无法建立连接，两节点一般都为standalone状态
```
# dmesg --ctime -w | grep drbd 日志中出现Split-Brain detected but unresolved, dropping connection!
[三 8月 13 20:34:47 2025] drbd nfs/0 drbd0: drbd_sync_handshake: 
[三 8月 13 20:34:47 2025] drbd nfs/0 drbd0: self 0E0052B97E996EAA:977BAF16F06E14DC:1C887D28A843874A:1C877D28A843874B bits:5 flags:0 
[三 8月 13 20:34:47 2025] drbd nfs/0 drbd0: peer A2C9761DB7505EFE:977BAF16F06E14DD:1C887D28A843874B:1C877D28A843874B bits:6144 flags:2 
[三 8月 13 20:34:47 2025] drbd nfs/0 drbd0: uuid_compare()=100 by rule 90 [三 8月 13 20:34:47 2025] drbd nfs/0 drbd0: helper command: /sbin/drbdadm initial-split-brain minor-0 [三 8月 13 20:34:47 2025] drbd nfs/0 drbd0: helper command: /sbin/drbdadm initial-split-brain minor-0 exit code 0 (0x0) 
[三 8月 13 20:34:47 2025] drbd nfs/0 drbd0: Split-Brain detected but unresolved, dropping connection! 
[三 8月 13 20:34:47 2025] drbd nfs/0 drbd0: helper command: /sbin/drbdadm split-brain minor-0 
[三 8月 13 20:34:47 2025] drbd nfs/0 drbd0: helper command: /sbin/drbdadm split-brain minor-0 exit code 0 (0x0)
```

#### 恢复步骤
当出现脑裂时，需要按以下步骤恢复
```
# Step 1: 确认保留数据的节点（Backup 已写入为最新）
# 当前节点（保留数据，成为 Primary）
drbdadm primary --force nfs

# Step 2: 对另一节点执行丢弃操作
drbdadm secondary nfs
drbdadm down nfs
drbdadm up nfs
drbdadm disconnect nfs
drbdadm -- --discard-my-data connect nfs

# Step 3: 保留数据节点连接对方
drbdadm connect nfs
```