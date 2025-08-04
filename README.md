# VS Code 轻量级智能浏览器插件  
> 代号：**ViaCode**  
> 设计目标：在 VS Code 内部提供「类 Via 浏览器」体验，体积 < 500 KB，所有渲染复用用户本机 Chrome/Edge，插件自身仅保留智能控制层与极简 UI。

---

## 1. 产品定位
| 维度 | 说明 |
|---|---|
| 形态 | VS Code 插件（.vsix） |
| 核心 | 超薄浏览器壳 + 智能导航层 |
| 体积 | ≤ 500 KB（零运行时依赖） |
| 代理 | 插件级代理（仅插件内部流量，不修改 VS Code 全局设置） |
| 智能 | 基于本地缓存 + 轻量规则的“预测式导航” |

---

## 2. 架构总览

```
┌────────────────────────────┐
│ VS Code 宿主               │
│  ├─ Webview Panel (iframe) │  ← 显示网页（复用本机 Chrome）
│  └─ Extension Host        │
│      ├─ Proxy Server      │  ← Node.js 本地代理 (127.0.0.1:随机端口)
│      ├─ Smart Router      │  ← 规则 / 缓存 / 预测
│      └─ Commands & API    │  ← VS Code 命令、状态栏图标
└────────────────────────────┘
```

---

## 3. 极简技术栈
| 模块 | 技术 | 大小 |
|---|---|---|
| 入口 | `extension.ts`（TypeScript） | 15 KB |
| 面板 | Webview + 静态 HTML/JS | 35 KB |
| 代理 | `http-proxy-middleware` 单文件 | 150 KB |
| 缓存 | 内存 LRU + IndexedDB | 0 KB（浏览器自带） |
| 图标 | 512×512 PNG | 10 KB |
| 总计 | — | **≈ 480 KB** |

---

## 4. 核心功能清单

### 4.1 超薄浏览器
- **无地址栏**：像 Via 一样，地址栏隐藏在顶部 1 px 细条，双击或 `Ctrl+L` 弹出。
- **手势**：左右滑前进/后退；通过 Webview 的 `postMessage` 实现。
- **暗黑模式**：自动跟随 VS Code 主题。

### 4.2 插件级代理（不污染全局）
- 启动时在 **随机空闲端口** 建本地代理：
  ```ts
  const proxy = createProxyServer({
    target: userUpstream,   // 用户配置
    changeOrigin: true,
    onProxyReq: addAuthHeader
  });
  ```
- iframe 的 `src` 指向 `http://localhost:{port}/proxy?url=...`，所有请求经此代理。
- VS Code 自身、其他插件、Marketplace 均 **不受影响**。

### 4.3 智能导航（Smart Router）
| 能力 | 触发方式 | 技术实现 |
|---|---|---|
| 预加载下一页 | 滚动到 80 % 时 | 解析 `<link rel="next">` 并提前 `fetch` 到缓存 |
| 搜索直达 | 地址栏输入 `g 关键词` | 本地映射表 `g` → Google，`b` → Bing |
| 离线包 | 检测到 PWA manifest | 一键 `beforeinstallprompt` |
| 快捷命令 | `Ctrl+Shift+P` | `via.open <url>` / `via.proxy.toggle` |

---

## 5. 目录结构（发布版）
```
.vscode/
├── extension.js          ← esbuild 单文件
├── media/
│   ├── index.html        ← 40 行 HTML
│   ├── via.js            ← 手势 + 暗黑 + 通信
│   └── styles.css
├── package.json
└── README.md
```

---

## 6. 开发三步曲（2 小时完成 MVP）

1. **生成骨架**
   ```bash
   yo code
   # 选 TypeScript，名字 via-browser
   ```

2. **构建无依赖单文件**
   ```bash
   npm i -D esbuild
   npx esbuild src/extension.ts --bundle --platform=node --outfile=extension.js
   ```

3. **打包 & 本地测试**
   ```bash
   npx vsce package
   code --install-extension via-browser-0.0.1.vsix
   ```

---

## 7. 后续可扩展（保持 500 KB 以内）
- 书签：存到 `globalState`，增量 < 5 KB。  
- 广告过滤：Adblock Plus 规则本地解析，规则文件外链按需下载。  
- 脚本注入：用户脚本 `< 1 KB` 直接写入 `settings.json`。

---

## 8. 一句话总结
**ViaCode = Via 浏览器思想 + VS Code 生态**  
→ 把“浏览器本体”交给用户电脑，插件只做“遥控器 + 智能大脑”，永远保持 < 500 KB。
