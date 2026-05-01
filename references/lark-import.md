---
name: lark-import
description: "飞书多维表格创建与批量导入完整流程，含认证、建表、字段配置、批量写入、错误处理"
---

# 飞书多维表格导入流程

## 前置条件

- 已安装 `lark-cli`
- 已有飞书知识库空间（space_id）

## 完整命令序列

### 1. 认证（首次使用）

```bash
lark-cli auth login --domain base
```

会打开飞书授权页面，扫码授权后即可。授权范围包含 `base:table:read`、`base:table:write`、`base:record:create` 等。

### 2. 在知识库中创建 bitable 节点

```bash
lark-cli wiki +node-create --space-id <SPACE_ID> --obj-type bitable --title "小红书收藏整理" --as user
```

返回值包含 `node_token`（知识库节点 token）和 `obj_token`（即 base_token）。

**示例输出：**
```json
{
  "node_token": "KdKewCgpIipQPFkXytwcB4Q0nDc",
  "obj_token": "UPdDbo5llabsess9dYQc3lm0naH"
}
```

### 3. 获取默认表 ID

```bash
lark-cli base +table-list --base-token <BASE_TOKEN> --as user
```

新建的 bitable 会有一个默认表，返回其 `table_id`。

### 4. 获取默认字段 ID

```bash
lark-cli base +field-list --base-token <BASE_TOKEN> --table-id <TABLE_ID> --as user
```

默认表会有几个默认字段（如"多行文本"、"数字"等），需要修改或删除。

### 5. 配置字段

#### 修改默认字段为"标题"

```bash
lark-cli base +field-update --base-token <BASE_TOKEN> --table-id <TABLE_ID> --field-id <FIELD_ID> --json '{"type":"text","name":"标题"}' --as user
```

#### 创建"作者"字段

```bash
lark-cli base +field-create --base-token <BASE_TOKEN> --table-id <TABLE_ID> --json '{"type":"text","name":"作者"}' --as user
```

#### 创建"链接"字段（URL 样式）

```bash
lark-cli base +field-create --base-token <BASE_TOKEN> --table-id <TABLE_ID> --json '{"type":"text","name":"链接","style":{"type":"url"}}' --as user
```

#### 创建"是否学习"字段（复选框）

```bash
lark-cli base +field-create --base-token <BASE_TOKEN> --table-id <TABLE_ID> --json '{"type":"checkbox","name":"是否学习"}' --as user
```

#### 删除不需要的默认字段

```bash
lark-cli base +field-delete --base-token <BASE_TOKEN> --table-id <TABLE_ID> --field-id <UNUSED_FIELD_ID> --yes --as user
```

### 6. 重命名表

```bash
lark-cli base +table-update --base-token <BASE_TOKEN> --table-id <TABLE_ID> --name "收藏列表" --as user
```

### 7. 准备批量数据文件

创建 `batch_data.json`，格式如下：

```json
{
  "fields": ["标题", "作者", "链接", "是否学习"],
  "rows": [
    ["笔记标题", "作者名", "https://www.xiaohongshu.com/user/profile/...?xsec_token=...", false],
    ["第二条笔记", "另一个作者", "https://...", false]
  ]
}
```

**注意事项：**
- `fields` 数组必须与 `rows` 中每行的顺序一一对应
- URL 字段的值必须是完整的 URL 字符串
- checkbox 字段的值为 `true` 或 `false`
- 单次批量创建最多 **200 条记录**

### 8. 批量导入

```bash
cd <数据文件目录>
lark-cli base +record-batch-create --base-token <BASE_TOKEN> --table-id <TABLE_ID> --as user --json @batch_data.json
```

**重要：** `--json @file` 要求文件路径是**相对路径**，需先 `cd` 到文件所在目录再执行。

### 9. 超过 200 条的分批写入

如果数据超过 200 条，需要分批：

```python
import json
import subprocess
import time

def batch_import(data_file, base_token, table_id, batch_size=200):
    with open(data_file, 'r', encoding='utf-8') as f:
        data = json.load(f)

    rows = data['rows']
    fields = data['fields']

    for i in range(0, len(rows), batch_size):
        batch = rows[i:i+batch_size]
        batch_data = {'fields': fields, 'rows': batch}

        batch_file = f'batch_{i//batch_size}.json'
        with open(batch_file, 'w', encoding='utf-8') as f:
            json.dump(batch_data, f, ensure_ascii=False)

        subprocess.run([
            'lark-cli', 'base', '+record-batch-create',
            '--base-token', base_token,
            '--table-id', table_id,
            '--as', 'user',
            '--json', f'@{batch_file}'
        ], check=True)

        print(f"Imported batch {i//batch_size + 1}: {len(batch)} records")
        time.sleep(1)  # 批次间延迟
```

## 返回给用户的信息

导入完成后，返回飞书知识库链接：

```
https://<domain>.feishu.cn/wiki/<node_token>
```

## 常见错误

| 错误 | 原因 | 解决 |
|------|------|------|
| `missing required scope(s)` | 未授权 base 权限 | 执行 `lark-cli auth login --domain base` |
| `--file must be a relative path` | `--json @` 使用了绝对路径 | 先 `cd` 到文件目录，再用相对路径 |
| `record count exceeds limit` | 单次超过 200 条 | 分批写入 |
| `invalid field type` | 字段类型不匹配 | 检查字段定义和数据格式 |
| `timeout` | 网络超时 | 重试，或减少批次大小 |
