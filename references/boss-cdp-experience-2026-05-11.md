# BOSS直聘 CDP 自动化实测经验总结

> **文档版本**：v1.0
> **总结日期**：2026-05-12
> **实测时间**：2026-05-11（13:46-22:55）
> **目标**：为 web-access + kozsail-boss-hiring-assistant 技能套件优化提供完整依据

---

## 一、项目背景

### 1.1 业务目标

深圳科臻赛（KOZSail）HR 团队需要通过 BOSS 直聘平台批量筛选 AI 产品经理等岗位候选人。当前流程为 HR 手动逐一查看推荐页、读简历、收藏、打招呼，效率极低。

### 1.2 三层招聘架构

```
Layer1：BOSS直聘站内粗筛（WorkBuddy + CDP 浏览器控制）
  ↓ 筛选通过的候选人
Layer2：飞书深度精筛（Flask + LLM，AI 评估+沟通）
  ↓ 评估结果
Layer3：HR 汇报决策
```

### 1.3 技术路线

- 使用 Chrome DevTools Protocol (CDP) 代理（端口 3456）控制浏览器
- 通过 `web-access` 技能执行 eval/clickAt/navigate/screenshot
- 通过 `kozsail-boss-hiring-assistant` 技能提供招聘领域知识

---

## 二、页面架构实测

### 2.1 推荐页三层 iframe 嵌套结构

```
/web/chat/recommend（父页面/shell）
  └─ .frame-box iframe → /web/frame/recommend/（推荐列表 iframe）
       ├─ .card-item（候选人卡片，约15个/页）
       │    ├─ .name（姓名，可点击打开浮窗）
       │    ├─ .btn-greet（打招呼按钮）
       │    ├─ .button-list（收藏按钮）
       │    └─ .btn-doc（简历按钮）
       └─ .boss-dialog__wrapper.dialog-lib-resume（简历浮窗，在 iframe 内渲染）
            ├─ .resume-right-side（经历概览，DOM 文本，约320字）
            ├─ .resume-detail-wrap > iframe → /web/frame/c-resume/（简历详情 canvas）
            └─ .dialog-footer（操作按钮区）
                 ├─ .like-icon-and-text（收藏）
                 ├─ .btn-quxiao（不合适）
                 ├─ .btn-report（举报）
                 ├─ .btn-coop-forward（转发牛人）
                 └─ .resumeGreet > .btn-greet（打招呼）
```

### 2.2 关键发现

| 发现 | 说明 |
|------|------|
| iframe 同源可访问 | 父页面可通过 `iframe.contentDocument` 读取 iframe 内容 |
| iframe 不可独立使用 | 单独打开 `/web/frame/recommend/` 时，点击候选人名字无法打开浮窗（浮窗依赖父页面 Vue 上下文） |
| 简历详情是 canvas | 嵌套 iframe `/web/frame/c-resume/` 内简历用 canvas 渲染，`innerText` 为空，需截图或 OCR |
| 经历概览是 DOM | 浮窗 `.resume-summary` 有 DOM 文本约320字，可直接提取 |

---

## 三、CDP 操作验证矩阵

### 3.1 完整验证结果

| # | 操作 | 方案 | 结果 | 备注 |
|---|------|------|------|------|
| 1 | 读取候选人卡片 | `iframe.contentDocument.querySelectorAll(".card-item")` | ✅ 可行 | 同源访问 |
| 2 | 提取卡片文本 | `card.querySelector(".name").innerText` + base64 编码 | ✅ 可行 | 中文必须 base64 |
| 3 | 点击名字打开浮窗 | 父页面创建 marker + clickAt | ✅ 可行 | pointer-events:none 的 marker |
| 4 | 读取浮窗简历 | `iframe.contentDocument.querySelector(".resume-summary")` | ✅ 可行 | 仅经历概览 |
| 5 | 读取完整简历 | 嵌套 iframe canvas 截图/OCR | ⚠️ 未测试 | canvas 渲染，innerText 空 |
| 6 | 推荐页卡片打招呼 | clickAt + marker | ✅ 3人成功 | 早期验证 |
| 7 | 浮窗内打招呼（clickAt） | 父页面 marker + clickAt | ❌ 不可行 | Vue 组件不响应 isTrusted 穿透事件 |
| 8 | 浮窗内打招呼（el.click） | `iframe.contentDocument.querySelector(".btn-greet").click()` | ✅ API成功但❌触发风控 | 账号被限24h |
| 9 | 浮窗内收藏（clickAt） | 父页面 marker + clickAt | ⚠️ 未验证 | 预期同打招呼 |

### 3.2 点击方案对比

| 方案 | isTrusted | 能否打开浮窗 | 能否触发浮窗内按钮 | 风控风险 |
|------|-----------|-------------|-------------------|----------|
| clickAt + marker | ✅ true | ✅ 可以 | ❌ 不可以 | 🟢 低 |
| el.click() | ❌ false | ❌ 不可以 | ✅ 可以（但触发风控） | 🔴 极高 |
| dispatchEvent() | ❌ false | ❌ 不可以 | ❌ 不可以 | 🟡 中 |

### 3.3 核心矛盾

**clickAt 能穿透到 iframe 的普通链接元素（如 .name），但无法穿透到 iframe 内的 Vue 组件按钮（如 .btn-greet）。而 el.click() 能触发 Vue 组件但会触发风控。**

这是当前最大的技术瓶颈：安全的点击方式打不到目标，能打到目标的方式不安全。

---

## 四、风控事故详细复盘

### 4.1 事故时间线

```
21:46  开始测试浮窗内打招呼按钮
  ↓    尝试 clickAt + marker → 无法触发浮窗内 Vue 按钮
  ↓    尝试 el.click() → 触发 API 请求成功，打招呼发出
22:xx  账号立即被限制 web 端登录
  ↓    页面跳转至限制提示页
  ↓    「检测到异常操作，web端登录受限24小时，预计2026-05-12 22:01:54解除」
```

### 4.2 触发操作

```javascript
// 在 iframe contentDocument 中执行
var iframe = document.querySelector(".frame-box iframe");
var doc = iframe.contentDocument;
var greetBtn = doc.querySelector(".resumeGreet .btn-greet");
greetBtn.click();  // ← 这行触发了风控
```

### 4.3 风控检测机制推测

| 可能的检测点 | 分析 |
|-------------|------|
| isTrusted 属性 | `el.click()` 产生的事件 `isTrusted=false`，CDP `Input.dispatchMouseEvent` 产生 `isTrusted=true` |
| 操作模式 | 自动化操作节奏过于规律，缺少人类随机性 |
| 事件链 | 真实用户操作有完整事件链（mousedown→mouseup→click），el.click() 只有 click |
| 操作频率 | 连续测试多个方案，短时间内多次触发 |

### 4.4 之前的 clickAt 打招呼为何未触发风控？

早期 session 用 clickAt 成功打招呼 3 人未触发风控，可能原因：
1. clickAt 产生 isTrusted=true 事件，更接近真实用户
2. 当时是卡片级打招呼，不是浮窗内操作
3. 当天操作总量较少

---

## 五、关键技术细节

### 5.1 clickAt + marker 方案

**原理**：在父页面创建一个视口定位的透明 marker div，用 CDP clickAt 点击这个 marker，CDP 在该坐标发起 `Input.dispatchMouseEvent`，产生 isTrusted=true 事件。

```javascript
// Step 1: 在 iframe 内获取目标元素坐标
var nameEl = card.querySelector(".name");
var nameRect = nameEl.getBoundingClientRect();
var iframeRect = iframe.getBoundingClientRect();

// Step 2: 在父页面创建 marker
var marker = document.createElement("div");
marker.id = "__click_marker_0";
marker.style.cssText =
  "position:fixed;" +
  "left:" + (nameRect.x + iframeRect.x) + "px;" +
  "top:" + (nameRect.y + iframeRect.y) + "px;" +
  "width:" + nameRect.width + "px;" +
  "height:" + nameRect.height + "px;" +
  "pointer-events:none;" +   // 关键：让鼠标事件穿透
  "z-index:99999;";
document.body.appendChild(marker);
```

```bash
# Step 3: CDP clickAt
curl -s -X POST "http://localhost:3456/clickAt?target=TARGET_ID" -d '#__click_marker_0'
```

**关键点**：
- `pointer-events:none` 让 marker 不拦截鼠标事件，CDP 事件穿过 marker 到达下面的元素
- 对于 `.name` 这种普通 `<a>` 链接，事件能正确触发页面导航/浮窗打开
- 对于 Vue 组件按钮（`.btn-greet`），事件到达 iframe 边界但不触发 Vue 事件处理器

### 5.2 中文内容提取

eval 返回中文会乱码，必须 base64 编码：

```javascript
var text = element.innerText;
btoa(unescape(encodeURIComponent(text)))
// 解码：decodeURIComponent(escape(atob(encoded)))
```

### 5.3 iframe 访问标准模式

```javascript
// 从父页面访问推荐列表 iframe
var iframe = document.querySelector(".frame-box iframe");
var doc = iframe.contentDocument;
var cards = doc.querySelectorAll(".card-item");
```

### 5.4 打招呼单步完成

推荐页点击「打招呼」按钮 → 自动发送默认问候语 → 按钮变为「继续沟通」→ 候选人会话自动出现在聊天列表。**不存在独立的 Step 2 发送操作**。

---

## 六、安全策略建议

### 6.1 操作安全分级

| 安全等级 | 操作 | 建议 |
|----------|------|------|
| 🟢 安全 | 读取候选人卡片 | 可批量执行 |
| 🟢 安全 | 读取浮窗简历 | 可批量执行 |
| 🟢 安全 | AI 评估打分 | 可批量执行 |
| 🟡 中等 | 收藏候选人 | 少量执行，间隔 ≥5秒 |
| 🔴 高风险 | 打招呼 | 建议 HR 手动 |
| 🔴 禁止 | el.click() 任何操作 | 绝对禁止 |

### 6.2 频率控制

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| 操作间隔 | 8-10 秒 | 任何连续操作之间 |
| 单轮上限 | 3 人 | 每轮最多处理 3 位候选人 |
| 轮次间隔 | ≥30 分钟 | 两轮操作之间 |
| 渐进测试 | 1→3→5 | 先1人验证安全再逐步扩展 |

### 6.3 风控信号检测

出现以下信号时**立即停止所有操作**：
- 验证码弹窗
- 「检测到异常操作」提示
- 页面突然跳转到登录/限制页
- 按钮点击无反应（可能权益耗尽或已被限制）

---

## 七、当前技术瓶颈与未解决问题

### 7.1 核心瓶颈

**浮窗内 Vue 组件按钮的安全点击方案缺失**

- clickAt + marker：事件无法穿透到 iframe 内 Vue 组件
- el.click()：能触发但触发风控
- dispatchEvent()：无法触发 Vue 事件

### 7.2 可能的突破方向

| 方向 | 思路 | 难度 | 风险 |
|------|------|------|------|
| CDP Input.dispatchMouseEvent 直接坐标点击 | 不用 marker，直接在按钮坐标发起鼠标事件序列 | 中 | 中 |
| 完整事件链模拟 | mousedown → mouseup → click 顺序触发 | 中 | 中 |
| iframe 内部 CDP 注入 | 在 iframe document 上下文中执行 CDP 命令 | 高 | 高 |
| 放弃浮窗内操作 | 推荐页卡片级打招呼（已验证可行） | 低 | 低 |
| 全手动模式 | 自动化只读+评估，HR手动操作收藏/打招呼 | 低 | 无 |

### 7.3 推荐策略

**短期（立即可行）**：
- 自动化只做读取 + AI 评估 + 生成报告
- 收藏和打招呼由 HR 手动完成
- 推荐页卡片级打招呼（非浮窗内）可用 clickAt 低频执行

**中期（需进一步验证）**：
- 尝试 CDP 直接坐标点击 + 完整事件链
- 在更安全的测试环境下验证

**长期（架构升级）**：
- 评估是否可以通过 BOSS API（如有）实现打招呼
- 考虑 Android 模拟器方案（App 端风控可能不同）

---

## 八、技能套件优化需求

### 8.1 web-access 技能优化

#### 8.1.1 新增能力需求

1. **iframe 内坐标点击**：支持在 iframe 内元素坐标直接发起 CDP `Input.dispatchMouseEvent`，无需父页面 marker 中转
2. **完整鼠标事件链**：支持 `mousedown → mouseup → click` 三步模拟，而非单次 click
3. **iframe 上下文 eval**：支持在指定 iframe 的 document 上下文中执行 JavaScript（当前 eval 在父页面执行）
4. **风控检测回调**：操作后自动检测页面是否出现风控信号（验证码、限制页、登录跳转）

#### 8.1.2 现有功能改进

1. **zhipin.com 站点模式**：补充浮窗内按钮点击的已知限制说明
2. **clickAt 方案**：增加对 Vue 组件按钮不可用的警告
3. **安全策略内置**：将频率控制、单轮上限内置到 web-access 配置中

### 8.2 kozsail-boss-hiring-assistant 技能优化

#### 8.2.1 references 文件更新

| 文件 | 优化项 |
|------|--------|
| `boss-greet-recipe.md` | 增加「推荐页卡片级打招呼」vs「浮窗内打招呼」两种路径选择指南；增加 clickAt 直接坐标方案（待验证） |
| `boss-recommend-resume-recipe.md` | 补充简历详情 canvas 截图/OCR 方案；补充浮窗关闭后的状态清理步骤 |
| `boss-recommend-read-recipe.md` | 补充虚拟滚动处理策略（更多候选人加载） |
| `boss-screening-service.md` | 将「安全策略」独立为专门章节，与业务逻辑解耦 |
| `boss-workflow.md` | 更新全流程，明确哪些步骤自动化、哪些需 HR 手动 |

#### 8.2.2 新增文件需求

| 新文件 | 用途 |
|--------|------|
| `boss-safety-policy.md` | 风控安全策略独立文档，包含频率控制、风控检测、事故应急预案 |
| `boss-debug-playbook.md` | 常见问题排查手册（权益耗尽 vs 风控限制 vs 技术故障） |
| `boss-iframe-ops-guide.md` | iframe 操作技术指南（三层嵌套下的各类操作方案对比） |

#### 8.2.3 knowledge 文件更新

| 文件 | 优化项 |
|------|--------|
| `BOSS筛选标准.md` | 增加自动化场景下的安全筛选标准 |
| `HR管理.md` | 增加 HR 手动操作清单（收藏+打招呼） |

---

## 九、已验证的完整操作流程

### 9.1 推荐页候选人粗筛流程（当前可用）

```
1. 确保浏览器已登录 BOSS 直聘，且在推荐页 /web/chat/recommend
2. 通过 iframe.contentDocument 读取候选人卡片列表
3. 提取卡片摘要（姓名、职位、薪资、标签）→ base64 编码
4. AI 四维评分（专业对口30% + 经验基础35% + 学历达标20% + 简历完整度15%）
5. 对评分达标的候选人：
   a. 创建 marker + clickAt 点击名字 → 打开简历浮窗
   b. 读取浮窗 .resume-summary 经历概览 → base64 编码
   c. AI 深度评估
6. 生成筛选报告（包含推荐/待定/淘汰分类）
7. ⚠️ 收藏和打招呼建议 HR 手动操作
```

### 9.2 推荐页卡片级打招呼流程（低风险，已验证）

```
1. 在推荐页卡片上找到 .btn-greet 按钮
2. 创建 marker + clickAt 点击
3. 等待按钮变为「继续沟通」
4. 间隔 8-10 秒
5. 下一位候选人
6. 单轮不超过 3 人
```

---

## 十、附录

### A. CDP 代理 API 参考

| 接口 | 用途 | 示例 |
|------|------|------|
| `GET /eval?target=ID` | 执行 JS | `eval?target=TAB_ID` + body: JS code |
| `POST /clickAt?target=ID` | 坐标点击 | `clickAt?target=TAB_ID` + body: CSS selector |
| `GET /navigate?url=URL` | 导航 | `navigate?url=https://...` |
| `GET /screenshot?target=ID` | 截图 | `screenshot?target=TAB_ID` |

### B. 关键 CSS 选择器速查

| 目标 | 选择器 |
|------|--------|
| 推荐列表 iframe | `.frame-box iframe` |
| 候选人卡片 | `.card-item` |
| 候选人姓名 | `.name` |
| 卡片打招呼按钮 | `.btn-greet` |
| 简历浮窗 | `.boss-dialog__wrapper.dialog-lib-resume` |
| 浮窗经历概览 | `.resume-summary` |
| 浮窗打招呼按钮 | `.resumeGreet .btn-greet` |
| 浮窗收藏按钮 | `.like-icon-and-text` |
| 权益耗尽标记 | `.overdue-tip-icon` |

### C. 事故记录

| 日期 | 事故 | 影响 | 恢复 |
|------|------|------|------|
| 2026-05-11 22:xx | iframe 内 el.click() 触发风控 | web 端登录受限 24h | 2026-05-12 22:01:54 |

### D. 相关文件索引

| 文件路径 | 内容 |
|----------|------|
| `~/.workbuddy/skills/kozsail-boss-hiring-assistant/references/boss-recommend-read-recipe.md` | 推荐页读取方案 |
| `~/.workbuddy/skills/kozsail-boss-hiring-assistant/references/boss-recommend-resume-recipe.md` | 浮窗简历读取方案 |
| `~/.workbuddy/skills/kozsail-boss-hiring-assistant/references/boss-greet-recipe.md` | 打招呼方案 |
| `~/.workbuddy/skills/kozsail-boss-hiring-assistant/references/boss-screening-service.md` | 筛选服务方案 |
| `~/.workbuddy/skills/web-access/references/site-patterns/zhipin.com.md` | BOSS 站点模式 |
