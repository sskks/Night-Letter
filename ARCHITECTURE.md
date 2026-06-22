# ARCHITECTURE.md — 夜信 Night Letter 架构草案

> 版本 1.0 | 2026-06-22

---

## 1. 系统架构总览

```
┌─────────────────────────────────────────────┐
│              浏览器 (PWA)                     │
│  ┌───────────────────────────────────────┐   │
│  │           index.html (单文件)          │   │
│  │  ┌─────────┬─────────┬────────────┐   │   │
│  │  │  HTML   │  CSS    │  JavaScript │   │   │
│  │  │ (结构层) │ (样式层) │  (逻辑层)   │   │   │
│  │  └─────────┴─────────┴────────────┘   │   │
│  └───────────────────────────────────────┘   │
│         │ localStorage │ Canvas API          │
│         ▼              ▼                     │
│  ┌──────────────┐ ┌──────────┐              │
│  │ 本地数据持久化 │ │ 动画渲染  │              │
│  └──────────────┘ └──────────┘              │
└─────────────┬───────────────────────────────┘
              │ HTTPS
              ▼
┌─────────────────────────────────────────────┐
│           腾讯云 CloudBase                    │
│  ┌────────────┐ ┌──────────────────────┐    │
│  │  认证服务   │ │    数据库 (MongoDB)    │    │
│  │ 邮箱/匿名   │ │ dreams / comments     │    │
│  └────────────┘ └──────────────────────┘    │
└─────────────────────────────────────────────┘
              │
              ▼ (外部 API)
┌─────────────────────────────────────────────┐
│           通义千问 qwen-turbo                  │
│           文本生成 API                        │
└─────────────────────────────────────────────┘
```

---

## 2. 模块划分

index.html 内按功能分区，逻辑模块如下：

| 模块 | 行号范围(约) | 职责 |
|------|-------------|------|
| **样式层** | 1-640 | CSS 变量、组件样式、动画、响应式 |
| **结构层** | 640-960 | HTML 骨架，8 个页面区块 |
| **核心逻辑** | 960-1250 | CloudBase 初始化、认证封装、数据 CRUD |
| **数据层** | 1250-1400 | localStorage 操作、数据迁移、情绪检测 |
| **AI 层** | 1400-1500 | 千问 API 调用、多人格系统、降级回复 |
| **页面逻辑** | 1500-2500 | 各页面的交互逻辑（登录、写信、对话、历史等） |
| **动画层** | 2500-3400 | 极光 Canvas、星图 Canvas、入场动画 |
| **初始化** | 3400-3700 | 页面加载、会话恢复、事件绑定 |

---

## 3. 数据模型

### Dream（梦境记录）

```json
{
  "id": 1719024000000,        // 时间戳 ID
  "text": "梦到了一片花海...",   // 梦境正文
  "emotion": "平静",           // 情绪标签：不安/想念/困惑/平静/喜悦/未知
  "type": "梦境",              // 类型标签：梦境/白日梦/灵感
  "ts": "2026-06-22T08:00:00.000Z"  // ISO 时间戳
}
```

### Checkin（每日签到）

```json
{
  "2026-06-22": {
    "mood": "平静",
    "ts": "2026-06-22T07:30:00.000Z"
  }
}
```

### SquarePost（广场帖子）

```json
{
  "_id": "auto-generated",
  "text": "梦到了...",
  "emotion": "平静",
  "author": "匿名旅人",
  "likes": 5,
  "comments": ["好温暖的梦"],
  "ts": "2026-06-22T08:00:00.000Z"
}
```

### User（用户）

```json
{
  "email": "user@example.com",
  "isAnonymous": false,
  "uid": "cloudbase-uid"
}
```

---

## 4. localStorage 键值设计

| Key | 类型 | 说明 |
|-----|------|------|
| `nightletter_dreams` | Array | 匿名用户的梦境数据 |
| `nightletter_dreams_{email}` | Array | 登录用户的梦境数据（按邮箱隔离） |
| `nightletter_checkins` | Object | 签到数据（日期为 key） |
| `nightletter_persona` | String | 当前对话人格（gentle/analyst/poet） |
| `nightletter_maxturns` | Number | 对话轮次上限 |
| `nightletter_session` | Object | 当前登录会话信息 |
| `nightletter_streak` | Object | 连续天数缓存（已弃用，由 calcStreak 实时计算） |

---

## 5. 外部依赖

| 依赖 | 版本 | 用途 | 加载方式 |
|------|------|------|---------|
| CloudBase SDK | 3.4.7 | 认证 + 数据库 | `<script src>` CDN |
| 通义千问 API | qwen-turbo | AI 文本生成 | fetch REST API |
| Noto Serif/Sans SC | Google Fonts CDN | 中英文字体 | `@import` CSS |

---

## 6. 通信流

### 写信流程
```
用户输入 → 情绪检测(detectEmotion) → saveDream() → localStorage
                                               ↓ (如已登录)
                                          syncDreamsToCloud() → CloudBase
```

### 对话流程
```
用户输入 → chatTurn 检查 → callQwenAPI() → 千问 API → 显示气泡
                              ↓ (超时/失败)
                         personaFallbacks → 人格化本地回复
```

### 登录流程
```
输入邮箱密码 → _cbSignIn() / _cbSignUp() → CloudBase Auth
                                            ↓ (成功)
                    _migrateStoreIfNeeded() → 数据迁移 → 刷新 UI
```
