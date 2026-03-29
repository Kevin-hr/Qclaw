# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

**Qclaw** 是一个 Electron 桌面应用，作为 OpenClaw 的安装配置向导 —— 让用户无需命令行即可配置 AI 模型和 IM 渠道（飞书/钉钉/QQ/企微）。

| 组件 | 技术栈 |
|------|--------|
| 桌面框架 | Electron |
| 前端 | React 18 + TypeScript |
| 构建工具 | Vite + vite-plugin-electron |
| UI 组件库 | Mantine 8 + Tailwind CSS |
| 打包工具 | electron-builder |

## 常用命令

```bash
npm install           # 安装依赖
npm run dev           # 启动开发服务器（附带 Electron）
npm run build         # 构建并打包生产版本
npm run build:app     # 仅构建 TypeScript + Vite（不打包）
npm test              # 运行 Vitest 测试
npm run typecheck     # TypeScript 类型检查
npm run test:model-regression  # 模型回归测试
```

## 架构概览

```
┌─────────────────────────────────────────────────────────────┐
│                        主进程 (Main Process)                 │
│  electron/main/                                              │
│  ├── index.ts           # 入口文件，窗口生命周期管理          │
│  ├── ipc-handlers.ts    # IPC 消息处理器（主↔渲染通信）      │
│  ├── cli.ts             # OpenClaw CLI 封装层                │
│  ├── openclaw-*.ts      # OpenClaw 服务模块（认证/升级等）   │
│  ├── gateway-*.ts       # 网关（Gateway）生命周期管理        │
│  ├── feishu-*.ts        # 飞书集成（安装/诊断/插件状态）     │
│  └── node-*.ts          # Node.js 运行时管理                │
├─────────────────────────────────────────────────────────────┤
│                      预加载脚本 (Preload Script)             │
│  electron/preload/index.ts                                   │
│  通过 window.api 暴露安全的 IPC 桥接通道                      │
├─────────────────────────────────────────────────────────────┤
│                      渲染进程 (Renderer Process)              │
│  src/                                                       │
│  ├── App.tsx             # 根组件，应用状态机               │
│  ├── pages/              # 引导向导页面 + Dashboard 页面     │
│  ├── components/         # 可复用 UI 组件                    │
│  ├── lib/                # 业务逻辑（渠道/提供商注册）        │
│  └── shared/             # 共享类型、策略、工具函数           │
└─────────────────────────────────────────────────────────────┘
```

### IPC 通信机制

主进程通过 `window.api` 向渲染进程暴露 API（在 preload 中定义）。

常用 API：
| API | 说明 |
|-----|------|
| `window.api.checkOpenClaw()` | 检测 OpenClaw 安装状态 |
| `window.api.getModelCapabilities()` | 获取模型列表 |
| `window.api.gatewayHealth()` | 检查网关运行状态 |
| `window.api.repairIncompatiblePlugins()` | 自动修复损坏的插件 |

### 应用状态机（App.tsx）

```
welcome → env-check → gateway-bootstrap
                          ↓
              ┌─────────┴─────────┐
              ↓                   ↓
           setup              dashboard
    (api-keys → channel-connect → pairing-code)
```

**状态流转说明**：
1. `welcome` - 欢迎页，用户确认后进入环境检测
2. `env-check` - 检测 Node.js 和 OpenClaw CLI，缺失时自动安装
3. `gateway-bootstrap` - 网关引导，准备 Dashboard 入口
4. `setup` - 向导模式：配置 AI 提供商 → 连接 IM 渠道 → 配对码
5. `dashboard` - 管理面板，可访问聊天/渠道/模型/技能/设置

### 页面路由（Dashboard 模式）

| 路由 | 组件 | 功能 |
|------|------|------|
| `/` | Dashboard | 主状态视图 |
| `/chat` | ChatPage | 直接对话面板 |
| `/channels` | ChannelsPage | IM 渠道管理 |
| `/models` | ModelsPage | 模型提供商配置 |
| `/skills` | SkillsPage | 技能扩展管理 |
| `/settings` | SettingsPage | 应用设置、数据备份 |

## 测试

测试使用 Vitest，测试文件分散在 `__tests__` 目录中：

| 目录 | 说明 |
|------|------|
| `src/components/__tests__/` | UI 组件测试 |
| `src/pages/__tests__/` | 页面组件测试 |
| `src/lib/__tests__/` | 业务逻辑测试 |
| `electron/main/__tests__/` | 主进程模块测试 |

运行指定测试文件：
```bash
npx vitest run src/components/__tests__/specific.test.tsx
```

## 关键设计模式

### OpenClaw CLI 集成
所有 OpenClaw CLI 调用都经由 `electron/main/cli.ts` 封装，该模块负责：
- CLI 输出解析
- OAuth 流程处理
- 错误转换

### 渠道/提供商注册表
- `src/lib/openclaw-channel-registry.ts` - IM 渠道定义（飞书/钉钉/QQ/企微）
- `src/lib/openclaw-provider-registry.ts` - AI 模型提供商配置

### Dashboard 入口引导
`src/shared/dashboard-entry-bootstrap.ts` 根据已有的 OpenClaw 配置判断：
- 直接进入 Dashboard（已配置）
- 进入 setup 向导（未配置）

## VS Code 调试

设置环境变量以启用 VS Code 调试：
```bash
VSCODE_DEBUG=1 VSCODE_DEBUG_LAUNCH=1 npm run dev
```

这会让 Electron 调试器附加到主进程。
