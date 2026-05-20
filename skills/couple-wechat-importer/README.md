# 微信聊天记录读取器

> 从微信数据库提取聊天记录，为情侣复盘提供结构化素材

---

## 效果展示

提取方A × 方B 聊天记录的分析结果：

| 指标 | 数值 |
|-----|------|
| 总消息 | N 条 |
| 日期范围 | YYYY-MM-DD ~ YYYY-MM-DD |
| 峰值日 | MM-DD（N条） |
| 发言比例 | 方A X% / 方B X% |
| 深夜占比 | XX% |
| 最长沉默 | X小时 |

---

## 快速开始

### Step 1：提取 Key + 解密数据库

确保微信正在运行（Key 从内存提取），在 skills 目录下执行：

```python
import sys, os, json

# 切换到工具目录
TOOLS_DIR = os.path.dirname(os.path.abspath(__file__))
sys.path.insert(0, TOOLS_DIR)
os.chdir(TOOLS_DIR)

from wxManager.decrypt.get_wx_info import read_info
with open('./wxManager/decrypt/version_list.json', encoding='utf-8') as f:
    vl = json.load(f)

result = read_info(vl)
key = result[0]['key']
wxid = result[0]['wxid']
print(f'Key: {key}')
print(f'wxid: {wxid}')

# 解密数据库
from wxManager.decrypt.decrypt_v3 import decrypt_db_files
src = fr'D:\\Documents\\WeChat Files\\{wxid}\\Msg'
out = r'D:\\Claude_Notes\\wechat_decrypted'
os.makedirs(out, exist_ok=True)
decrypt_db_files(key, src, out)
print('解密完成！')
```

**成功标志：** `errcode=200`，输出目录有 `MicroMsg.db` 和 `Multi/MSG0.db`

### Step 2：提取联系人列表

```bash
python tools/wechat_parser.py --db-dir D:/Claude_Notes/wechat_decrypted/ --list-contacts
```

输出示例：
```
找到 N 个联系人

微信ID                           备注                   昵称
----------------------------------------------------------------------
gh_826a68f6c123
wxid_xxxxxxxxxxx                某某                 张三
wxid_yyyyyyyyyyy                某人                 李四
...
```

### Step 3：提取聊天记录

```bash
python tools/wechat_parser.py --db-dir D:/Claude_Notes/wechat_decrypted/ --target "TA的昵称" --no-context
```

或指定 wxid：
```bash
python tools/wechat_parser.py --db-dir D:/Claude_Notes/wechat_decrypted/ --target "wxid_xxxxxxxxxxx"
```

### Step 4：分析聊天记录

用 Python 脚本分析，输出完整报告：

```python
import sqlite3
from datetime import datetime

db_path = 'D:/Claude_Notes/wechat_decrypted/Multi/MSG0.db'
wxid = 'wxid_xxxxxxxxxxx'  # TA 的 wxid

conn = sqlite3.connect(db_path)
c = conn.execute("""
    SELECT CreateTime, IsSender, Type, StrContent
    FROM MSG
    WHERE (StrTalker = ? OR StrTalker = ?) AND Type = 1 AND StrContent IS NOT NULL
    ORDER BY CreateTime ASC
""", (wxid, wxid + ':'))

messages = c.fetchall()
conn.close()

# 统计
total = len(messages)
dates = set(datetime.fromtimestamp(m[0]).strftime('%Y-%m-%d') for m in messages)
a = sum(1 for m in messages if m[1] == 1)
b = sum(1 for m in messages if m[1] == 0)

print(f'总消息: {total}')
print(f'日期范围: {min(dates)} ~ {max(dates)}')
print(f'方A: {a} 条 ({a/total*100:.1f}%)')
print(f'方B: {b} 条 ({b/total*100:.1f}%)')
```

---

## 工具结构

```
tools/
├── wechat_decryptor.py   # 解密主脚本
├── wechat_parser.py      # 解析脚本
└── wxManager/decrypt/
    ├── get_wx_info.py      # Key 提取（从微信进程内存）
    ├── decrypt_v3.py        # 数据库解密
    ├── decrypt_v4.py       # 数据库解密（v4版）
    ├── version_list.json    # 微信版本列表
    └── common.py           # 通用函数
```

---

## 常见问题

### Q: Key 提取失败？
**A:** 确保微信正在运行。微信重启后 Key 会变，需要重新提取。

### Q: 解密后数据库打不开？
**A:** Key 过期或版本不匹配。重新提取 Key 再试。

### Q: 找不到联系人？
**A:** 用 `--list-contacts` 查看完整列表，确认昵称或备注名。

### Q: 消息条数为 0？
**A:** 检查 wxid 是否正确。微信内部 ID 和微信号不同，用 `--list-contacts` 确认。

---

## 注意事项

1. **Key 有效期**：每次微信重启后必须重新提取
2. **数据安全**：解密后数据库仅本地使用，不要分发
3. **Windows 微信**：数据路径 `D:\Documents\WeChat Files\{wxid}\Msg`

---

*情侣复盘系统 · 第一步 | 2026-05-20*
