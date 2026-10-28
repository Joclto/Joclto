# 刘飞天 · Feitian Liu

> 制冷及低温工程硕士 → 能源算法 → AI 工程。现居深圳，供职于万物云。
>
> 关注方向：把能源、制造等传统行业的业务问题，落成能长期稳定运行的软件系统。

---

## 🧰 技术栈

**语言**

![Python](https://img.shields.io/badge/-Python-3776AB?style=flat-square&logo=python&logoColor=white)
![Rust](https://img.shields.io/badge/-Rust-000000?style=flat-square&logo=rust&logoColor=white)
![TypeScript](https://img.shields.io/badge/-TypeScript-3178C6?style=flat-square&logo=typescript&logoColor=white)
![Kotlin](https://img.shields.io/badge/-Kotlin-7F52FF?style=flat-square&logo=kotlin&logoColor=white)
![SQL](https://img.shields.io/badge/-SQL-4479A1?style=flat-square&logo=postgresql&logoColor=white)

- **Python**：asyncio / 多进程、PySide6 桌面端、Playwright 自动化、数据处理
- **Rust**：egui、Tauri 桌面应用、系统级工具
- **TypeScript / React**：前端界面与实时看板
- **Kotlin**：Android（Jetpack Compose）端开发

**后端与基础设施**

- PostgreSQL、WebSocket 实时通道、REST API
- Docker 部署、进程守护、日志分级与告警降噪
- Windows 无人值守运维：计划任务、看门狗、自愈脚本

**工程实践**

- Git 分支/PR 工作流，代码走审查后合并
- PyInstaller / Tauri 打包分发
- 中文注释、可回溯的提交信息

---

## 🚀 代表项目

### [InfoPush](https://github.com/Joclto/InfoPush) · 自托管消息推送服务

> 一条 HTTP 请求，把消息推到你的所有设备。

- HTTP 推送 + WebSocket 实时投递双通道
- 覆盖 Android、Windows 桌面，支持 Docker 一键部署
- PostgreSQL 持久化，端到端可控，不依赖第三方推送平台
- 技术栈：Kotlin / Python / Rust / Tauri / PostgreSQL · MIT

### [service-pilot](https://github.com/Joclto/service-pilot) · Windows 服务管理器

> 把 Windows 后台服务的管理，做成一块能实时看的面板。

- Tauri 2 + React 19 + Rust 构建的原生桌面应用
- 启动 / 停止 / 监控 / 检查 Windows 后台服务，实时状态刷新
- 面向真实运维场景：不用再跟 services.msc 和命令行搏斗
- 技术栈：TypeScript / React / Rust / Tauri · MIT

---

## 🧠 我的工程习惯

**1. 规划与执行分离**

复杂任务不交给一个模型从头做到尾：

- **Hermes**（AI 助理）负责需求拆解、方案设计、任务卡编写与代码审查；
- **Claude Code** 负责具体编码实现；
- 人只做两件事：**仲裁方案，验收结果**。

好处是每一步都有独立的第二双眼睛：写的人不审自己，审的人不动手。

**2. 最小改动优先（ponytail）**

日常启用 `ponytail`（lazy senior dev 模式）约束 AI 的产出：

- 能一行解决，不写两行；
- 删除优于新增；
- 不写没人要的抽象、配置项和"以后可能用得上"的代码；
- 但风险逻辑必须带测试——懒的是方案，不是质量。

**3. 证据驱动，跑过才算完成**

不靠"看起来对"：测试通过、真实输出、日志核验之后，才算做完一件事。写代码如此，运维排障也如此。

**4. 把重复劳动变成流水线**

能自动化的不做第二次手工：自举式自动开发流水线、定时巡检、看门狗自愈——人只在需要判断的地方出现。

**5. 克制的可观测性**

日志分级，告警只推送真正需要人介入的事。**告警风暴是缺陷，不是勤奋。**

---

<sub>这个主页会随我的工程习惯一起更新。</sub>
