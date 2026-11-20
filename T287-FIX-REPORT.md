# 交付文档：修复多文档规则集中补丁文档永久改写全局模板的缺陷

## 1. 现象与根因

### 1.1 现象

多文档规则集文件按 YAML 文档顺序装载，文档动作语义为：

- `action: global`（模板文档）：不产生规则，仅把文档内容存为全局模板，给后续普通规则补全局属性；
- `action: repeat`（补丁文档）：把增量内容并入**前一条规则**后解析产出规则；
- 无 action 的普通文档：与全局模板深合并后解析为规则。

当模板文档之后出现补丁文档时，补丁内容会**永久改写全局模板**，导致排在补丁之后的每一条普通规则静默继承补丁文档的标识（id）、标题（title）、描述（description）与检测逻辑（detection），后果为：

1. 规则集内出现重复规则标识，下游按 id 建立的检索索引发生别名覆盖；
2. 普通规则的检测条件被改写成指向补丁带来的外来检测项，规则语义被整体替换；
3. 下游检索侧去重与关联引用全部失真。
4. 同一文件去掉模板文档的对照组不触发该问题（无 `global_rule` 即无共享可变状态）。

### 1.2 根因（定位到 `sigma/collection.py` 的 `SigmaCollection.from_dicts`）

修复前的多文档动作处理维护两个状态变量：

```python
prev_rule = dict()
global_rule: NestedDict = dict()
```

三个关键分支的对象身份关系如下：

```python
# global 分支
global_rule = rule
prev_rule = global_rule          # ← prev_rule 与 global_rule 指向同一个可变 dict

# repeat 分支
prev_rule = deep_dict_update(prev_rule, rule)   # ← 就地改写 prev_rule，即改写 global_rule

# 普通规则分支
deep_dict_update(rule, global_rule)             # ← 合并到的已是被补丁污染的模板
```

`deep_dict_update` 是**就地（in-place）更新并返回目标 dict** 的函数。`global` 分支让 `prev_rule` 与 `global_rule` 形成别名（alias），随后 `repeat` 分支通过 `prev_rule` 就地写入补丁字段时，等价于直接改写全局模板对象。模板被污染后，后续普通规则合并到的 `global_rule` 已携带补丁的 id/title/description/detection，于是批量产生与补丁同标识、同检测逻辑的规则。

此外，普通分支 `deep_dict_update(rule, global_rule)` 的第二个参数在嵌套 dict 层面同样会被就地遍历，模板与工作副本之间缺少边界，属于同类别名风险。

### 1.3 为什么去掉模板文档的对照组干净

没有 `global` 文档时 `global_rule` 始终为空 dict，`prev_rule` 只在普通规则与 repeat 之间流转，不存在跨规则生命周期的共享模板对象，补丁的就地更新没有长期存活的受害者，与审计侧观测一致。

## 2. 修复方案与取舍理由

### 2.1 方案：边界处深拷贝，让全局模板成为私有不可变状态

在「状态存入」与「状态合并」两个边界统一用 `copy.deepcopy` 切断别名，所有补丁合并都在独立工作副本上发生：

| 分支 | 修复前 | 修复后 | 理由 |
| --- | --- | --- | --- |
| `global` | `global_rule = rule`；`prev_rule = global_rule` | `global_rule = deepcopy(rule)`；`prev_rule = deepcopy(global_rule)` | 模板与输入文档、与 `prev_rule` 三方彻底解耦，模板此后不被任何就地更新触及 |
| `repeat` | `deep_dict_update(prev_rule, rule)` | `deep_dict_update(deepcopy(prev_rule), rule)` | 每次补丁先复制工作副本再叠加，连续补丁的累积结果只在 `prev_rule` 链上流动，永不回流到模板 |
| 普通规则 | `deep_dict_update(rule, global_rule)` | `deep_dict_update(rule, deepcopy(global_rule))` | 合并在模板的副本上进行，深合并对嵌套 dict 的就地遍历不会触及模板本身（防御性，消除残余别名面） |

### 2.2 共享形态取舍

题面允许自行决定「修复方式与共享形态」。评估过两个方向：

- **方向 A：保持共享单例模板 + 全量深拷贝隔离（采用）**。对外装载入口（`from_dicts`/`from_yaml`/`load_ruleset`）的签名、返回结构与调用方式完全不变，灰度探针组与全量检索节点在滚动升级并存期继续共用同一公开装载入口，无需任何调用侧改造；隔离成本收敛在装载函数内部。
- **方向 B：取消共享，每文档携带独立模板快照（不采用）**。需要改变状态在分支间的传递形态，改动面会外溢到动作处理的控制流，且与既有 `reset`/`repeat` 语义的耦合点更多，回归风险大。

选择方向 A 的理由：缺陷本质是**可变共享状态缺少所有权边界**，而非共享本身有错。深拷贝把模板的「读共享」保留、把「写隔离」补齐，是最小语义扰动的修法；规则集文档数量通常为个位数到数十个，深拷贝纯数据 dict 的开销相对 YAML 解析与规则对象构建可忽略，不构成发布管道的吞吐瓶颈。

### 2.3 既有语义保持情况

| 题面要求保持的语义 | 保持方式 |
| --- | --- |
| 模板给普通规则注入全局属性 | 普通分支仍执行 `deep_dict_update(rule, 模板副本)`，合并顺序与覆盖规则（模板叶子键覆盖同名键、文档独有键保留）一字未改 |
| 补丁并入前一条规则 | `repeat` 分支仍以 `prev_rule` 为基底叠加当前文档后整体解析，产出规则的内容与顺序不变 |
| 连续补丁依次叠加的累积语义 | `prev_rule` 每次承载上一轮补丁产物的副本再叠加新补丁，累积链不中断；新增专门用例验证两级补丁的检测项并集与 condition 覆盖 |
| 规则集内规则标识不得重复 | 模板不再被补丁改写后，普通规则恢复使用模板标识，与补丁标识天然不同；用例对 id 列表去重计数并校验 id 索引不发生别名覆盖 |

## 3. 改动面登记（按文件逐一定位）

改动面严格收在**规则集装载实现及其测试**两个文件，未触碰规则解析、后端转换、管道等其他模块。

### 3.1 `sigma/collection.py`（规则集装载实现，4 处）

| 位置 | 改动 |
| --- | --- |
| 第 4 行（import 区） | 新增 `from copy import deepcopy` |
| `from_dicts` 普通规则分支（约 215 行） | `deep_dict_update(rule, global_rule)` → `deep_dict_update(rule, deepcopy(global_rule))`，并加注释说明模板私有 |
| `from_dicts` 的 `global` 分支（约 224–229 行） | `global_rule = rule`/`prev_rule = global_rule` → 两处 `deepcopy`，切断模板与输入文档、与 `prev_rule` 的别名 |
| `from_dicts` 的 `repeat` 分支（约 234–236 行） | `deep_dict_update(prev_rule, rule)` → `deep_dict_update(deepcopy(prev_rule), rule)`，补丁在独立副本上累积 |

### 3.2 `tests/test_collection.py`（既有测试文件，新增用例落位于此）

在文件末尾新增 1 个最小样例常量组（dict 形态与 YAML 形态各一份）与 5 个测试函数（其中 1 个参数化为 10 轮）：

| 新增用例 | 覆盖的断言类别 |
| --- | --- |
| `test_global_template_not_mutated_by_repeat_patch_attributes` | **属性一致**：补丁后普通规则的 id/title/description 与模板（自身文档 + 全局注入）一致，不继承补丁标识与文案；同时校验补丁产物自身属性正确 |
| `test_global_template_not_mutated_by_repeat_patch_detection` | **检测逻辑一致**：普通规则只含模板检测项 `global_sel` 与模板 condition；补丁产物保持模板+补丁的累积检测项 |
| `test_global_template_not_mutated_by_repeat_patch_unique_identifiers` | **标识唯一**：id 列表无重复，且 `collection[UUID(...)]` 索引指向正确规则对象 |
| `test_consecutive_repeat_patches_accumulate_without_template_mutation` | 连续两级补丁的累积叠加语义 + 模板不被任一补丁改写 + 全程标识唯一 |
| `test_global_patch_ruleset_stable_across_ten_load_rounds[0..9]` | 同一最小多文档 YAML 经公开入口 `from_yaml` 重复装载 10 轮，每轮三类断言全部成立 |

既有 33 个用例零修改、零删除。

## 4. 题面要求与改动的逐条对应关系

| # | 题面要求 | 对应落点 |
| --- | --- | --- |
| 1 | 定位根因并修复 | 本文件 §1.2（别名 + 就地更新）；`sigma/collection.py` 三处分支改动 |
| 2 | 补丁之后的普通规则属性与检测逻辑必须与自身文档一致 | 普通分支模板副本合并；属性/检测两个用例 |
| 3 | 规则集内规则标识不得重复 | 模板不再被污染；标识唯一用例（含 id 索引别名校验） |
| 4 | 模板注入全局属性的既有语义保持 | 普通分支合并顺序与覆盖规则未变；既有 `test_action_global` 等全部通过 |
| 5 | 补丁并入前一条规则的既有语义保持 | `repeat` 分支基底仍为 `prev_rule`；用例校验补丁产物内容 |
| 6 | 连续补丁依次叠加的累积语义保持 | `prev_rule` 副本链；两级补丁专用用例 |
| 7 | 修复方式与共享形态取舍自行设计并写明理由 | 本文件 §2.1、§2.2 |
| 8 | 改动面按文件逐一定位登记 | 本文件 §3 |
| 9 | 各条题面要求与改动对应关系逐条成立 | 本表格 |
| 10 | 验收命令与构造输入两组实测读数随文给出 | 本文件 §5、§6 |
| 11 | 构造输入为模板后补丁的最小多文档样例，快照环境零外部依赖自建，重复装载 10 轮 | §6 样例；仅依赖仓内 `sigma` 包与 PyYAML；10 轮脚本与参数化用例 |
| 12 | 每轮读数含规则条数、补丁后普通规则四要素、重复标识计数 | §6.2 读数表 |
| 13 | 新用例落位于既有 `tests/test_collection.py`，三类断言补齐并沿用既有框架 | §3.2，pytest 函数式风格与既有用例一致 |
| 14 | 改动面收在装载实现及其测试，既有测试保持通过 | 仅两个文件改动；§5 验收结果 |
| 15 | 验收以 `python -m pytest tests/test_collection.py -q` 零失败为准 | §5.1 |
| 16 | 改动区域为规则集装载的多文档动作处理 | 全部代码改动位于 `from_dicts` 的 action 分支 |

## 5. 验收命令与实测结果

### 5.1 题面指定验收命令

```
python -m pytest tests/test_collection.py -q
```

实测结果（修复后）：

```
47 passed in 0.52s
```

构成：修复前基线 33 个用例全部保持通过 + 新增 4 个命名用例 + 10 轮参数化装载用例。

### 5.2 全量回归

```
python -m pytest tests/ -q
```

实测结果：`1959 passed, 2 skipped, 1 error`。其中唯一 error 为 `tests/files/template_vars.py` 被 pytest 误收集的既有环境问题（该文件是测试夹具而非测试模块），在未改动的干净 HEAD 上同样存在；本次改动未引入任何新增失败。

## 6. 构造输入与 10 轮实测读数

### 6.1 构造输入（最小多文档规则集，零外部依赖）

三个 YAML 文档：`global` 模板 → `repeat` 补丁 → 补丁之后的普通规则（仅含 logsource，其余属性靠模板注入）。

```yaml
action: global
title: Global Template Title
id: 11111111-1111-1111-1111-111111111111
description: global template description
logsource:
    category: template_cat
detection:
    global_sel:
        GlobalField: global_value
    condition: global_sel
---
action: repeat
title: Patch Title
id: 22222222-2222-2222-2222-222222222222
description: patch description
detection:
    patch_sel:
        PatchField: patch_value
    condition: patch_sel
---
logsource:
    category: ordinary_cat
```

### 6.2 修复后 10 轮读数（每轮均同）

| 轮次 | 规则条数 | 规则 id 序列 | 重复标识计数 | 补丁后普通规则 id | 标题 | 描述 | 检测项 | condition |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1–10（每轮一致） | 2 | `['22222222-…', '11111111-…']` | 0 | `11111111-1111-1111-1111-111111111111` | `Global Template Title` | `global template description` | `['global_sel']` | `['global_sel']` |

每轮补丁产物（rule[0]）读数：id `22222222-…`、title `Patch Title`、description `patch description`、检测项 `['global_sel', 'patch_sel']`、condition `['patch_sel']`——模板与补丁累积合并，补丁语义完整保留。

### 6.3 修复前对照读数（同一构造输入，同一公开入口）

第 1 轮即失败：

```
rule_count=2
rule_ids=['22222222-2222-2222-2222-222222222222', '22222222-2222-2222-2222-222222222222']
duplicate_id_count=1
ordinary_rule[1] id=22222222-2222-2222-2222-222222222222 title='Patch Title'
  description='patch description' detection_names=['global_sel', 'patch_sel']
  condition=['patch_sel']
```

普通规则被补丁整体串改（标识重复、标题/描述替换、检测逻辑指向外来检测项 `patch_sel`），与审计侧发现的故障形态完全吻合，反向印证根因定位。
