# 夜信 Night Letter — 缺陷审计与开发流程

> 审计日期：2026-06-22
> 审计范围：登录注册模块 + 共鸣角（广场）模块

---

## 一、缺陷审计：登录注册模块

### 严重级别定义

- **S1 阻塞级**：功能不可用或数据会丢失
- **S2 严重级**：功能有明显缺陷，影响核心体验
- **S3 一般级**：体验不佳但可用，属于体验优化
- **S4 建议级**：不影响功能，但存在设计或安全改进空间

### 缺陷清单

| # | 缺陷 | 级别 | 位置 | 问题描述 | 状态 |
|---|------|------|------|---------|------|
| A1 | 页面刷新丢失登录态 | S1 | `_getCurrentUser()` L1135 | `_cbUserCache` 是纯内存变量，页面刷新后归零。虽然 `onAuthStateChange` 监听器会恢复，但存在时序差——初始化时 `_checkOnboarding()` (L3678) 可能在监听器触发前就执行，导致已登录用户看到登录弹窗 | ✅ Sprint 1 已修复 |
| A2 | 无会话恢复机制 | S1 | 初始化代码 L1200-1238 | CloudBase 初始化后没有主动调用 `_cbGetSession()` 恢复会话。`_cbUserCache` 完全依赖异步监听器，首屏渲染时用户状态为空 | ✅ Sprint 1 已修复 |
| A3 | 云端同步无去重 | S2 | `syncDreamsToCloud()` L1611 | 每次登录都执行 `col.add()` 逐条添加，没有检查云端是否已存在相同 `localId`，重复登录会产生大量重复文档 | ⏳ Sprint 2 |
| A4 | 合并策略可能丢数据 | S2 | `mergeDreams()` L1644 | `map[d.id]=d` 后者覆盖前者。如果本地数据更新但云端未同步，登录后云端旧数据会覆盖本地新数据 | ⏳ Sprint 2 |
| A5 | 注册后未自动登录(v1) | S2 | `doRegister()` L1484 | v1 SDK 的 `signUpWithEmailAndPassword` 不会自动登录，代码虽然有 `_cbSignIn` 补登录逻辑，但如果 signIn 也失败（如网络波动），用户会卡在"注册成功但未登录"的状态 | ⏳ Sprint 2 |
| A6 | 注册无密码确认 | S3 | `doRegister()` L1463 | 没有"确认密码"输入框，用户可能因输入错误密码而注册后无法登录 | ⏳ Sprint 4 |
| A7 | 注册无邮箱验证 | S3 | `doRegister()` L1463 | 没有发送验证邮件的流程，任意格式正确的邮箱均可注册。CloudBase 控制台可配置是否强制验证 | ⏳ 待定 |
| A8 | 匿名登录双轨制 | S3 | `loginAnonymously()` L1521 | 成功时设 `_cbUserCache`，失败回退时设 `localStorage nightletter_session='__anon__'`。两套匿名状态判断逻辑（`_getCurrentUser()` vs `localStorage==='__anon__'`）容易不同步 | ✅ Sprint 1 已修复 |
| A9 | 登录/注册按钮状态混乱 | S3 | `toggleLoginMode()` L1393 | `loginBtn.onclick` 被动态赋值，但 HTML 中 `regBtn` (L946) 的 `display:none` 始终隐藏且无代码切换其显示状态，regBtn 实际上是死代码 | ⏳ Sprint 4 |
| A10 | 退出登录不提示数据状态 | S4 | `doLogout()` L1545 | 退出后直接显示登录弹窗，没有告知用户本地数据仍保留、云端数据不受影响 | ⏳ Sprint 4 |
| A11 | 无密码强度指示器 | S4 | `doRegister()` L1463 | 只有"至少6位"的校验，没有密码强度提示（弱/中/强） | ⏳ 待定 |

---

## 二、缺陷审计：共鸣角（广场）模块

| # | 缺陷 | 级别 | 位置 | 问题描述 |
|---|------|------|------|---------|
| B1 | 点赞无去重 | S2 | `likeDream()` L3580 | 没有检查当前用户是否已点赞过该梦境。每次点击都会触发 `cbDb.command.inc(1)`，用户可以无限次点赞，数据膨胀且不公平 |
| B2 | 评论者身份硬编码 | S2 | `doSubmitComment()` L3663 | 所有评论者显示为"匿名旅人"。`comments.push({user:'匿名旅人',...})` 没有使用当前用户的实际身份信息，无法区分不同用户的评论 |
| B3 | 评论硬限 3 条 | S3 | `doSubmitComment()` L3655 | `if(comments.length>=3)` 限制每条梦境只能有 3 条评论，对于活跃社区来说太严格，且没有解释为什么是 3 |
| B4 | 无分页/下拉加载 | S2 | `loadSquareDreams()` L3518 | 固定 `.limit(50)`，超过 50 条后老内容永远看不到。没有下拉加载或分页机制 |
| B5 | 广场标题仍为"梦境广场" | S3 | HTML L812 | `ns-t` 元素文本为"梦境广场"，如果已更名为"共鸣角"需要同步更新 UI 文本 |
| B6 | 无内容审核/过滤 | S3 | `doShareToSquare()` L3483 | 分享梦境时没有敏感词过滤或内容审核，任意文本直接发布到公开集合 |
| B7 | 无情绪筛选 | S3 | `renderSquareDreams()` L3538 | 广场页面没有按情绪分类筛选的功能，无法快速找到特定情绪的梦境 |
| B8 | 无排序选项 | S3 | `loadSquareDreams()` L3518 | 固定按 `createTime desc` 排序，没有"最热"（按点赞数排序）或"最多评论"选项 |
| B9 | 分享按钮对匿名用户可见 | S3 | HTML L812 | 广场顶部导航的分享按钮始终显示，匿名用户点击后才提示需要登录，体验不佳 |
| B10 | 无举报功能 | S4 | — | 社区功能缺少举报/屏蔽机制，无法处理不当内容 |
| B11 | 无搜索功能 | S4 | — | 无法在广场中搜索特定关键词的梦境 |
| B12 | 缺少"共鸣"按钮 | S3 | `renderSquareDreams()` L3571 | brainstorm-v2.md 创意 #13 提到用"共鸣"替代简单的"点赞"，但当前仍使用标准的点赞 ✨ 按钮，没有体现"共鸣"概念 |
| B13 | 广场空状态缺引导 | S4 | HTML L814-823 | 空状态只有"成为第一个分享的人吧"，缺少引导性说明（什么是广场、如何参与等） |

---

## 三、缺陷优先级矩阵

按照 Vibe Coding 方法论的 Impact/Effort 评估：

```
高影响 低投入（Quick Wins，立即做）
├── A1 页面刷新恢复会话       — 加 _cbGetSession() 即可
├── A2 初始化时主动获取会话    — 同上，一处改动
├── A9 清理死代码 regBtn      — 删除无用 HTML
└── B5 更新广场标题文本        — 一行改动

高影响 中投入（Big Bets，重点做）
├── A3 云端同步去重           — 需重写 syncDreamsToCloud
├── A4 合并策略优化           — 需重写 mergeDreams
├── B1 点赞去重              — 需加 likedBy 字段 + 查询
├── B2 评论者身份             — 需关联用户信息
└── B4 分页加载              — 需加 cursor 分页逻辑

中影响 低投入（Fill-in，顺手做）
├── A6 注册加确认密码          — 加一个 input
├── A8 统一匿名判断逻辑        — 合并两套判断
├── B7 情绪筛选              — 加 chip 筛选栏
└── B9 匿名用户隐藏分享按钮    — 条件渲染

中影响 中投入（可延后）
├── A7 邮箱验证              — 需 CloudBase 配置
├── B3 评论限制调整           — 改上限 + UI 说明
├── B6 内容过滤              — 需敏感词库
└── B8 排序选项              — 加排序 UI + 查询
```

---

## 四、开发流程设计

按照 Vibe Coding 三阶段方法论，将缺陷修复组织为 4 个迭代周期：

### Sprint 1：会话稳定性（1-2天） ✅ 已完成

> 目标：解决登录态丢失这个最影响体验的 S1 问题

**实际完成内容：**
- A1 ✅ 页面刷新恢复登录态 — `_checkOnboarding()` 移入 `_cbGetSession()` 回调内
- A2 ✅ 统一会话恢复机制 — SDK 优先 + localStorage JSON 回退，合并两个重复恢复块
- A8 ✅ 匿名登录双轨制统一 — 所有路径（登录/注册/匿名/离线）统一使用 JSON session 标记 + `_setUserCache()`
- 额外：`renderAccountSection()` 改用 `user.isAnonymous` 字段判断
- 额外：`_checkOnboarding()` 移除旧 `__anon__` 字符串依赖
- 额外：首页添加齿轮按钮进入设置页
- 额外：`loadDreamsFromCloud()` 添加 `.catch()` 优雅降级
- 额外：修正 CloudBase 环境 ID 拼写错误 (d3ga4y → d3g4qv)
- 额外：`_cbGetSession()` 添加调试日志

**迭代 Prompt 模板：**

```
## 任务
修复页面刷新后登录态丢失的问题。

## 上下文
- _getCurrentUser() 返回内存缓存 _cbUserCache（L1135）
- CloudBase 初始化后只注册了 onAuthStateChange 监听（L1219）
- 但首屏渲染时监听器可能还未触发，导致 _checkOnboarding() 认为用户未登录

## 约束
- 只修改 index.html
- 不要改动 CloudBase SDK 配置
- 不要改动 doLogin/doRegister/doLogout 的核心逻辑

## 实现要求
1. CloudBase 初始化成功后，立即调用 _cbGetSession() 获取当前会话
2. 将结果写入 _cbUserCache
3. 如果已有会话，跳过登录弹窗
4. 添加 console.log 日志以便调试

## 验收标准
- [ ] 已登录用户刷新页面后，不会再次弹出登录框
- [ ] 未登录用户刷新后正常显示登录框
- [ ] 匿名登录后刷新不会弹出登录框
- [ ] 控制台显示 [Auth] session restored 日志
```

**验证步骤：**
1. 登录账号 → 刷新页面 → 确认直接进入首页
2. 匿名登录 → 刷新 → 确认直接进入首页
3. 未登录 → 刷新 → 确认显示登录弹窗
4. 退出登录 → 刷新 → 确认显示登录弹窗

### Sprint 2：数据同步安全（2-3天） ✅ 已完成

> 目标：解决重复同步和数据覆盖问题

**实际完成内容：**
- A3 ✅ syncDreamsToCloud 去重 — 先查询云端已有 localId，只上传不存在的，跳过已存在
- A4 ✅ mergeDreams 冲突策略 — 按 ts 时间戳比较，保留更新的那条（本地更新不被云端旧数据覆盖）
- A5 ✅ doRegister 注册后登录重试 — v1 SDK signIn 失败后等待 1s 重试一次，错误提示区分"注册成功但登录失败"
- 额外：syncDreamsToCloud 和 loadDreamsFromCloud 添加 DATABASE_COLLECTION_NOT_EXIST 优雅降级
- 额外：同步和合并操作添加 console.log 日志

**迭代 Prompt 模板：**

```
## 任务
重写 syncDreamsToCloud 和 mergeDreams，增加去重和冲突保护。

## 上下文
- syncDreamsToCloud() 每次调用都 add 所有梦境（L1611）
- mergeDreams() 用后覆盖前的 map 策略（L1644）
- saveDream() 在本地保存后也会调用 add（L1072）

## 约束
- 只修改 index.html
- 不要改动 CloudBase 数据库安全规则
- 使用 localId 字段作为去重键

## 实现要求
1. syncDreamsToCloud: 先查询云端已有的 localId，只上传不存在的
2. mergeDreams: 按 ts 比较，保留更新的那条
3. saveDream 中的 add 调用保持不变（单条新增不需要去重）

## 验收标准
- [ ] 同一账号重复登录后，云端 dreams 集合不产生重复文档
- [ ] 本地修改过的梦境不会被云端旧数据覆盖
- [ ] 新写的梦境能正确同步到云端
- [ ] 切换设备后能看到所有梦境
```

### Sprint 3：广场基础体验（3-4天）

> 目标：修复广场模块的 S2/S3 缺陷，提升社区体验

**拆分为 3 个子任务：**

**3a. 点赞去重 + 评论身份：**

```
## 任务
修复广场点赞无去重和评论身份硬编码问题。

## 实现要求
1. likeDream: 在 publicDreams 文档中增加 likedBy 数组字段（存用户 uid）
2. 点赞前检查当前用户 uid 是否已在 likedBy 中
3. 已点赞时按钮显示高亮状态，点击取消点赞
4. doSubmitComment: 用当前用户的邮箱前缀或"匿名旅人N"替代硬编码
5. 渲染时区分"我的评论"和"他人的评论"

## 验收标准
- [ ] 同一用户对同一梦境只能点赞一次
- [ ] 再次点击已点赞的梦境可取消点赞
- [ ] 评论显示的用户名与当前登录身份关联
- [ ] 匿名用户的评论显示为"匿名旅人"
```

**3b. 分页加载：**

```
## 任务
为广场添加下拉加载更多功能。

## 实现要求
1. 首次加载 20 条（不是 50 条），记录最后一条的 createTime
2. 滚动到底部时自动加载下一批 20 条
3. 显示"加载中"动画
4. 没有更多数据时显示"已经到底了"

## 验收标准
- [ ] 首屏加载 20 条，页面流畅
- [ ] 滚动到底部自动加载更多
- [ ] 加载时显示 spinner 动画
- [ ] 所有数据加载完后显示提示
```

**3c. 情绪筛选 + 排序：**

```
## 任务
为广场添加情绪筛选标签和排序切换。

## 实现要求
1. 顶部添加情绪 chip 筛选栏（全部/平静/不安/喜悦/想念/困惑/未知）
2. 添加排序切换（最新/最热）
3. 筛选和排序在前端完成（数据已加载到 squareDreams 数组）
4. 切换时平滑过渡动画

## 验收标准
- [ ] 点击情绪 chip 只显示对应情绪的梦境
- [ ] "最热"排序按 likes 降序
- [ ] 筛选 + 排序可组合使用
- [ ] 切换时有过渡动画
```

### Sprint 4：体验打磨（2天）

> 目标：清理 S3/S4 级别的小问题，打磨细节

**修复清单：**

```
## 任务
批量修复以下小问题：

1. A9: 删除 regBtn 死代码（L946），确保 toggleLoginMode 只操作 loginBtn
2. A6: 注册表单增加"确认密码"输入框
3. A8: 统一匿名判断逻辑，只保留 _getCurrentUser() 一种方式
4. A10: 退出登录时显示"本地数据已保留"提示
5. B5: 如果确定叫"共鸣角"，更新所有 UI 文本
6. B9: 匿名用户隐藏广场的分享按钮
7. B3: 评论上限从 3 调到 10，并在 UI 上说明

## 验收标准
- [ ] 注册时有确认密码框，两次不一致提示错误
- [ ] 退出登录时显示确认弹窗含"本地数据保留"文案
- [ ] 匿名模式下广场不显示分享按钮
- [ ] 评论最多 10 条，超出时提示"留言已满"
```

---

## 五、迭代顺序与依赖关系

```
Sprint 1（会话稳定）
    │
    ├─ 无前置依赖，可立即开始
    ├─ 产出：刷新不丢登录态
    │
    ▼
Sprint 2（数据安全）
    │
    ├─ 依赖 Sprint 1（需要稳定的用户身份）
    ├─ 产出：云端同步不丢不重
    │
    ▼
Sprint 3（广场体验）
    │
    ├─ 依赖 Sprint 1 + 2（需要稳定身份 + 数据安全）
    ├─ 3a/3b/3c 可并行开发
    ├─ 产出：广场可用性达到产品标准
    │
    ▼
Sprint 4（体验打磨）
    │
    ├─ 依赖 Sprint 3（广场功能稳定后再打磨）
    ├─ 产出：细节完善，可发布版本
```

---

## 六、每个 Sprint 的标准流程

按 Vibe Coding 编码迭代的五个原则：

```
1. 明确范围 → 只改 index.html 的指定行号区间
2. 小步提交 → 每个子任务完成后立即 git commit
3. 验证步骤 → 本地 python -m http.server 8765 测试
4. 回归检查 → 确认修改不影响其他功能
5. 文档同步 → 更新 PROJECT-STATUS.md 的功能状态
```

**Git 提交格式：**

```
Sprint 1: fix: restore login session on page refresh
Sprint 2: fix: deduplicate cloud sync and improve merge strategy
Sprint 3a: feat: add like dedup and user identity in square comments
Sprint 3b: feat: add infinite scroll pagination to square
Sprint 3c: feat: add emotion filter and sort options to square
Sprint 4: fix: polish auth and square UX details
```

---

## 七、完成后验证清单

所有 Sprint 完成后，逐条验证：

```
登录注册：
- [ ] 邮箱注册 → 自动登录 → 数据同步到云端
- [ ] 邮箱登录 → 刷新页面不丢登录态
- [ ] 匿名登录 → 刷新不丢登录态
- [ ] 切换账号 → 数据正确隔离
- [ ] 退出登录 → 显示登录弹窗 + 提示数据保留
- [ ] 注册时密码不一致提示错误

共鸣角/广场：
- [ ] 分享梦境到广场 → 显示在列表中
- [ ] 点赞/取消点赞 → 计数正确，不可重复点
- [ ] 评论梦境 → 显示当前用户身份
- [ ] 滚动加载更多 → 流畅不卡顿
- [ ] 按情绪筛选 → 只显示对应情绪梦境
- [ ] 按最新/最热排序 → 顺序正确
- [ ] 匿名用户 → 不显示分享按钮，点赞/评论提示登录

数据安全：
- [ ] 重复登录不产生重复文档
- [ ] 本地修改不被云端旧数据覆盖
- [ ] 切换设备后数据完整
```
