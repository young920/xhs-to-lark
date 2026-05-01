# xhs-to-lark

小红书收藏夹抓取并导入飞书多维表格的 Claude Code Skill。

## 功能

- 使用 Playwright 浏览器自动化抓取小红书用户收藏的笔记
- 通过 MutationObserver + 累积收集策略处理虚拟滚动
- 使用 lark-cli 创建飞书多维表格并批量导入数据
- 自动清理数据（作者名去尾数、去重）

## 前置条件

- [Playwright](https://playwright.dev/python/)（`pip install playwright` 和 `playwright install chromium`）
- [lark-cli](https://github.com/anthropics/claude-code)
- 小红书网页版登录态

## 使用方式

在 Claude Code 中输入：

```
/xhs-to-lark
```

或直接描述需求：

> 把我的小红书收藏夹整理到飞书多维表格

## 文件结构

```
xhs-to-lark/
├── SKILL.md                          # Skill 主文件
├── README.md                         # 本文件
└── references/
    ├── scrape-script.md              # 完整 Python 抓取脚本
    └── lark-import.md                # 飞书导入流程
```

## 核心流程

1. Playwright 打开持久化浏览器上下文（保存登录态）
2. 检测登录状态，未登录则等待用户手动登录
3. 注入 MutationObserver，初始化累积收集器
4. 滚动页面（300px/350ms），持续收集笔记
5. 达到目标数量或超时后停止
6. 清理数据（作者名去尾数、去重）
7. lark-cli 创建 bitable 并配置字段
8. 批量导入记录到飞书多维表格
9. 返回飞书知识库链接

## 已知限制

- 小红书链接含 `xsec_token`，会话绑定且会过期
- 收藏夹使用虚拟滚动，必须用 MutationObserver 累积收集
- 单次批量导入最多 200 条记录
- 遇到验证码需用户手动处理

## License

MIT
