---
name: autohome-koubei-to-feishu-base
description: 把汽车之家口碑HTML页面的各小点(最满意/空间/续航/驾驶感受/外观/内饰/性价比/智能化等)提取出来,改写后批量写入飞书多维表格。当用户给出汽车之家口碑HTML文件,并要求"改写入多维表格/飞书Base/批量改写"时使用。
agent_created: true
---

# 汽车之家口碑 → 飞书多维表格 批量改写流程

## 适用场景

用户保存了一份汽车之家口碑详情页 HTML(通常在 `C:/Users/Administrator/Downloads/<车型>怎么样_优缺点_<用户>_口碑_汽车之家.html`),希望:
- 提取其中各个口碑小点(标题+评分+正文)
- 在原文基础上改写每条(用于二次创作/规避查重/批量内容生产)
- 把结果批量写入飞书多维表格

## 标准工作流

### Step 1: 提取小点(BeautifulSoup)

汽车之家口碑页的小点 anchor 是文本行的「最满意 / 最不满意 / 空间 / 驾驶感受 / 续航 / 外观 / 内饰 / 性价比 / 智能化」。

- 用 `soup.get_text('\n', strip=True)` 转纯文本
- 按行扫,每行 strip 后精确等于 anchor 关键词的,记为一个小点
- 第一个「空间」往往是页面头部装饰(紧跟"快速回复/发表口碑"),内容里含 `快速回复` 就丢弃
- 评分:小点标题下一行如果是 1-5 的整数,提取为 score
- 正文尾部从「上述内容的版权归」开始全部截掉(页脚版权噪音)

参考脚本: `D:\项目\汽车之家\.workbuddy\scratch\extract_koubei.py` + `clean_koubei.py`

### Step 2: 改写(模型生成)

为每个小点手写一段保留事实(车型/数字/细节)、但句式完全不同的改写版本。要点:
- 保留所有具体数字(4910mm/2910mm/CLTC 500km/180cm/80kg/12万/4天1700公里等)
- 语气自然像真人口吻,避免生硬的同义词替换
- 长度可以与原文有 ±20% 差异,不必严格等长

### Step 3: 建飞书多维表格

```bash
lark-cli base +base-create \
  --name "<表名>" \
  --table-name "<首个数据表名>" \
  --fields '[{"name":"小点","type":"text"},{"name":"评分","type":"number"},{"name":"原文","type":"text"},{"name":"改写后","type":"text"}]' \
  --time-zone Asia/Shanghai
```

记下返回的 `base_token` 和 `table.id`。

### Step 4: 批量写入(带重试)

只有 `+record-upsert`(逐条),没有 batch-create。每条写一个 JSON:

```python
fields = {'小点': name, '原文': original, '改写后': rewritten}
if score is not None:
    fields['评分'] = score
```

## 关键坑位(必须避开)

1. **lark-cli 绝对路径**:Python `subprocess` 调 `lark-cli` 必须用 `.cmd` 完整路径:
   ```
   C:\Users\Administrator\.workbuddy\binaries\node\cli-connector-packages\lark-cli.cmd
   ```
   裸 `lark-cli` 会 `FileNotFoundError`。

2. **record-list 返回结构**:真实结构是
   ```json
   {"data": {"data": [[...每行字段值数组...]], "record_id_list": ["rec..."], "fields": ["小点","原文","改写后","评分"]}}
   ```
   **不是** `data.records`。取 record_id 用 `data.record_id_list`,取每行字段值用 `data.data` + `data.fields` zip。

3. **record-search 不能空 keyword**:想列全部记录就用 `+record-list`,别用 search。

4. **网络瞬时失败**:lark-cli 偶发 `read tcp ... wsar...` 错误,**必须重试**(建议 5 次,每次间隔 2 秒)。

5. **沙箱无网络**:沙箱 Python 不能 pip install。需要新依赖时改用系统 Python:
   ```
   C:\Users\Administrator\AppData\Local\Programs\Python\Python313\python.exe
   ```
   并配合 `--break-system-packages` 或已存在的 venv。

6. **Windows 终端 GBK 乱码**:Python 脚本里加
   ```python
   import sys, io
   sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8')
   ```
   或运行时加 `-X utf8`。

7. **去重**:写入前先 `+record-list` 拿 `record_id_list`,如果已经有同名小点就先批量 `+record-delete`(要带 `--yes`,每个 id 一个 `--record-id`),避免重跑导致重复记录。

8. **Windows 传中文 JSON payload 给 lark-cli 必挂(2026-07-21 实测)**:`subprocess.run([lark-cli.cmd, ..., '--json', '{"中文字段":"..."}'])` 会报 `'C:' 不是内部或外部命令` 或静默失败——CreateProcess 对含非 ASCII 的长命令行解析有 bug。**唯一可靠方案:`--json @文件`**
   ```python
   # payload 写临时文件(UTF-8),@文件名用纯ASCII相对路径,subprocess 传 cwd
   with open('_payload.json', 'w', encoding='utf-8') as f:
       json.dump(payload, f, ensure_ascii=False)
   subprocess.run([LARK, 'base', '+record-upsert', ...,
                   '--json', '@_payload.json', '--format', 'json'],
                  cwd=SCRATCH_DIR, capture_output=True)
   ```
   注意:@后的路径必须是**相对路径**(绝对路径含中文会被 lark-cli 拒绝 "invalid JSON file path"),所以要 `cwd=` 配合。

9. **cmd.exe 包装不可行**:试图写临时 .cmd 文件内嵌 JSON(用 `\"` 转义)也行不通——cmd.exe 不把 `\"` 当转义引号,报「文件名、目录名或卷标语法不正确」。别再试这条路,直接用 `--json @file`。

10. **subprocess 读 lark-cli 输出要二进制**:`capture_output=True, text=True, encoding='utf-8'` 会在 lark-cli 输出混 GBK 字节时炸 `UnicodeDecodeError`(异常在线程里,主流程静默失败难排查)。改为 `capture_output=True` 拿 bytes,自己 `.decode('utf-8', errors='ignore')`,且 text/err 两路都要尝试解析 JSON。

11. **record-upsert 的 payload 不能用 `fields` 包裹(2026-07-22 实测)**:payload 里字段要**平铺在顶层**,更新已记录时 record_id 走 CLI 参数 `--record-id`,不要放进 payload。错误写法 `{"fields": {...}, "record_id": "rec..."}` 会报 `800010701 Record write payload must not be wrapped in 'fields'`。正确:
    ```json
    {"小点": "整篇", "改写后": "...", "处理状态": "已改写"}
    ```
    ```python
    [LARK, 'base', '+record-upsert', '--base-token', B, '--table-id', T,
     '--json', '@_payload.json', '--record-id', rid, '--format', 'json']
    ```
    (注意:旧脚本 push_v2_final.py 用的就是错误包裹格式,勿再参考;新参考 push_v4.py)

## 参考产物

- 提取脚本:`D:\项目\汽车之家\.workbuddy\scratch\extract_koubei.py`
- 清洗脚本:`D:\项目\汽车之家\.workbuddy\scratch\clean_koubei.py`
- 改写脚本:`D:\项目\汽车之家\.workbuddy\scratch\rewrite_koubei.py`
- 推送脚本:`D:\项目\汽车之家\.workbuddy\scratch\clear_and_repush.py`

最终成果示例(Base): https://my.feishu.cn/base/Uphbbgprjad1d7sIKpNcG1Aenfd
