# DexGEM H200 S3 使用教程与 Demo

> 面向 H200 Coder workspace 用户；家目录 `~/README.md` 托管块同源。

## S3 使用教程：数据读写走 API，禁止从 /mnt/oss 读数据

`/mnt/oss` 是 rclone FUSE **只读浏览视图**，唯一用途：看目录结构、读各桶
`00-README.md`、确认对象是否存在。**它不是数据通道**：

- ❌ 训练/推理直接读 `/mnt/oss/models/...` 权重、`/mnt/oss/datasets/...` 数据
- ❌ `cp` / `rsync` / `tar` / `cat` / `python open()` 从 `/mnt/oss` 大文件或批量读取
- ❌ DataLoader、checkpoint 加载、任何程序内路径参数指向 `/mnt/oss/...`

原因：FUSE 把每次 open/read 翻译成独立 S3 请求、无文件缓存，吞吐远低于
200G fabric 直连，批量读取会放大成海量请求打向网关；目录缓存 30 秒，可能
读到过期视图；挂载是尽力服务，训练中途掉挂载会以诡异 I/O 错误失败。

**正确姿势**：凭据与工具已预配置（`~/.aws`、rclone remote `oss`、boto3），
零配置直连 S3 端点，先把数据拉到本地盘（大文件首选 `/tmp`，需跨重启保留
放 `/home/coder`）再使用：

```bash
# 浏览（/mnt/oss 仅限这类用途）
ls /mnt/oss/models/
# 列举（零配置）
aws s3 ls s3://models/
# 拉模型/数据集到本地（大批量推荐 s5cmd，端点见 /etc/oss-guide.md）
aws s3 cp --recursive s3://models/<前缀>/ /tmp/dst/
s5cmd --endpoint-url "$(awk '/endpoint_url/{print $3}' ~/.aws/config)" sync s3://datasets/<前缀>/ /tmp/dst/
# 上传/删除 scratch（免申请；放 <你的名字>/ 前缀下）
aws s3 cp ./out.bin s3://scratch/<你的名字>/out.bin
# 程序内访问：boto3，endpoint_url 取 /etc/oss-guide.md 的 fabric 端点，
# 不要把 /mnt/oss 路径传给程序
```

| 场景 | 做法 |
|---|---|
| 看目录 / 读桶说明 | `ls /mnt/oss/<桶>/`（仅此处允许用 FUSE） |
| 拉模型权重 / 数据集 | `aws s3 cp` / `s5cmd sync` 到 `/tmp` 或 `/home/coder` |
| 程序内读 S3 | boto3 直连端点，不传 `/mnt/oss` 路径 |
| 上传 / 删除 | `aws s3 cp` / `aws s3 rm`（scratch 免申请；models/datasets/runtime 写权限 @oss-bot 申请） |

完整端点/凭据/权限语义/申请语法：`/etc/oss-guide.md`；
桶内副本 `/mnt/oss/00-readme/README.md`。

### 两条访问路径：pod 内走 fabric，公网只是外部入口

**这是性能差异最大的一个事实，务必分清：**

| 路径 | 端点 | 谁在用 | 实测吞吐 |
|---|---|---|---|
| **fabric 直连（首选）** | `http://10.200.0.10:19000` | pod 内一切 S3 流量（本 workspace 已预配置，aws/boto3 零配置默认就是它） | 200G 专线；宿主基准 14 GB/s |
| 公网入口 | `https://oss.dex-gem.ai:1443` | **仅供集群外**（笔记本/外部服务器/外部协作者） | ~50 MiB/s 封顶，并发无用 |

- pod 内**永远不要**把端点改成公网 :1443 或 LAN :9000——绕远路且慢 20 倍以上；
  `~/.aws/config` 里的 `endpoint_url` 就是 fabric，不要动。
- 外部使用（给协作者/在自己笔记本跑）：凭据用同一个团队 key
  （`svc-team-ro`，值见 pod 内 `/etc/oss-guide.md` 或找平台），配置：

```bash
# 外部机器上 ~/.aws/config 加一个 profile（不要覆盖默认）
# [profile oss-public]
# region = us-east-1
# endpoint_url = https://oss.dex-gem.ai:1443
aws --profile oss-public s3 ls                       # 外部列举
aws --profile oss-public s3 cp s3://scratch/<名字>/f.bin .   # 外部下载
```

公网路径只适合小文件/应急；**大批量数据搬运请把任务挪进 workspace 里跑**
（pod 内 fabric 直连），不要从外面拉。

### 命名公约（上传前必读）

**路径第一段（一级前缀）必须是内部 codename**（`egoDag`、`cosmos3exp`、
`glm52` 式），回答「哪个项目的什么」：

- ✅ `models/egoDag/…`、`datasets/egoDag/vla-0317/…`
- ❌ `models/ckpt_261001/…`（时间戳/编号不是名字）
- ❌ `datasets/dataset/vla/…`（通用词堆叠、模棱两可）
- 同一 codename / 同一使用范围的数据收在同一路径下，不散落；
  版本/日期层级放 codename 之下（`egoDag/ckpt/20261001/`）。
- 例外：`scratch` 一级前缀用个人名字（它是 30 天个人临时区）。

不合规前缀的写权限申请会被驳回，存量不合规前缀会被要求整改。

### 5 分钟上手 demo（pod 内逐条跑）

```bash
# 0. 确认走 fabric（必须输出 http://10.200.0.10:19000，否则报平台）
awk '/endpoint_url/{print $3}' ~/.aws/config
# 1. 列桶（应见 00-readme/models/datasets/runtime/scratch 五桶）
aws s3 ls
# 2. 生成 64MiB 测试文件并上传到 scratch 自己的前缀
dd if=/dev/urandom of=/tmp/s3demo-64m.bin bs=1M count=64
aws s3 cp /tmp/s3demo-64m.bin s3://scratch/<你的名字>/s3demo-64m.bin
# 3. 下载回来，md5 必须一致
aws s3 cp s3://scratch/<你的名字>/s3demo-64m.bin /tmp/s3demo-back.bin
md5sum /tmp/s3demo-64m.bin /tmp/s3demo-back.bin
# 4. 大对象读速自测（用固定锚点对象，读 1GiB 到 /dev/null；
#    勿用 aws CLI 多进程压测——python 在小 CPU 配额 pod 里会打满 CPU，数字是假的；
#    也不要用 aws s3 ls --recursive 找大对象——models 桶全量列举很慢）
curl -s -u svc-team-ro:$(awk '/aws_secret_access_key/{print $3}' ~/.aws/credentials) \
  --aws-sigv4 "aws:amz:us-east-1:s3" -r 0-1073741823 -o /dev/null \
  "http://10.200.0.10:19000/models/glm52/FP8/model-00005-of-00141.safetensors" \
  -w "speed: %{speed_download} B/s\n"
# 5. 清理现场
aws s3 rm s3://scratch/<你的名字>/s3demo-64m.bin && rm -f /tmp/s3demo-*.bin
```

外部机器版 demo：同上，只是每条 `aws` 换成 `aws --profile oss-public`，
第 0 步改为确认 `~/.aws/config` 里 `[profile oss-public]` 的
`endpoint_url = https://oss.dex-gem.ai:1443`。
