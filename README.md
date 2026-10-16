# GXNU 课表采集器

> ⚠️ **2026-10-16 起，本项目已整体迁移**至
> [yuyangu/userscripts](https://github.com/yuyangu/userscripts)（课表采集器脚本 + 同步脚本统一维护）。
> 安装 / 更新请访问新仓库；本仓库仅作历史归档，不再更新。

把**广西师范大学**教务系统的课表，同步成 Obsidian 可读的 Markdown 笔记。

包含两部分：

| 组件 | 形态 | 职责 |
|---|---|---|
| **课表采集器** | 油猴脚本（Tampermonkey） | 在已登录的课表页一键导出 JSON 快照 |
| **同步脚本** | Python | 读取快照 → 解析 → 变更比对 → 写 Obsidian |

> 设计前提：教务系统 `v20` Cookie 受 Chromium App-Bound Encryption 保护，
> 脚本**无法**离线解密提取。因此改由**浏览器自身**（油猴）读取已登录页面，
> 全程不碰加密 Cookie、不注入进程、不提权。

---

## 支持的数据源

三个来源结构完全不同，脚本自动识别，无需手动指定：

| | 教务「我的课表」 | 教务「课表查询」 | 易班「易知独秀」 |
|---|---|---|---|
| 入口 | `/jw/admin?sfjrxk=1` | `/jw/admin/api/getKbxx` | `/yzdx/dean/course` |
| 页面形态 | 主页面 + **iframe** | 顶层单页 `<table>` | Vue3 SPA，**CSS Grid 无 table** |
| 周次 | 一次给全学期 | 一次只渲染一周 | 一次给全学期 |
| 鉴权 | Cookie | Cookie | **Bearer token** |

易班接口只认 `Authorization: Bearer`。脚本**不读** localStorage 里的 token，
而是在 `document-start` 注入钩子，**捕获页面自身发出的那次请求响应**。

---

## 快速开始

### 一、装采集器（油猴脚本）

1. 给浏览器装 [Tampermonkey](https://www.tampermonkey.net/) 扩展
2. 打开下面的链接，Tampermonkey 会自动弹出安装页 → 点「安装」

   **[⬇ 点此安装 GXNU 课表采集器](https://raw.githubusercontent.com/yuyangu/userscripts/main/gxnu-course-collector.user.js)**

3. 打开课表页，右下角出现「📚 课表采集器」面板 → 点「采集并导出课表」
4. 浏览器下载 `course-schedule-*.json` 到默认下载目录

> 易班页需**进页后刷新一次**：从易班首页点进来属于 SPA 内部跳转，
> 不会重新加载页面，钩子赶不上页面已发出的那次请求。

### 二、装同步脚本

```bash
# 1. 取回仓库
git clone https://github.com/yuyangu/gxnu-course-collector.git
cd gxnu-course-collector/sync

# 2. 准备配置（学号等页面 DOM 中不出现的信息）
cp config.example.json config.json
#    编辑 config.json，填入你自己的姓名/学号/校区

# 3. 运行
python sync_course_schedule.py
```

首次运行会**建立基线**（不算变更）。之后只在课表**真的改了**时才提醒。

可用参数：

```bash
python sync_course_schedule.py --dry-run        # 只比对，不动文件
python sync_course_schedule.py --list-exports   # 查看导出文件与消费状态
python sync_course_schedule.py --show-current   # 打印当前周课表
```

Obsidian Vault 路径默认取 `~/Documents/Obsidian Vault`，
可用环境变量覆盖：`KB_VAULT="D:/MyVault"`。

### 三、输出

每个学期生成三个视图：

| 文件 | 形态 |
|---|---|
| `课表_<学期>_周视图.md` | 7 列 × 12 节 HTML 网格，连堂课纵向合并 |
| `课表_<学期>_月视图.md` | 按日期逐日排列，含周末 |
| `课表_<学期>.md` | 课程汇总 + 实践环节 + 节次时间表 |

---

## 自动更新

脚本头部已内置更新地址，Tampermonkey 会**每天自动检查一次**更新：

```
// @updateURL    https://raw.githubusercontent.com/yuyangu/userscripts/main/gxnu-course-collector.user.js
// @downloadURL  https://raw.githubusercontent.com/yuyangu/userscripts/main/gxnu-course-collector.user.js
```

手动检查：Tampermonkey 面板 → 已安装脚本 → 点「检查更新」。

### ⚠ 关于更新地址（2026-10-16 迁移后）

脚本已迁移至 [yuyangu/userscripts](https://github.com/yuyangu/userscripts)，
更新地址改为 **raw.githubusercontent.com 直连**（实时生效，无 CDN 缓存延迟）。
Tampermonkey 会自动检查更新；也可在面板手动点「检查更新」。

### 发布新版本（维护者）

1. 改脚本 → **递增 `@version`**（如 `1.7.1` → `1.7.2`）
2. 上传到 userscripts 仓库
3. 用户端 Tampermonkey「检查更新」即可收到

> ⚠ 不改 `@version`，Tampermonkey 不会认为有新版本。

---

## 目录结构

```
.
├── 安装说明.md                      # 详细安装与排错（采集器已迁往 userscripts）
├── README.md
├── LICENSE
└── sync/
    ├── sync_course_schedule.py     # 同步主程序
    ├── config.example.json         # 配置模板
    └── README.md                   # 同步端设计与踩坑记录
```

---

## 安全边界

**油猴脚本**

| 项目 | 说明 |
|---|---|
| `@match` | 仅 `jwjx.gxnu.edu.cn/jw/admin*` 与 `yiban.gxnu.edu.cn/yzdx*` |
| 网络请求 | **零** —— 不使用 `fetch`/`XHR`/`WebSocket` 外发数据 |
| 数据流向 | 仅通过浏览器下载写入本机默认下载目录 |
| 读取范围 | 仅课表页 DOM（含**同源** iframe）；不读 Cookie、不读 token、跨源帧自动跳过 |
| 可审阅性 | 单文件、无压缩、无混淆 |

**同步脚本**：只读导出文件，只写 Obsidian 的 `课表/` 目录，**无任何网络访问**。

> ⚠ 采集导出的 JSON **含你的姓名**。学号不在页面 DOM 中，由本地 `config.json` 提供。
> 请注意导出文件与 `config.json` 的存放位置，勿提交到公开仓库。

---

## 已知局限

1. 解析依赖课表页的表格结构，教务改版可能导致解析失败（会明确报错，不会静默产出 0 条）
2. 校历第 1 周起始日与当前学期为常量，换学期需更新
3. 周视图用 HTML `<table>`（Markdown 表格表达不了 `rowspan`），
   Obsidian **阅读模式**下渲染为网格，源码模式下看到的是标签

更多细节见 [`sync/README.md`](sync/README.md) 与 [`安装说明.md`](安装说明.md)。

---

## License

[MIT](LICENSE)
