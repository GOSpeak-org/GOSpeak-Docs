# GOSpeak-Docs — VitePress 文档站

面向**部署用户**的文档站（`@gospeak/docs`），与仓库根 `AGENTS.md`（面向开发者）分工不同。

## 结构

```
guide/        快速开始、导览、桌面端、配置、域、权限、存储、邮件、OAuth、FAQ
deploy/       部署总览、compose、单容器、单二进制、反代 HTTPS、数据库、监控、排障
sfu/          语音后端总览、自托管、云
reference/    api.md、errors.md、events.md、plugin-api.md
architecture/ 架构
public/       logo、favicon、screenshots/
.vitepress/   config.ts（nav + sidebar）、theme/
```

## 命令

```bash
pnpm --filter @gospeak/docs dev      # 本地预览
pnpm --filter @gospeak/docs build    # 构建静态站
```

## 约定

- `base`：开发 `/`，生产 `/GOSpeak-Docs/`（GitHub Pages 子路径，与 `.vitepress/config.ts` 一致）。
- **新增页面必须同时改 `.vitepress/config.ts` 的 sidebar**，否则页面不可达。
- 截图放 `public/screenshots/`，引用用绝对路径。
- `reference/` 是对代码的镜像（错误码、事件、接口），**改代码要同步更新**；错误码以 `server/server/internal/pkg/errors.go` 为准。
- 部署细节的**唯一权威是 `server/deploy/DEPLOY.md`**（位于 Server 子模块）；本站只做面向用户的转述，出现冲突以 DEPLOY.md 为准。
- 语言 zh-CN；不写与当前版本不符的历史方案。
