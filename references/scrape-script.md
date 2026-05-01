---
name: scrape-script
description: "完整的小红书收藏夹抓取 Python 脚本，含登录检测、MutationObserver、虚拟滚动处理、数据清理"
---

# 小红书收藏夹抓取脚本

## 完整脚本

```python
import asyncio
import json
import re
from playwright.async_api import async_playwright

# 持久化浏览器数据目录（固定路径，保存登录态 cookies）
USER_DATA_DIR = 'C:/Users/36193/Desktop/huludao3d/browser_data'

# 目标收藏夹 URL（替换为实际用户 ID）
TARGET_URL = 'https://www.xiaohongshu.com/user/profile/5b51aa2ae8ac2b3b9c4a34db?tab=fav&subTab=note'

async def main():
    async with async_playwright() as p:
        # 使用持久化上下文保存登录态
        context = await p.chromium.launch_persistent_context(
            USER_DATA_DIR,
            headless=False,
            user_agent='Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/130.0.0.0 Safari/537.36',
            viewport={'width': 1280, 'height': 900}
        )
        page = context.pages[0] if context.pages else await context.new_page()

        print("Navigating...")
        await page.goto(TARGET_URL, wait_until='domcontentloaded', timeout=60000)
        await asyncio.sleep(8)

        # 登录检测
        if 'login' in page.url or 'passport' in page.url:
            print("Please log in manually...")
            for _ in range(180):
                await asyncio.sleep(2)
                if 'user/profile' in page.url:
                    print("Login OK!")
                    await asyncio.sleep(5)
                    break
            else:
                print("Timeout")
                await context.close()
                return

        # 注入 MutationObserver 累积收集器（核心：解决虚拟滚动问题）
        print("Scrolling and collecting items...")
        await page.evaluate('''() => {
            window.__collectedItems = {};
            window.__lastAddTime = Date.now();

            // 监听 DOM 变化，自动收集新出现的笔记
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

            // 初始扫描已存在的元素
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
                if (title) {
                    window.__collectedItems[noteId] = { noteId, title, author, link: fullUrl };
                }
            });
        }''')

        # 滚动收集策略：300px/350ms，超时 10s 无新数据则停止
        target = 100
        stale_rounds = 0
        for i in range(2000):
            await page.evaluate('window.scrollBy(0, 300)')
            await asyncio.sleep(0.35)
            count = await page.evaluate('Object.keys(window.__collectedItems).length')
            last_add = await page.evaluate('Date.now() - window.__lastAddTime')

            if count >= target:
                print(f"Reached target: {count} items")
                break

            if last_add > 10000:
                stale_rounds += 1
                if stale_rounds > 3:
                    print(f"No new items for {stale_rounds} rounds, stopping at {count}")
                    break
            else:
                stale_rounds = 0

            if count % 20 == 0 and count > 0:
                print(f"  Collected {count} items...")

        # 导出收集结果
        items = await page.evaluate('Object.values(window.__collectedItems)')
        print(f"Total collected: {len(items)}")

        # 数据清理：作者名去掉末尾粉丝数
        for item in items:
            author = item.get('author', '')
            cleaned = re.sub(r'\d+$', '', author).strip()
            if cleaned:
                item['author'] = cleaned

        # 保存 JSON
        output_path = 'C:/Users/36193/Desktop/huludao3d/小红书收藏数据.json'
        with open(output_path, 'w', encoding='utf-8') as f:
            json.dump(items, f, ensure_ascii=False, indent=2)

        print(f"Saved {len(items)} items")
        await context.close()

asyncio.run(main())
```

## 关键技术点说明

### 1. 虚拟滚动处理

小红书收藏夹使用虚拟滚动（virtual scroll），DOM 中只保留约 15-20 个可见元素。不能用 `querySelectorAll().length` 计数，必须用 MutationObserver 累积收集所有出现过的元素。

### 2. 持久化浏览器上下文

`launch_persistent_context` 会保存 cookies、localStorage 到指定目录，避免每次重新登录。首次使用需用户手动登录（扫码或手机号）。

### 3. 链接格式

收藏页的可点击链接格式为：
```
/user/profile/{userId}/{noteId}?xsec_token=...&xsec_source=pc_collect
```

- `xsec_token` 是会话绑定的访问令牌，**会过期**
- `/explore/{noteId}` 格式在无 token 时会重定向到首页或返回 404
- **必须** 保留完整的带 `xsec_token` 的 URL

### 4. 页面结构

```html
section.note-item
  ├── a.cover[href="/user/profile/{userId}/{noteId}?xsec_token=..."]  ← 封面链接
  ├── a.title[href="同上"]                                            ← 标题链接
  │   └── .title span                                                 ← 标题文本
  └── a.author[href="/user/profile/{authorId}?..."]                   ← 作者链接
      └── .name                                                       ← 作者名（可能带末尾数字=粉丝数）
```

### 5. 滚动参数

- 每次滚动 300px
- 间隔 350ms
- 超过 10 秒无新数据则停止
- 可调整 `target` 变量控制目标数量
