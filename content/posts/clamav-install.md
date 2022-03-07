---
title: "Linux 杀毒软件 ClamAV 安装"
categories: [ "linux" ]
tags: [ "杀毒软件", "ClamAV" ]
draft: false
slug: "clamav-install"
date: "2022-03-07T21:52:00+08:00"
---

**Clam AntiVirus**（**ClamAV**）是免费而且开放源代码的杀毒软件，软件与病毒码的更新皆由社群免费发布。Github 地址：https://github.com/Cisco-Talos/clamav

### 安装 ClamAV

```bash
yum install -y clamav*
```

### 配置 ClamAV

```bash
cat > /etc/freshclam.conf <<EOF
# 数据库配置文件夹
DatabaseDirectory /var/lib/clamav
# 更新日志文件夹
UpdateLogFile /var/log/freshclam.log
# 日志大小
LogFileMaxSize 2M
# 日志记录时间
LogTime yes
# 所属用户
DatabaseOwner root
# 同步病毒库的地址
DatabaseMirror database.clamav.net
EOF
```

### 更新病毒库

大概需要几分钟时间

```bash
freshclam
```

### 进行病毒扫描测试

```bash
clamscan -ri </path1/to/scan> </path2/to/scan>
```

常用配置说明

```
--recursive[=yes/no(*)]     -r      递归查找
--infected                  -I      只打印受影响的文件信息
--remove[=yes/no(*)]                删除受影响的文件。(不建议使用,根据扫描结果进行手动删除,避免误删。)
```

### 配置邮箱

如果不需要邮件通知的可以忽略此步骤。

#### 安装邮件服务

```bash
yum install -y mailx
```

#### 修改配置

修改大写字母为你的邮箱配置

```bash
cat > /etc/mail.rc <<EOF
set from=USERNAME@YOURDOMAIN.COM
set smtp=smtps://smtp.exmail.qq.com:465
set smtp-auth=login
set smtp-auth-user=USERNAME@YOURDOMAIN.COM
set smtp-auth-password=YOURPASSWORD
set ssl-verify=ignore
set nss-config-dir=/etc/pki/nssdb/
EOF
```

#### 发送邮件进行测试

```bash
echo "Your message" | mail -v -s "Message Subject" email@address
```

如果出现 “Error in certificate: Peer's certificate issuer is not recognized” 属于正常。

### 定期扫描

使用 `crontab -e` 增加两个定时任务

```bash
# 每天凌晨 2 点半进行病毒库更新
30 2 * * * /bin/freshclam
# 每天凌晨 3 点半进行病毒扫描
30 3 * * * /bin/clamscan -ri </path1/to/scan> | mail -s "病毒文件扫描报告" 'youremailaddress'
```

如果不需要发送邮件使用下面的配置

```bash
# 每天凌晨 2 点半进行病毒库更新
30 2 * * * /bin/freshclam
# 每天凌晨 3 点半进行病毒扫描
30 3 * * * /bin/clamscan -ri </path1/to/scan>
```



