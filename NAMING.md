---
name: s3-naming-convention
description: S3（models/datasets/runtime/scratch）对象前缀的命名公约：一级前缀必须用内部 codename（如 egoDag），禁止无语义时间戳名（ckpt_261001）和模棱两可通用名（dataset/vla）；同一 codename/同一使用范围的数据必须收在同一路径下。凡在 S3 上新建前缀、上传数据、申请写权限或 review 他人前缀时使用。
---

# S3 前缀命名公约

## 核心规则

**路径的第一段（桶下第一级前缀）必须回答「哪个项目的什么」，用内部 codename。**

```
<bucket>/<codename>/<子路径…>     ✅ 唯一合法形态
```

- ✅ `models/egoDag/…`、`datasets/egoDag/vla-0317/…`、`models/cosmos3exp/…`、`models/glm52/…`
- ❌ `models/ckpt_261001/…` —— 时间戳/编号不是名字，一个月后没人知道这是什么
- ❌ `datasets/dataset/vla/…` —— 通用词堆叠、模棱两可：桶名已经说了是 dataset，
  `vla` 不指向任何项目（是哪条线的 VLA？谁在用？）
- ❌ `test`、`tmp`、`data1`、`new`、`final` 之类占位名

## 细则

1. **一级前缀 = codename**：项目/模型的内部代号（egoDag、cosmos3exp、glm52 式）。
   不知道 codename 就先问清楚再传，不要先随便建一个「以后再说」。
2. **推荐驼峰命名**（`egoDag`、`egoAnnoQueue`、`DexCanvas` 式）；同一个名字的
   大小写/连字符变体视为违规分裂（`egoAnnoQueue` 与 `ego-anno-queue` 并存 =
   违规，应合并为驼峰正形）。公开模型/方法原名（sam3、hawor、hot3d）保持
   官方拼写即可。
3. **同 codename 同路径**：同一 codename 或同一使用范围的数据必须收在同一
   `<bucket>/<codename>/` 下，不散落到多个一级前缀；版本/实验/日期区分的
   层级放 codename **之下**（如 `egoDag/ckpt/20261001/`，不是
   `ckpt_261001/`）。
4. **scratch 是唯一例外**：一级前缀用个人名字（`scratch/<你的名字>/`），
   因为 scratch 是 30 天个人临时区，不是资产。
5. **申请写权限时**：@oss-bot 申请的前缀必须符合本公约；不合规前缀的申请
   应被驳回，已存在的不合规前缀应迁移整改。
6. **review 口径**：看到一级前缀不是 codename（时间戳、通用词、占位名），
   即视为公约违规，要求整改。

## 现状一级前缀判定（2026-11-02 用户裁定，yes=合规 no=违规）

| 前缀 | 判定 | 理由 |
|---|---|---|
| `models/cosmos3exp/`、`models/glm52/` | yes | codename |
| `models/sam3/`、`models/hawor/`、`models/foundationstereo/`、`models/dinov2/`、`models/droid/`、`models/unidepth/`、`models/s2m2/`、`models/stereoanywhere/` | yes | 公开模型/方法官方名，视为 codename |
| `models/checkpoints/` | no | 通用词，不知归属项目 |
| `models/vla/` | no | 模棱两可通用词 |
| `models/ego-pipeline/` | no | 用途描述非 codename（应为 `egoDag` 类代号；且连字符非驼峰） |
| `models/minimax-h3/`、`models/hands-wilor/`、`models/fast-foundationstereo/`、`models/monster/`、`models/da3/` | yes | 可识别的模型/项目名（灰色但可接受；新前缀请用驼峰 codename） |
| `datasets/DexCanvas/`、`datasets/EgoSuite-Open100K/`、`datasets/egodex/`、`datasets/hoi4d/`、`datasets/hot3d/`、`datasets/arctic/`、`datasets/taco/` | yes | 项目/公开数据集名 |
| `datasets/benchmarks/`、`datasets/proj-verify/` | no | 通用词 |
| `datasets/mocap_v0/` | no | 通用词+版本号占一级（版本层应在 codename 之下） |
| `datasets/vla/`、`datasets/4dgs/` | no | 模棱两可技术词 |
| `datasets/LightwheelAI/`、`datasets/mocap-chingmu/`、`datasets/lightwheel-cosmos-caption/`、`datasets/ego-cam/`、`datasets/archebase/`、`datasets/lance/`、`datasets/openaoe/`、`datasets/stb/`、`datasets/stellanex/` | yes | 可识别来源/项目名（灰色可接受） |
| `runtime/egoAnnoQueue/` + `runtime/ego-anno-queue/` 并存 | no | 同一服务两种写法分裂，合并为驼峰 `egoAnnoQueue` |
| `runtime/a800_data/`、`runtime/imu_data/`、`runtime/uploads/` | no | 通用词/机器名 |
| `runtime/wgz/` | no | 人名缩写；runtime 一级前缀应是 codename/服务名 |
| `runtime/cosmos3/`、`runtime/cosmos3exp/`、`runtime/manoRL/`、`runtime/paper-reader/`、`runtime/ego-anno-queue/`（合并后） | yes | codename/服务名 |
| `scratch/<个人名>/`（chensiyuan、lyw…） | yes | scratch 例外条款 |
| `scratch/egoAnnoQueue/` | no | 服务名不应进个人临时区（应去 runtime） |

## 依据

S3 是团队正式资产库：前缀名是唯一自描述元数据（无目录语义、无强制标签）。
名字必须让不上下文的人一年后仍能定位归属项目和用途。用户裁定
（2026-11-02）：「前缀越靠前越不应该出现 ckpt_261001 这种名字，也不能出现
/dataset/vla 这种模棱两可的名字；应当尽可能使用内部 codename；同一
codename 或使用范围的数据应放到同一个路径」。2026-11-03 补充裁定：判定表
直接按 yes/no 记录，鼓励驼峰命名。
