---
name: xhs-to-lark
version: 1.0.0
description: "小红书收藏夹抓取并导入飞书多维表格。使用 Playwright 浏览器自动化抓取小红书用户收藏的笔记，然后通过 lark-cli 创建飞书多维表格并批量导入数据。"
metadata:
  requires:
    bins: ["lark-cli", "python3"]
    pythonPackages: ["playwright"]
---

# xhs-to-lark

> **前置条件：** 需要安装 Playwright（`pip install playwright`）和 lark-cli。用户需在小红书网页版登录态下操作。

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

#### 抓取脚本模板

```python
import asyncio
import json
import re
from playwright.async_api import async_playwright

USER_DATA_DIR = 'C:/Users/{username}/browser_data'  # 持久化目录

async def scrape_favorites(profile_url, target_count=100):
    async with async_playwright() as p:
        context = await p.chromium.launch_persistent_context(
            USER_DATA_DIR,
            headless=False,
            user_agent='Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/130.0.0.0 Safari/537.36',
            viewport={'width': 1280, 'height': 900}
        )
        page = context.pages[0] if context.pages else await context.new_page()

        await page.goto(profile_url, wait_until='domcontentloaded', timeout=60000)
        await asyncio.sleep(8)

        # 登录检测
        if 'login' in page.url or 'passport' in page.url:
            print("请在浏览器中手动登录...")
            for _ in range(180):
                await asyncio.sleep(2)
                if 'user/profile' in page.url:
                    break
            else:
                await context.close()
                return []

        # 注入 MutationObserver 累积收集
        await page.evaluate('''() => {
            window.__collectedItems = {};
            window.__lastAddTime = Date.now();
            const observer = new MutationObserver(() => {
                document.querySelectorAll('section.note-item').forEach(section => {
                    const coverLink = section.querySelector('a.cover');
                    if (!coverLink) return;
                    const href = coverLink.getAttribute('href') || '';
                    const match = href.match(/\\/([a-f0-9]{24})(?:\\?|$)/);
                    if (!match) return;
                    const noteId = match[1];
                    if (window.__collectedItems[noteId]) return;
                    const titleEl = section.querySelector('.title span, .title');
                    const title = titleEl ? titleEl.textContent.trim() : '';
                    const authorEl = section.querySelector('.author .name, .name');
                    const author = authorEl ? authorEl.textContent.trim() : '';
                    const fullUrl = coverLink.href || '';
                    if (title) {
                        window.__collectedItems[noteId] = { noteId, title, author, link: fullUrl };
                        window.__lastAddTime = Date.now();
                    }
                });
            });
            observer.observe(document.body, { childList: true, subtree: true });
            // 初始扫描
            document.querySelectorAll('section.note-item').forEach(section => {
                const coverLink = section.querySelector('a.cover');
                if (!coverLink) return;
                const href = coverLink.getAttribute('href') || '';
                const match = href.match(/\\/([a-f0-9]{24})(?:\\?|$)/);
                if (!match) return;
                const noteId = match[1];
                const titleEl = section.querySelector('.title span, .title');
                const title = titleEl ? titleEl.textContent.trim() : '';
                const authorEl = section.querySelector('.author .name, .name');
                const author = authorEl ? authorEl.textContent.trim() : '';
                const fullUrl = coverLink.href || '';
                if (title) window.__collectedItems[noteId] = { noteId, title, author, link: fullUrl };
            });
        }''')

        # 滚动加载
        stale_rounds = 0
        for i in range(2000):
            await page.evaluate('window.scrollBy(0, 300)')
            await asyncio.sleep(0.35)
            count = await page.evaluate('Object.keys(window.__collectedItems).length')
            last_add = await page.evaluate('Date.now() - window.__lastAddTime')
            if count >= target_count:
                break
            if last_add > 10000:
                stale_rounds += 1
                if stale_rounds > 3:
                    break
            else:
                stale_rounds = 0

        items = await page.evaluate('Object.values(window.__collectedItems)')

        # 清理作者名（去掉末尾数字粉丝数）
        for item in items:
            item['author'] = re.sub(r'\d+$', '', item.get('author', '')).strip()

        await context.close()
        return items
```

### 2.2 创建飞书多维表格

使用 `lark-cli` 在飞书知识库中创建 bitable 并导入数据。

#### 命令序列

```bash
# 1. 在知识库中创建 bitable 节点
lark-cli wiki +node-create --space-id <SPACE_ID> --obj-type bitable --title "小红书收藏整理" --as user

# 2. 获取默认表 ID
lark-cli base +table-list --base-token <BASE_TOKEN> --as user

# 3. 重命名默认字段（第一个字段改为"标题"）
lark-cli base +field-update --base-token <BASE_TOKEN> --table-id <TABLE_ID> --field-id <FIELD_ID> --json '{"type":"text","name":"标题"}' --as user

# 4. 创建其他字段
lark-cli base +field-create --base-token <BASE_TOKEN> --table-id <TABLE_ID> --json '{"type":"text","name":"作者"}' --as user
lark-cli base +field-create --base-token <BASE_TOKEN> --table-id <TABLE_ID> --json '{"type":"text","name":"链接","style":{"type":"url"}}' --as user
lark-cli base +field-create --base-token <BASE_TOKEN> --table-id <TABLE_ID> --json '{"type":"checkbox","name":"是否学习"}' --as user

# 5. 删除不需要的默认字段
lark-cli base +field-delete --base-token <BASE_TOKEN> --table-id <TABLE_ID> --field-id <FIELD_ID> --yes --as user

# 6. 重命名表
lark-cli base +table-update --base-token <BASE_TOKEN> --table-id <TABLE_ID> --name "收藏列表" --as user

# 7. 批量导入数据（单次最多 200 条）
lark-cli base +record-batch-create --base-token <BASE_TOKEN> --table-id <TABLE_ID> --as user --json @batch_data.json
```

#### batch_data.json 格式

```json
{
  "fields": ["标题", "作者", "链接", "是否学习"],
  "rows": [
    ["笔记标题", "作者名", "https://www.xiaohongshu.com/user/profile/...?xsec_token=...", false]
  ]
}
```

> **注意：** `--json @file` 要求文件路径是相对路径，需先 `cd` 到文件所在目录。

### 2.3 认证流程

首次使用 `lark-cli` 的 base 功能需要授权：

```bash
lark-cli auth login --domain base
```

会弹出飞书授权页面，用户扫码登录后自动获得 base 相关权限。

## 3. 已知限制

### 3.1 小红书链接格式

- 收藏页抓取到的链接格式为 `https://www.xiaohongshu.com/user/profile/{userId}/{noteId}?xsec_token=...&xsec_source=pc_collect`
- `xsec_token` 是会话绑定的访问令牌，**会过期**
- `/explore/{noteId}` 格式在无 token 时会重定向到首页或返回 404（error_code=300031）
- 目前没有找到稳定的公开链接格式，链接需要在登录态下才能访问

### 3.2 虚拟滚动

- 小红书收藏夹使用虚拟滚动，DOM 中只保留约 15-20 个可见元素
- 不能用 `document.querySelectorAll('section.note-item').length` 来判断总数
- 必须用 MutationObserver 累积收集所有出现过的元素

### 3.3 批量导入限制

- 飞书多维表格单次批量创建最多 200 条记录
- 超过 200 条需分批写入，批次间延迟 0.5-1 秒

## 4. 完整工作流

```
用户给出小红书收藏夹 URL
    ↓
Playwright 打开浏览器，用户登录
    ↓
MutationObserver + 滚动加载，累积收集笔记
    ↓
保存为 JSON 文件
    ↓
lark-cli 创建 bitable（知识库节点）
    ↓
配置字段：标题、作者、链接、是否学习
    ↓
批量导入记录
    ↓
返回飞书知识库链接
```
