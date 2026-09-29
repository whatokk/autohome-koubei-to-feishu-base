# 口碑 HTML 转飞书多维表格

> WorkBuddy Skill · 屋里涛说

把汽车之家口碑详情页 HTML 里各小点（最满意/空间/续航/驾驶感受/外观/内饰/性价比/智能化等）提取出来，改写后批量写入飞书多维表格。

## 技能清单

| 技能 | 说明 |
|---|---|
| **autohome-koubei-to-feishu-base** · 小点提取改写入库 | 输入已保存的口碑详情页 HTML，输出写好的飞书 Base 记录。 |


## 安装

把 `skills/` 下的技能目录拷贝到 WorkBuddy 的技能目录：

```bash
cp -r skills/* ~/.workbuddy/skills/
```

Windows PowerShell：

```powershell
Copy-Item .\skills\* "$env:USERPROFILE\.workbuddy\skills\" -Recurse -Force
```

重启 WorkBuddy 后，技能列表即可看到。

## 使用要点

- 触发词：改写入多维表格、飞书Base、批量改写。
- 典型输入路径：`Downloads/<车型>怎么样_优缺点_<用户>_口碑_汽车之家.html`。

## 环境依赖

- Python 3.13
- 飞书连接器

## 目录规范

```
autohome-koubei-to-feishu-base/
└── skills/
    ├── autohome-koubei-to-feishu-base/
```

每个技能遵循统一结构：`SKILL.md`（必需，含 name/description frontmatter）+ `scripts/`（可选）+ `references/`（可选）。

---

## License

MIT — 随意取用、修改、二次分发。
