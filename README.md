# SJTU VPN Ubuntu 配置备忘

在 Ubuntu 上配置上海交通大学 VPN（IKEv2/IPsec + EAP-PEAP，学生 jAccount），并记录与 Tailscale 共同使用时的路由排除问题及解决方案。

## 内容

- **[SJTU_VPN_Ubuntu_Guide.md](./SJTU_VPN_Ubuntu_Guide.md)** — 完整配置备忘：
  - strongSwan（`charon-systemd` + `swanctl`）安装与配置（证书、凭据、连接、断开、排障）
  - 与 Clash 共存的注意事项
  - 与 Tailscale 混用时，VPN 路由表（table 220）与 Tailscale 路由表（table 52）冲突导致的 ping/SSH 不通问题，及通过 `throw` 路由排除 Tailscale 网段的解决方案

## 安全说明

本文档只含配置示例与占位符，**不包含真实凭据**。含明文 jAccount 密码的真实配置文件（如 `/etc/swanctl/conf.d/sjtu.conf`）请勿提交到仓库或公开分享。