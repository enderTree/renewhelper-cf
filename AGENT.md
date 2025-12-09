# AGENT.md - LifeReminder 项目指南

> 本文档供 AI Agent 或开发者快速了解项目结构和核心逻辑。

## 📋 项目概述

**LifeReminder (时光提醒)** 是一个基于 Cloudflare Workers 的全栈生活事件提醒应用，核心场景：

- 🎂 **生日提醒**：家人、朋友生日，支持农历生日
- 💍 **纪念日提醒**：结婚纪念日、恋爱纪念日等，自动显示第 X 周年
- 🎊 **节日提醒**：春节、中秋、圣诞等中西方节日
- 📅 **周期性日程**：任何需要定期提醒的事项

## 🏗️ 技术栈

| 组件 | 技术 |
|------|------|
| 运行时 | Cloudflare Workers (Serverless) |
| 数据存储 | Cloudflare KV |
| 前端 | Vue 3 + Element Plus (单文件内嵌) |
| 认证 | JWT + 混合限流 (内存+KV) |
| 日历同步 | ICS (iCalendar) 标准 |
| 农历计算 | 内置算法 (1900-2100) |

## 📁 文件结构

```
renewhelper-cf/
├── _worker.js      # 单文件全栈应用 (后端API + 前端Vue)
├── package.json    # 项目元数据
├── README.md       # 中文文档
├── README_EN.md    # 英文文档
├── AGENT.md        # 本文件
└── assets/         # 截图等资源
```

## 🔑 核心架构

### 单文件设计

整个应用在 `_worker.js` 中实现，主要模块：

1. **DataStore** (数据存储层)
   - 基于 KV 的 CRUD 操作
   - 乐观锁机制防止并发冲突
   - 数据模型：`items` (事件列表), `settings` (配置), `logs` (日志)

2. **Auth** (认证模块)
   - JWT 令牌生成与验证
   - 混合限流策略 (内存 + KV)

3. **Notifier** (通知模块)
   - 适配器模式，支持多渠道
   - 当前支持：Telegram, Bark, PushPlus, NotifyX, Resend, Webhook, 飞书, 企业微信

4. **LUNAR** (农历计算)
   - 公历转农历 (`solar2lunar`)
   - 农历转公历 (`lunar2solar`)
   - 支持范围：1900-2100

5. **ICS Generator** (日历生成)
   - 标准 iCalendar 格式
   - 支持精确时区和提醒时间

### API 路由

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | /api/login | 登录认证 |
| GET | /api/items | 获取事件列表 |
| POST | /api/items | 添加/更新事件 |
| DELETE | /api/items/:id | 删除事件 |
| GET | /api/settings | 获取设置 |
| POST | /api/settings | 保存设置 |
| POST | /api/check | 手动触发检查 |
| GET | /api/ics/:token | ICS 日历订阅 |
| POST | /api/notify/test | 测试通知渠道 |

### 数据模型

**事件 (Item)**
```javascript
{
  id: "timestamp_string",
  name: "爸爸生日",
  createDate: "2000-01-15",      // 起始日期
  lastRenewDate: "2024-01-15",   // 上次周期开始
  intervalDays: 1,               // 周期数量
  cycleUnit: "year",             // 周期单位: day/month/year
  type: "cycle",                 // 模式: cycle(周期)/reset(单次)
  enabled: true,
  tags: ["生日"],
  useLunar: true,                // 是否农历
  notifyDays: 3,                 // 提前提醒天数
  notifyTime: "08:00",           // 提醒时间
  autoRenew: true,               // 自动进入下周期
  autoRenewDays: 3               // 过期X天后自动处理
}
```

**设置 (Settings)**
```javascript
{
  timezone: "Asia/Shanghai",
  pushEnabled: true,
  notifyThreshold: 7,
  enabledChannels: ["telegram", "bark"],
  notifyConfig: {
    telegram: { token: "", chatId: "" },
    bark: { server: "", key: "" },
    // ... 其他渠道
  }
}
```

## 🔧 开发指南

### 添加新通知渠道

1. 在 `Notifier.adapters` 中添加适配器函数
2. 在 `DataStore.getSettings` 中添加默认配置
3. 在前端 `messages` 中添加 I18N 标签
4. 在前端模板中添加配置 UI (`el-tab-pane`)
5. 在 `channelMap` 和 `testing` 对象中注册

### I18N 翻译

- 后端翻译：`I18N` 对象 (用于通知消息)
- 前端翻译：Vue 组件内 `messages` 对象

### 周年数显示

对于 `cycleUnit === 'year'` 的事件，通知消息会自动计算并显示周年数：
- 计算方式：`nextDueDate.year - createDate.year`
- 中文格式：`（X 年）`
- 英文格式：`(X yr)`

### 内置节日库

位于 Vue 组件的 `presetFestivals` 数组，包含：
- 中国传统节日 (农历)
- 西方节日 (公历)
- 自动设置 `useLunar` 标志

## 🚨 注意事项

1. **KV 变量名**：必须是 `RENEW_KV`，不可更改
2. **Cron 格式**：`0/30 * * * *` (每半小时检查)
3. **农历范围**：仅支持 1900-2100 年
4. **乐观锁**：保存时检查 `_version` 防止冲突

## 📝 常见修改场景

### 修改默认提醒天数
搜索 `notifyDays: 3` 并修改数值

### 修改默认周期单位
搜索 `cycleUnit: 'year'` (默认改为年周期)

### 添加新标签建议
修改 `tagPlaceholder` 翻译文本

### 调整通知消息格式
修改 `I18N` 对象中的模板字符串

---

*最后更新: 2025-12 | 版本: v2.0.0*
