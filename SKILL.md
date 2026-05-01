---
name: xhs-to-lark
version: 1.1.0
description: "小红书收藏夹抓取并导入飞书多维表格。使用 Playwright 浏览器自动化抓取小红书用户收藏的笔记，然后通过 lark-cli 创建飞书多维表格并批量导入数据。"
metadata:
  requires:
    bins: ["lark-cli", "python3"]
    pythonPackages: ["playwright"]
---

# xhs-to-lark

> **前置条件：** 需要安装 Playwright（`pip install playwright` 和 `playwright install chromium`）和 lark-cli。用户需在小红书网页版登录态下操作。
> **执行前必做：** 先阅读 [`references/scrape-script.md`](references/scrape-script.md) 了解完整抓取脚本，再阅读 [`references/lark-import.md`](references/lark-import.md) 了解飞书导入流程。

## 1. 何时使用本 Skill

- 用户要求整理小红书收藏夹
- 用户要求将小红书笔记导入飞书多维表格
- 用户给出小红书个人主页链接（含 `tab=fav`）

## 2. 核心流程

### 2.1 抓取小红书收藏夹

小红书收藏夹使用 **虚拟滚动（virtual scroll）**，DOM 中只保留可视区域附近的元素。必须使用 **MutationObserver + 累积收集** 策略，而非简单计数 DOM 元素。

#### 关键技术点

1. **持久化浏览器上下文**：使用 `launch_persistent_context` 保存登录态 cookies，避免每次重新登录
2. **JS 侧累积收集**：在页面注入 MutationObserver，将出现过的笔记写入 `window.__collectedItems` 对象（以 noteId 为 key 去重），避免虚拟滚动回收已加载的 DOM 节点导致丢失
3. **滚动策略**：每次滚动 300px，间隔 350ms，持续直到达到目标数量或超时
4. **链接格式**：收藏页的可点击链接格式为 `/user/profile/{userId}/{noteId}?xsec_token=...&xsec_source=pc_collect`，其中 `xsec_token` 是会话绑定的访问令牌
5. **User-Agent**：使用完整桌面 Chrome UA，避免被反爬检测。完整脚本见 [`references/scrape-script.md`](references/scrape-script.md)

#### 页面结构

```
section.note-item
  ├── a.cover[href="/user/profile/{userId}/{noteId}?xsec_token=..."]  ← 封面链接（可点击）
  ├── a.title[href="同上"]                                            ← 标题链接
  │   └── .title span                                                 ← 标题文本
  └── a.author[href="/user/profile/{authorId}?..."]                   ← 作者链接
      └── .name                                                       ← 作者名（可能带末尾数字=粉丝数）
```

- `a.cover` 的 `href` 是带 `xsec_token` 的完整链接，是唯一可靠的可点击入口
- 页面还有隐藏的 `a[href="/explore/{noteId}"]`（`display: none`），**不要用这个**，它不带 token 无法访问

#### 抓取脚本

完整脚本见 [`references/scrape-script.md`](references/scrape-script.md)。

### 2.2 创建飞书多维表格并导入

使用 `lark-cli` 在飞书知识库中创建 bitable 并导入数据。完整流程见 [`references/lark-import.md`](references/lark-import.md)。

#### 命令序列

```bash
# 1. 认证（首次）
lark-cli auth login --domain base

# 2. 在知识库中创建 bitable 节点
lark-cli wiki +node-create --space-id <SPACE_ID> --obj-type bitable --title "小红书收藏整理" --as user

# 3. 获取默认表 ID
lark-cli base +table-list --base-token <BASE_TOKEN> --as user

# 4. 获取默认字段 ID
lark-cli base +field-list --base-token <BASE_TOKEN> --table-id <TABLE_ID> --as user

# 5. 配置字段（改、增、删）
lark-cli base +field-update --base-token <BASE_TOKEN> --table-id <TABLE_ID> --field-id <FIELD_ID> --json '{"type":"text","name":"标题"}' --as user
lark-cli base +field-create --base-token <BASE_TOKEN> --table-id <TABLE_ID> --json '{"type":"text","name":"作者"}' --as user
lark-cli base +field-create --base-token <BASE_TOKEN> --table-id <TABLE_ID> --json '{"type":"text","name":"链接","style":{"type":"url"}}' --as user
lark-cli base +field-create --base-token <BASE_TOKEN> --table-id <TABLE_ID> --json '{"type":"checkbox","name":"是否学习"}' --as user
lark-cli base +field-delete --base-token <BASE_TOKEN> --table-id <TABLE_ID> --field-id <UNUSED_FIELD_ID> --yes --as user

# 6. 重命名表
lark-cli base +table-update --base-token <BASE_TOKEN> --table-id <TABLE_ID> --name "收藏列表" --as user

# 7. 准备数据文件 batch_data.json（相对路径）
# 8. 批量导入
cd <数据文件目录>
lark-cli base +record-batch-create --base-token <BASE_TOKEN> --table-id <TABLE_ID> --as user --json @batch_data.json
```

#### batch_data.json 格式

```json
{
  "fields": ["标题", "作者", "链接", "是否学习"],
  "rows": [
    ["笔记标题", "作者名", "https://www.xiaohongshu.com/user/profile/...?xsec_token=...", false],
    ["第二条笔记", "另一个作者", "https://...", false]
  ]
}
```

> **注意：** `--json @file` 要求文件路径是**相对路径**，需先 `cd` 到文件所在目录再执行。

### 2.3 批量数据清理

抓取到的原始数据需要清理：

| 字段 | 清理规则 |
|------|---------|
| 作者 | 去掉末尾数字（粉丝数），如 `小e同学2546` → `小e同学` |
| 链接 | 保留完整 URL（含 `xsec_token`），不要截断 |
| 标题 | 保留原始文本，含 emoji |

清理脚本：

```python
import re
for item in items:
    item['author'] = re.sub(r'\d+$', '', item.get('author', '')).strip()
```

## 3. 已知限制与注意事项

### 3.1 小红书链接格式（重要）

- 收藏页抓取到的链接格式为 `https://www.xiaohongshu.com/user/profile/{userId}/{noteId}?xsec_token=...&xsec_source=pc_collect`
- `xsec_token` 是会话绑定的访问令牌，**会过期**
- `/explore/{noteId}` 格式在无 token 时会重定向到首页或返回 404（error_code=300031 "当前笔记暂时无法浏览"）
- 目前没有找到稳定的公开链接格式，链接需要在登录态下才能访问
- **不要** 使用 `/explore/{noteId}` 格式，**必须** 保留完整的带 `xsec_token` 的 URL

### 3.2 虚拟滚动

- 小红书收藏夹使用虚拟滚动，DOM 中只保留约 15-20 个可见元素
- 不能用 `document.querySelectorAll('section.note-item').length` 来判断已加载总数
- 必须用 MutationObserver 累积收集所有出现过的元素
- 滚动过快会导致加载不完全，过慢会浪费时间；300px/350ms 是经验值

### 3.3 登录态管理

- 使用 `launch_persistent_context` 持久化浏览器数据（cookies、localStorage）
- 持久化目录需固定，不要每次新建
- 首次使用需用户手动登录（扫码或手机号）
- 登录检测：URL 包含 `login` 或 `passport`
- 等待登录超时：建议 3 分钟（180 秒）

### 3.4 飞书批量导入限制

- 单次批量创建最多 200 条记录
- 超过 200 条需分批写入，批次间延迟 0.5-1 秒
- `--json @file` 必须使用相对路径
- 首次使用 base 功能需执行 `lark-cli auth login --domain base` 授权

### 3.5 反爬与限流

- 小红书有反爬检测，滚动过快可能触发验证
- 建议使用真实 User-Agent（桌面 Chrome）
- 不要在短时间内反复抓取同一页面
- 如果遇到验证码，需用户手动处理

## 4. 完整工作流

```
用户给出小红书收藏夹 URL（含 tab=fav）
    ↓
Playwright 打开持久化浏览器上下文
    ↓
检测登录状态 → 未登录则等待用户手动登录
    ↓
注入 MutationObserver，初始化累积收集器
    ↓
滚动页面（300px/350ms），持续收集笔记
    ↓
达到目标数量或超时 → 停止滚动
    ↓
导出收集到的笔记数据
    ↓
清理数据（作者名去尾数、去重）
    ↓
保存为 JSON 文件
    ↓
lark-cli 认证（首次需要 auth login --domain base）
    ↓
创建 bitable 节点（wiki +node-create --obj-type bitable）
    ↓
配置字段：标题、作者、链接（url样式）、是否学习（checkbox）
    ↓
准备 batch_data.json（相对路径）
    ↓
批量导入记录（+record-batch-create --json @batch_data.json）
    ↓
返回飞书知识库链接给用户
```

## 5. 参考文档

| 文档 | 说明 |
|------|------|
| [`references/scrape-script.md`](references/scrape-script.md) | 完整 Python 抓取脚本，含登录检测、MutationObserver、滚动策略、数据清理 |
| [`references/lark-import.md`](references/lark-import.md) | 完整飞书导入流程，含认证、建表、字段配置、批量写入、错误处理 |
