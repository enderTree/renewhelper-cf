# AGENT.md - RenewHelper 项目指南

## 项目概述

**RenewHelper (时序·守望)** 是一个基于 Cloudflare Workers 的全栈服务生命周期管理工具，版本 v1.3.2。该项目是一个完全 Serverless 的单文件应用，用于管理周期性订阅、域名续费、服务器到期等场景的提醒和自动化管理。

## 技术栈

| 层面 | 技术 |
|------|------|
| **运行时** | Cloudflare Workers (Edge Runtime) |
| **存储** | Cloudflare KV |
| **前端** | Vue 3 + Element Plus (内嵌单文件) |
| **认证** | JWT (HMAC-SHA256) |
| **定时任务** | Cloudflare Cron Triggers |

## 项目结构

```
renewhelper-cf/
├── _worker.js        # 核心代码文件 (全部逻辑，约2500行)
├── wrangler.toml     # Cloudflare Workers 配置
├── package.json      # 项目元数据
├── README.md         # 中文文档
├── README_EN.md      # 英文文档
├── LICENSE           # MIT 许可证
└── assets/           # 截图和资源文件
```

## 核心架构 (`_worker.js`)

### 代码组织结构

1. **农历核心 (1-210行)**
   - `LUNAR_DATA`: 农历数据表和转换算法 (1900-2100)
   - `calcBiz`: 农历/公历计算业务逻辑
     - `l2s()`: 农历转公历
     - `addPeriod()`: 周期增加计算

2. **基础设施 (210-250行)**
   - `Router`: 简易路由类
   - `response()` / `error()`: 响应工具函数

3. **业务服务 (250-550行)**
   - `Auth`: JWT 认证模块
     - `login()`, `verify()`, `sign()`, `verifyToken()`
     - 使用 Web Crypto API 进行 HMAC-SHA256 签名
     - 恒定时间比较防止时序攻击
   - `DataStore`: KV 数据存储层
     - 键: `SYS_CONFIG`, `DATA_ITEMS`, `LOGS`
     - 乐观锁版本控制防止并发冲突
   - `RateLimiter`: 混合限流
     - 内存层: 1秒/次
     - KV层: 每日100次/IP

4. **通知服务 (550-800行)**
   - 支持渠道: Telegram, Bark, PushPlus, NotifyX, Resend (Email), Webhook, 飞书 (Feishu), 企业微信 (WeCom)
   - `NotifyService.send()`: 统一发送接口

5. **定时任务处理 (800-1100行)**
   - `CronService.run()`: Cron 触发器入口
   - 自动续期、自动禁用、到期提醒

6. **API 路由 (1100-1400行)**
   - `POST /api/login` - 登录认证
   - `GET /api/list` - 获取服务列表
   - `POST /api/save` - 保存数据
   - `GET /api/export` - 导出备份
   - `POST /api/import` - 导入数据
   - `POST /api/test-notify` - 测试通知
   - `GET /api/calendar.ics` - ICS 日历订阅
   - `GET /api/logs` - 获取日志
   - `POST /api/check` - 手动触发检查

7. **前端 HTML (1400-2500行)**
   - Vue 3 + Element Plus 单文件前端
   - 支持深色/浅色主题
   - 中英双语界面
   - 响应式设计

## 数据模型

### 服务项 (Item)
```javascript
{
  id: string,          // UUID
  name: string,        // 服务名称
  tags: string[],      // 标签数组
  mode: 'subscribe' | 'expire',  // 循环订阅/到期重置
  isLunar: boolean,    // 是否使用农历
  cycle: { value: number, unit: 'day'|'month'|'year' },
  startDate: string,   // 起始日期 YYYY-MM-DD
  nextDate: string,    // 下次到期日期
  lunarInfo: object,   // 农历信息 (如果启用)
  autoRenew: boolean,  // 自动续期
  autoDisable: boolean,// 自动禁用
  enabled: boolean,    // 是否启用
  notifyDays: number[] // 提前提醒天数
}
```

### 系统设置 (Settings)
```javascript
{
  enableNotify: boolean,
  autoDisableDays: number,
  language: 'zh' | 'en',
  timezone: string,
  jwtSecret: string,
  calendarToken: string,
  enabledChannels: string[],
  notifyConfig: {
    telegram: { token, chatId },
    bark: { server, key },
    pushplus: { token },
    notifyx: { apiKey },
    resend: { apiKey, from, to },
    webhook: { url }
  }
}
```

## 环境变量

| 变量名 | 必需 | 说明 |
|--------|------|------|
| `AUTH_PASSWORD` | 否 | 登录密码，默认 `admin` |
| `RENEW_KV` | 是 | KV 命名空间绑定 (不可更改名称) |

## 开发指南

### 本地开发
```bash
# 安装 wrangler CLI
npm install -g wrangler

# 登录 Cloudflare
wrangler login

# 本地开发
wrangler dev

# 部署
wrangler deploy
```

### Cron 触发器
必须配置 `0/30 * * * *` (每30分钟) 以启用自动化功能。

## 关键逻辑说明

### 农历计算
- 基于 1900-2100 年农历数据表
- 支持闰月处理
- 缓存优化避免重复计算

### 周期计算
- 公历: 按天/月/年直接加减
- 农历: 先转公历计算，再转回农历

### 并发安全
- 使用乐观锁 (version 字段) 防止数据覆盖
- 版本冲突时返回 `VERSION_CONFLICT` 错误

### 安全机制
- JWT 7天过期
- 恒定时间比较防时序攻击
- 双层限流 (内存 + KV)
- 敏感操作需二次确认

## 常见修改场景

1. **添加新通知渠道**: 在 `NotifyService` 中添加新的 `send*` 方法
2. **修改前端样式**: 搜索 `<style>` 标签修改 CSS
3. **添加新 API**: 使用 `app.get()` 或 `app.post()` 注册路由
4. **修改默认配置**: 修改 `DataStore.getSettings()` 中的 `defaults` 对象

## 注意事项

- 这是单文件应用，所有代码在 `_worker.js` 中
- KV 绑定名称必须为 `RENEW_KV`
- Cron 表达式必须为 `0/30 * * * *`
- 前端使用 CDN 加载 Vue 和 Element Plus
- 支持数据导入/导出用于备份和迁移

