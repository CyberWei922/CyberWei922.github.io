# cyberwei922.github.io

> 个人主页源码 — 安徽大学微电子科学与工程 2024 级

**[🌐 在线访问 →](https://cyberwei922.github.io)**

---

## 关于

微电子专业学生，同时对计算机系统有自驱探索。自学前端开发，擅长 Linux 系统、网络架构与 Android 底层，热衷于把硬件榨到极限。正备战全国大学生电子设计竞赛。

---

## 技术栈

| 领域 | 技术 |
|------|------|
| 前端 | HTML · CSS · JavaScript |
| 编程语言 | C · Python · Verilog（自学中）|
| Linux / 系统 | Ubuntu Server · iptables/nftables · Shell · Docker |
| 网络与路由 | 旁路由架构 · 透明代理 · 固件逆向 · 策略路由 |
| Android 底层 | Magisk / KernelSU · LSPosed · Fastboot/ADB |
| 硬件 / EE | 微电子专业课 · FPGA · 电赛备赛中 |

---

## 项目记录

### 小米 BE3600 固件逆向与 ShellClash 部署 `2024-10`
利用官方固件鉴权漏洞强制开启 SSH，在原生系统上部署 ShellClash，修改 iptables/nftables 实现全局透明代理，保留原厂 Wi-Fi 硬件加速。  
`Linux` `网络` `固件逆向`

### HP T630 瘦客机服务器化与旁路由架构 `2024-11`
将低功耗 x86 瘦客机部署为家用服务器，解决 UEFI 引导与多盘分区问题，调优旁路由网关架构消除流量环路，实现透明代理流控。  
`Linux` `网络` `服务器`

### Android 底层魔改：Nothing OS / HyperOS 深度定制 `2024-12`
针对多设备（Nothing Phone 2、小米 13 Ultra）适配 KernelSU/Magisk Root，部署 LSPosed 框架，修复国内 FCM 推送断连问题，大幅提升续航与通知时效。  
`Android` `KernelSU` `LSPosed`

### 自制网页工具集 `2023-09`
自学前端三件套，为班级写了随机点名系统、晚自习倒计时时钟等工具，在实际场景中持续使用。  
`HTML` `CSS` `JavaScript`

---

## 网站结构

```
.
├── index.html          # 单页应用主体（主页 + 博客 + 文章）
├── data/
│   ├── skills.json     # 技能数据
│   ├── projects.json   # 项目数据
│   ├── gear.json       # 设备数据
│   └── strava.json     # 骑行数据（CI 自动同步）
└── posts/              # Markdown 博客文章
```

博客文章使用 GitHub API 动态加载，Strava 骑行数据通过 GitHub Actions 定时同步。

---

## 联系

- GitHub：[@CyberWei922](https://github.com/CyberWei922)
- Strava：[CyberWei](https://www.strava.com/athletes/152238954)
- Email：2206064568@qq.com
