+++
title = "acme.sh 免费证书神器"
date = "2025-03-25"
tags = ["SSL"]
categories = ["Tech"]
summary = "独立开发好工具介绍"
description = "免费证书 acme.sh"
+++

在生产环境中，为网站配置 HTTPS 已经成为标配。本文将介绍如何通过 [`acme.sh`](https://github.com/acmesh-official/acme.sh) 工具，结合阿里云的 DNS API，实现自动化签发和更新免费的 Let's Encrypt 证书，并将其应用到 Nginx 中。

## 准备工作

在开始之前，需要具备以下条件：

- 一个在阿里云注册的域名（如 `domain.cn`）。
- 创建一个阿里云的 RAM 子账号，并授予其管理 DNS 的权限。
- 安装了 `acme.sh` 工具（可以通过 `curl` 或 `git` 安装）。
- 部署了 Nginx 服务。

## 创建 RAM 子账号并授权

1. 登录 [阿里云 RAM 控制台](https://ram.console.aliyun.com/)。
2. 创建一个新用户，例如 `acme-dns`，为其创建 AccessKey ID 和 AccessKey Secret。
3. 为该用户授予 AliyunDNSFullAccess 权限。

## 设置环境变量

在终端中导出阿里云的访问密钥，用于 DNS 验证。

```bash
export Ali_Key="你的AccessKeyID"
export Ali_Secret="你的AccessKeySecret"
```

你也可以将上述变量写入 `~/.bashrc` 或 `~/.zshrc` 中，便于长期使用。

## 第一步：使用 DNS 自动验证方式申请证书

使用以下命令，通过阿里云 DNS 自动完成域名验证，并申请证书：

```bash
acme.sh --issue --dns dns_ali -d domain.cn -d *.domain.cn
```

说明：
- `--dns dns_ali` 表示使用阿里云 DNS 方式进行验证。
- `-d domain.cn -d *.domain.cn` 表示为主域名和通配符子域名申请证书。

更多支持的 DNS 服务商，请参考 [官方文档](https://github.com/acmesh-official/acme.sh/wiki/dnsapi#dns_ali)。

## 第二步：安装证书到指定路径

证书签发完成后，需要将其安装到 Nginx 可读取的路径中：

```bash
acme.sh --install-cert -d domain.cn \
  --key-file /home/admin/cert/www.domain.cn.key \
  --fullchain-file /home/admin/cert/www.domain.cn.pem \
  --reloadcmd "sudo service nginx force-reload"
```

说明：
- `--key-file` 指定私钥保存路径。
- `--fullchain-file` 指定包含证书链的证书文件路径。
- `--reloadcmd` 在证书更新后自动重载 Nginx。

## 第三步：配置 Nginx 使用新证书

编辑你的 Nginx 配置文件，加入 SSL 证书配置：

```nginx
server {
    listen 443 ssl;
    server_name domain.cn;

    ssl_certificate     /home/admin/cert/www.domain.cn.pem;
    ssl_certificate_key /home/admin/cert/www.domain.cn.key;

    # 其他配置...
}
```

重启 Nginx：
```bash
sudo service nginx restart
```

## 自动续期

`acme.sh` 默认会自动创建一个 crontab 任务，实现证书的自动续期，无需人工干预。你可以通过以下命令查看当前 crontab：

```bash
crontab -l
```

## 总结

通过 `acme.sh` 结合阿里云 DNS 的方式，可以实现证书申请、安装、续期的全自动化，极大地降低了运维成本，是中小企业和个人开发者部署 HTTPS 的利器。

