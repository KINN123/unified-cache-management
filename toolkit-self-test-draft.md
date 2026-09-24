# UCM Toolkit 自验报告

## 1. 测试环境

| 项目 | 配置 |
| --- | --- |
| 测试日期 | 2026.9.23 |
| 测试设备 | 8 × NVIDIA H100 80GB HBM3 |
| NVIDIA 驱动 | 580.173.02 |
| 内核 | 5.15.0-185-generic |
| vLLM 版本 | 0.27.1 |
| UCM 包版本 | uc-manager-cuda-cu130=0.8.0 |
| /dev/shm 总容量 | 1007.82 GiB |
| 容器镜像 | `registry.dev.huawei.com/flash_stor/vllm-openai:v0.27.1-ucm-0.8.0-r1` |
| Toolkit 版本 | `0.8.0rc1` |
| 安装包 | `ucm_toolkit-0.8.0rc1-py3-none-any.whl` |
| 容器名称 | `ucm-quickstart` |
| 容器平台 | `linux/amd64` |
| 执行用户 | root |
| 命令执行目录 | `/workspace/storage`、`/mnt` |
| 模型挂载 | `/mnt/model-2:/workspace/model` |
| 存储挂载 | `/data/ucm:/workspace/storage` |

### 1.1 容器启动命令

```bash
export IMAGE=registry.dev.huawei.com/flash_stor/vllm-openai:v0.27.1-ucm-0.8.0-r1
mkdir -p /data/ucm/log /data/ucm/cache
docker run --rm -it \
  --platform linux/amd64 \
  --entrypoint /bin/bash \
  --workdir /workspace \
  --gpus all \
  --network=host \
  --ipc=host \
  -v /mnt/model-2:/workspace/model \
  -v /data/ucm:/workspace/storage \
  --name ucm-quickstart \
  "$IMAGE"
```

在容器内的 `/workspace/storage` 目录执行以下安装与通用命令。

## 2. 安装与通用命令验证

| 编号 | 执行命令 | 实际结果 | 结论 |
| --- | --- | --- | --- |
| 1 | `python3 -m pip install ./ucm_toolkit-0.8.0rc1-py3-none-any.whl` | 输出 `Successfully installed ucm-toolkit-0.8.0rc1`，安装成功。 | 通过 |
| 2 | `ucm-toolkit --help` | 正常显示 list、doctor、build、clean、run 五个顶层命令及帮助说明。 | 通过 |
| 3 | `ucm-toolkit list` | 正常列出 dev-sandbox、metrics-view、nic-monitor、posix-aio、precheck 五个工具及简介。 | 通过 |
| 4 | `ucm-toolkit doctor` | 正常输出各工具检查结果；dev-sandbox 构建目录及二进制、nic-monitor 的 ethtool 显示为 MISSING。 | 检查命令正常运行，存在环境缺失项 |

### 2.1 安装结果

Toolkit `0.8.0rc1` 安装成功，安装后 `ucm-toolkit` 命令可用。安装过程中出现 root 用户运行 pip 的警告，但未影响安装及后续帮助、列表命令的执行。

### 2.2 环境检查结果

| 工具 | 实际输出 | 判断 |
| --- | --- | --- |
| dev-sandbox | source_dir 为 OK；build_dir、copy、trans、aio 为 MISSING。 | 源码目录存在，指定位置未找到构建目录和三个测试程序。 |
| metrics-view | `no environment checks` | 没有环境检查项。 |
| nic-monitor | script、bash 为 OK，ethtool 为 MISSING。 | 脚本与 bash 可用，未检测到 ethtool。 |
| posix-aio | 测试脚本路径为 OK。 | 测试脚本存在。 |
| precheck | 提示执行 `ucm-toolkit run precheck`。 | doctor 显示具体预检命令的使用提示。 |

## 3. precheck 测试

### 3.1 执行命令与结果

| 编号 | 执行目录 | 执行命令 | 实际结果 | 结论 |
| --- | --- | --- | --- | --- |
| PRE-01 | `/mnt` | `ucm-toolkit run precheck --skip-bandwidth` | 输出 3 pass、0 warn、0 fail、0 skip、3 info，结果为 PASSED。 | 通过 |
| PRE-02 | `/mnt` | `ucm-toolkit run precheck --mount-path /mnt/ucm-cache-test` | 执行 12 个带宽组合，其中 8 个完成、4 个 AIO 组合超时；最终为 PASSED (with warnings)。 | 存在 AIO 超时及带宽阈值告警 |
| PRE-03 | `/workspace/storage` | `ucm-toolkit run precheck --only kernel --only accelerator_driver` | 仅输出 accelerator_driver 和 kernel 两项，均为 PASS；结果为 2 pass、0 warn、0 fail、0 skip、0 info。 | 通过 |
| PRE-04 | `/workspace/storage` | `ucm-toolkit run precheck --skip-bandwidth --json` | 输出可解析的 JSON，包含 6 个检查项；failed=false、warned=false。 | 通过 |

### 3.2 环境预检

| 检查项 | 实际结果 | 判定 |
| --- | --- | --- |
| serving_stack | vllm=0.27.1 | INFO |
| uc_manager | uc-manager-cuda-cu130=0.8.0 | INFO |
| accelerator_driver | compute_cap=9.0，阈值为 >=8.0 | PASS |
| kernel | 5.15.0，阈值为 >=5.10 | PASS |
| memory_shm | 总容量 1007.82 GiB，阈值为 >=512 GiB | PASS |
| aio_resources | max-nr=65536、nr=0、available=65536、max_aio_workers=16 | INFO |

环境预检正常完成，驱动算力、内核和共享内存三项均满足工具阈值，没有告警或失败。

JSON 输出还确认了驱动版本为 `580.173.02`，设备为 8 张 `NVIDIA H100 80GB HBM3`，完整内核版本为 `5.15.0-185-generic`。

### 3.3 带宽测试结果

测试路径为 `/mnt/ucm-cache-test`。扫描 180 KiB、8 MiB 两种分片大小，1、8、16 三种 worker 数量，以及 psync、aio 两种引擎，共 12 个组合。

下表摘录工具输出的 aggregate 值，单位为 GB/s，不是单 worker 带宽。

| 分片大小 | workers | 引擎 | dump | load | comprehensive | mixed | 执行结果 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 180 KiB | 1 | psync | 3.653 | 2.087 | 2.870 | 1.740 | 完成 |
| 180 KiB | 1 | aio | 4.071 | 2.160 | 3.115 | 1.898 | 完成 |
| 180 KiB | 8 | psync | 5.968 | 5.609 | 5.788 | 4.877 | 完成 |
| 180 KiB | 8 | aio | — | — | — | — | 120 秒超时 |
| 180 KiB | 16 | psync | 6.263 | 8.225 | 7.244 | 7.618 | 完成 |
| 180 KiB | 16 | aio | — | — | — | — | 120 秒超时 |
| 8 MiB | 1 | psync | 4.490 | 6.728 | 5.609 | 6.187 | 完成 |
| 8 MiB | 1 | aio | 4.784 | 6.729 | 5.756 | 5.677 | 完成 |
| 8 MiB | 8 | psync | 4.755 | 7.121 | 5.938 | 6.650 | 完成 |
| 8 MiB | 8 | aio | — | — | — | — | 120 秒超时 |
| 8 MiB | 16 | psync | 4.774 | 7.160 | 5.967 | 6.849 | 完成 |
| 8 MiB | 16 | aio | — | — | — | — | 120 秒超时 |

#### 3.3.1 多 worker AIO 超时

180 KiB 和 8 MiB 两种分片配置下，aio 引擎在 8、16 workers 时均超时，共 4 个异常组合。原始错误如下：

```text
shard=184320B workers=8 engine=aio ERR: combo timed out after 120s (a worker may be stuck on aio I/O; try psync or raise fs.aio-max-nr)
shard=184320B workers=16 engine=aio ERR: combo timed out after 120s (a worker may be stuck on aio I/O; try psync or raise fs.aio-max-nr)
shard=8388608B workers=8 engine=aio ERR: combo timed out after 120s (a worker may be stuck on aio I/O; try psync or raise fs.aio-max-nr)
shard=8388608B workers=16 engine=aio ERR: combo timed out after 120s (a worker may be stuck on aio I/O; try psync or raise fs.aio-max-nr)
```

同一次测试中，psync 的 6 个组合全部完成，aio 的两个单 worker 组合完成。异常集中在多 worker AIO 场景，记录为带宽测试中的执行异常。错误文字中的 I/O 阻塞或 AIO 资源调整属于工具提示，不作为已确认根因。

#### 3.3.2 带宽阈值告警

工具汇总的四项最佳聚合值均来自 180 KiB、16 workers、psync 组合：

| 指标 | 最佳聚合带宽（GB/s） | 阈值（GB/s） | 判定 |
| --- | --- | --- | --- |
| dump | 6.263 | >=8.0 | WARN |
| load | 8.225 | >=8.0 | PASS |
| comprehensive | 7.244 | >=8.0 | WARN |
| mixed | 7.618 | >=8.0 | WARN |

仅 load 达到阈值，dump、comprehensive、mixed 均低于 8.0 GB/s，因此 bandwidth 检查项为 WARN。

最终输出：

```text
Result: PASSED (with warnings) | 3 pass, 1 warn, 0 fail, 0 skip, 3 info
```

这里的 PASSED (with warnings) 是预检汇总结果，不代表所有带宽组合均执行成功。报告分别记录性能阈值告警与 4 个 AIO 超时，不将本项判为全部通过。

### 3.4 指定检查项

指定 `kernel` 和 `accelerator_driver` 后，输出仅包含这两个检查项，均为 PASS。检查项筛选功能正常。

### 3.5 JSON 输出

输出为合法 JSON，包含 serving_stack、uc_manager、accelerator_driver、kernel、memory_shm、aio_resources 六项，顶层 `failed`、`warned` 均为 false。驱动、内核、共享内存的 `status` 均为 PASS，与文本形式的判定一致。

两次输出中的 AIO 资源快照如下：

| 输出记录 | max-nr | nr | available | max_aio_workers |
| --- | --- | --- | --- | --- |
| 首次跳过带宽的文本检查 | 65536 | 0 | 65536 | 16 |
| 后续跳过带宽的 JSON 检查 | 65536 | 32768 | 32768 | 8 |

后一次检查显示 AIO 已占用事件数增加，可用事件数及估算并发 worker 数随之减少。这是两次检查时的资源状态变化，不能单凭该变化认定为资源泄漏。

## 4. posix-aio 测试

### 4.1 模型驱动、layerwise 模式

执行目录：`/workspace/storage`。

```bash
ucm-toolkit run posix-aio --model /workspace/model/DeepSeek-V32 --tp 8 --input-len 4096 \
  --worker-number 8 --layerwise --storage-backend /mnt
```

实际结果：模型识别为 DSA，输出完整的 UCM Store IO Info，并启动 8 个 worker 执行 dump。日志中 worker 00–07 均输出耗时和带宽统计，测试正常推进。

结论：本次 DSA 模型的 layerwise 模式功能验证通过，参数计算和并发 dump 正常。

### 4.2 IO 参数核对

| 项目 | 实际结果 | 核对 |
| --- | --- | --- |
| 模型 | `/workspace/model/DeepSeek-V32`，DSA | 与本次执行命令一致 |
| 层数 | 61 | 每个文件包含 61 个 shard |
| head_dim | 704 | 512 + 64 + 128 = 704 |
| dtype | bfloat16，2 bytes/elem | 按 2 字节计算 |
| block_size / input_len | 128 / 4096 tokens | block_number = ceil(4096 / 128) = 32 |
| layerwise | True | 与 `--layerwise` 一致 |
| per_layer/block | 180224 bytes | (512 + 64 + 128) × 128 × 2 = 180224 |
| shard size | 180224 bytes | 已为 4096 的整数倍，对齐后不变 |
| file size | 10993664 bytes | 180224 × 61 = 10993664 |
| total data/worker | 351797248 bytes，约 0.328 GiB | 10993664 × 32 = 351797248 |

IO 大小、文件大小及每个 worker 的数据量计算一致。

### 4.3 运行结果

日志片段包含 epoch 000–003 的 dump 记录，数据规模均为 `[180224 x 32 x 61]`。单 worker 带宽为 0.573–0.597 GB/s，输出的带宽与数据量、total_cost 换算结果一致。

典型输出：

```text
epoch=000, worker=06, dump=[180224 x 32 x 61], avg_cost=9.919ms, p99_cost=11.288ms, total_cost=605.065ms, bw=0.581GB/s.
epoch=001, worker=06, dump=[180224 x 32 x 61], avg_cost=9.668ms, p99_cost=12.068ms, total_cost=589.752ms, bw=0.597GB/s.
epoch=002, worker=06, dump=[180224 x 32 x 61], avg_cost=9.661ms, p99_cost=11.623ms, total_cost=589.332ms, bw=0.597GB/s.
epoch=003, worker=06, dump=[180224 x 32 x 61], avg_cost=9.660ms, p99_cost=11.350ms, total_cost=589.269ms, bw=0.597GB/s.
```

上述带宽为每个 worker 的统计值，不是 8 个 worker 的聚合带宽。本次记录未出现参数解析错误或 dump 报错。

## 5. nic-monitor 测试

执行目录：`/mnt`。本次验证前台监控用例。

```bash
sudo ucm-toolkit run nic-monitor fg 5
```

实际结果：

- 成功启动监控界面，列出 18 个网卡接口，显示采样间隔为 5s。
- 正常输出各接口的接收和发送速率。例如 `ens10f0` 的接收速率为 10.40 KB/s、发送速率为 42.27 KB/s。
- 驱动列为空，端口速率显示为 `---`，利用率显示为 `N/A`。
- 按 Ctrl+C 后输出“监控已停止。”，正常结束。

结论：前台监控的启动、收发速率展示和手动停止功能通过；端口速率及利用率本次显示为不可用。

## 6. metrics-view 测试

### 6.1 时间窗口查询

执行目录：`/workspace`。

```bash
ucm-toolkit run metrics-view query \
  --window 10m \
  --aggr-by 1m \
  --config metrics_lite \
  --config-param tp_size=8
```

预期结果：使用 `metrics_lite` 配置，查询最近 10 分钟的指标，每 1 分钟聚合一次，并应用 `tp_size=8` 参数。

实际结果：命令抛出 Traceback，未输出指标查询结果。关键错误为：

```text
ValueError: Config parameter not found: tp_size
```

结论：不通过，按文档示例传入 `tp_size` 后查询失败。

### 6.2 问题记录：tp_size 参数与配置不一致

| 项目 | 记录 |
| --- | --- |
| 问题编号 | MET-01 |
| 触发条件 | `--config metrics_lite --config-param tp_size=8` |
| 异常位置 | `terminal_view_metrics/config.py:47`，`apply_config_param_overrides` |
| 问题现象 | 应用配置参数时抛出 ValueError，提示找不到 tp_size |
| 影响 | 本次时间窗口查询中断，无法输出指标结果 |
| 判断 | 文档示例使用的 tp_size 参数未被当前 metrics_lite 配置识别，示例与当前安装版本的配置不一致 |

从异常栈可见，程序已进入 query 的配置处理流程，错误发生在应用配置参数时，不是查询返回空数据。

## 7. 检查结论

Toolkit 安装、帮助信息和工具列表验证通过。`doctor` 正常输出检查结果，记录到 dev-sandbox 构建目录及二进制缺失、nic-monitor 未检测到 ethtool。

precheck 的环境预检、指定检查项和不含带宽的 JSON 输出验证通过。带宽测试存在两类问题：

- 执行异常：4 个多 worker AIO 组合在 120 秒后超时。
- 性能告警：最佳聚合 dump、comprehensive、mixed 带宽低于 8.0 GB/s 阈值，load 达标。

带宽测试最终显示 PASSED (with warnings)，本报告保留上述异常与告警，不将其归为全部通过。

posix-aio 的 DeepSeek-V32（DSA）模型 layerwise 场景功能验证通过，IO 参数计算正确，8 个 worker 正常输出 dump 耗时和带宽统计。

nic-monitor 前台监控可正常显示 18 个接口的收发速率，并响应 Ctrl+C 停止；驱动列为空，端口速率和利用率显示为不可用。

metrics-view 时间窗口查询不通过：传入文档示例中的 `tp_size=8` 后，程序在配置处理阶段报 `Config parameter not found: tp_size`，未输出查询结果。

## 附件：命令输出

### A.1 安装 Toolkit

命令：

```bash
python3 -m pip install ./ucm_toolkit-0.8.0rc1-py3-none-any.whl
```

输出：

```text
Processing ./ucm_toolkit-0.8.0rc1-py3-none-any.whl
Installing collected packages: ucm-toolkit
Successfully installed ucm-toolkit-0.8.0rc1
WARNING: Running pip as the 'root' user can result in broken permissions and conflicting behaviour with the system package manager, possibly rendering your system unusable. It is recommended to use a virtual environment instead: https://pip.pypa.io/warnings/venv. Use the --root-user-action option if you know what you are doing and want to suppress this warning.
```

### A.2 查看帮助

命令：

```bash
ucm-toolkit --help
```

输出：

```text
usage: ucm-toolkit [-h] {list,doctor,build,clean,run} ...

positional arguments:
  {list,doctor,build,clean,run}
    list                List top-level toolkit tools
    doctor              Inspect toolkit tool setup
    build               Build a toolkit tool
    clean               Clean a toolkit tool
    run                 Run a toolkit tool

options:
  -h, --help            show this help message and exit
```

### A.3 查看工具列表

命令：

```bash
ucm-toolkit list
```

输出：

```text
dev-sandbox    buildable Build the CMake-based dev-sandbox test project.
metrics-view   runnable  Collect and query Prometheus metrics from a terminal.
nic-monitor    runnable  Run passive NIC load monitoring.
posix-aio      runnable  Run the POSIX AIO store test script.
precheck       runnable  Run UCM environment pre-checks before deploying UCM: serving-stack and uc-manager versions, accelerator driver (CUDA compute capability or Ascend HDK), kernel version, memory & /dev/shm size, and the posix-store bandwidth benchmark. Reports PASS/WARN/FAIL per item with remediation advice for failures.
```

### A.4 检查运行环境

命令：

```bash
ucm-toolkit doctor
```

输出：

```text
dev-sandbox:
  source_dir: /usr/local/lib/python3.12/dist-packages/ucm_toolkit/_resources/dev-sandbox OK
  build_dir:  /root/.cache/ucm-toolkit/479681a718dee7d0/0.8.0rc1/dev-sandbox-build MISSING
  copy : /root/.cache/ucm-toolkit/479681a718dee7d0/0.8.0rc1/dev-sandbox-build/module/copy/copy MISSING
  trans: /root/.cache/ucm-toolkit/479681a718dee7d0/0.8.0rc1/dev-sandbox-build/module/trans/trans MISSING
  aio  : /root/.cache/ucm-toolkit/479681a718dee7d0/0.8.0rc1/dev-sandbox-build/module/aio/aio MISSING
metrics-view: no environment checks
nic-monitor:
  script:  /usr/local/lib/python3.12/dist-packages/ucm_toolkit/_resources/nic_monitor_pro.sh OK
  bash:    OK
  ethtool: MISSING
  note:    the monitor script must be run as root or with sudo
posix-aio: /usr/local/lib/python3.12/dist-packages/ucm_toolkit/_resources/posixstore_aio_test.py OK
precheck: run `ucm-toolkit run precheck` to check the env
```

### A.5 precheck：跳过带宽的环境预检

执行目录：`/mnt`

```bash
ucm-toolkit run precheck --skip-bandwidth
```

输出（按字段恢复换行）：

```text
UCM environment pre-check 
──────────────────────────────────────────────────────────────────────────────

[INFO] serving_stack vllm=0.27.1
[INFO] uc_manager uc-manager-cuda-cu130=0.8.0
[PASS] accelerator_driver compute_cap=9.0
  threshold: compute_cap >= 8.0
[PASS] kernel 5.15.0
  threshold: >= 5.10
[PASS] memory_shm shm=1007.82 GiB (1007.56 GiB free)
  threshold: >= 512 GiB
[INFO] aio_resources max-nr=65536 nr=0 available=65536 max_aio_workers=16 
──────────────────────────────────────────────────────────────────────────────
 Result: PASSED | 3 pass, 0 warn, 0 fail, 0 skip, 3 info
```

### A.6 precheck：包含带宽的预检

执行目录：`/mnt`

```bash
ucm-toolkit run precheck --mount-path /mnt/ucm-cache-test
```

输出（按字段恢复换行）：

```text
[bw] probing aio...
[bw] shard=180KiB workers=1 engine=psync ...
[bw] shard=180KiB workers=1 engine=aio ...
[bw] shard=180KiB workers=8 engine=psync ...
[bw] shard=180KiB workers=8 engine=aio ...
[bw] shard=180KiB workers=16 engine=psync ...
[bw] shard=180KiB workers=16 engine=aio ...
[bw] shard=8MiB workers=1 engine=psync ...
[bw] shard=8MiB workers=1 engine=aio ...
[bw] shard=8MiB workers=8 engine=psync ...
[bw] shard=8MiB workers=8 engine=aio ...
[bw] shard=8MiB workers=16 engine=psync ...
[bw] shard=8MiB workers=16 engine=aio ...

bandwidth matrix:

180KiB workers=1 engine=psync (n=8 samples/direction, per-worker below)
  dump avg=3.653 std=0.296 min=2.882 max=3.798 GB/s
  load avg=2.087 std=0.524 min=0.967 max=2.648 GB/s
  comprehensive = 2.870 GB/s
  mixed avg=1.740 std=0.140 min=1.604 max=2.067 GB/s (read-heavy 1:4)
  aggregate (1w): dump=3.653 load=2.087 comp=2.870 mixed=1.740 GB/s

180KiB workers=1 engine=aio (n=8 samples/direction, per-worker below)
  dump avg=4.071 std=0.961 min=1.533 max=4.503 GB/s
  load avg=2.160 std=0.537 min=1.050 max=2.785 GB/s
  comprehensive = 3.115 GB/s
  mixed avg=1.898 std=0.135 min=1.708 max=2.104 GB/s (read-heavy 1:4)
  aggregate (1w): dump=4.071 load=2.160 comp=3.115 mixed=1.898 GB/s

180KiB workers=8 engine=psync (n=64 samples/direction, per-worker below)
  dump avg=0.746 std=0.364 min=0.579 max=3.281 GB/s
  load avg=0.701 std=0.281 min=0.244 max=2.180 GB/s
  comprehensive = 0.724 GB/s
  mixed avg=0.610 std=0.056 min=0.530 max=0.784 GB/s (read-heavy 1:4)
  aggregate (8w): dump=5.968 load=5.609 comp=5.788 mixed=4.877 GB/s

shard=184320B workers=8 engine=aio ERR: combo timed out after 120s (a worker may be stuck on aio I/O; try psync or raise fs.aio-max-nr)

180KiB workers=16 engine=psync (n=128 samples/direction, per-worker below)
  dump avg=0.391 std=0.306 min=0.159 max=2.180 GB/s
  load avg=0.514 std=0.216 min=0.350 max=1.987 GB/s
  comprehensive = 0.453 GB/s
  mixed avg=0.476 std=0.134 min=0.379 max=1.187 GB/s (read-heavy 1:4)
  aggregate (16w): dump=6.263 load=8.225 comp=7.244 mixed=7.618 GB/s

shard=184320B workers=16 engine=aio ERR: combo timed out after 120s (a worker may be stuck on aio I/O; try psync or raise fs.aio-max-nr)

8MiB workers=1 engine=psync (n=8 samples/direction, per-worker below)
  dump avg=4.490 std=0.130 min=4.386 max=4.746 GB/s
  load avg=6.728 std=0.042 min=6.666 max=6.794 GB/s
  comprehensive = 5.609 GB/s
  mixed avg=6.187 std=0.064 min=6.067 max=6.277 GB/s (read-heavy 1:4)
  aggregate (1w): dump=4.490 load=6.728 comp=5.609 mixed=6.187 GB/s

8MiB workers=1 engine=aio (n=8 samples/direction, per-worker below)
  dump avg=4.784 std=0.102 min=4.544 max=4.920 GB/s
  load avg=6.729 std=0.023 min=6.698 max=6.765 GB/s
  comprehensive = 5.756 GB/s
  mixed avg=5.677 std=0.628 min=4.582 max=6.251 GB/s (read-heavy 1:4)
  aggregate (1w): dump=4.784 load=6.729 comp=5.756 mixed=5.677 GB/s

8MiB workers=8 engine=psync (n=64 samples/direction, per-worker below)
  dump avg=0.594 std=0.017 min=0.566 max=0.657 GB/s
  load avg=0.890 std=0.012 min=0.881 max=0.953 GB/s
  comprehensive = 0.742 GB/s
  mixed avg=0.831 std=0.048 min=0.798 max=1.052 GB/s (read-heavy 1:4)
  aggregate (8w): dump=4.755 load=7.121 comp=5.938 mixed=6.650 GB/s

shard=8388608B workers=8 engine=aio ERR: combo timed out after 120s (a worker may be stuck on aio I/O; try psync or raise fs.aio-max-nr)

8MiB workers=16 engine=psync (n=128 samples/direction, per-worker below)
  dump avg=0.298 std=0.035 min=0.289 max=0.690 GB/s
  load avg=0.448 std=0.010 min=0.442 max=0.527 GB/s
  comprehensive = 0.373 GB/s
  mixed avg=0.428 std=0.040 min=0.404 max=0.709 GB/s (read-heavy 1:4)
  aggregate (16w): dump=4.774 load=7.160 comp=5.967 mixed=6.849 GB/s

shard=8388608B workers=16 engine=aio ERR: combo timed out after 120s (a worker may be stuck on aio I/O; try psync or raise fs.aio-max-nr)

UCM environment pre-check 
──────────────────────────────────────────────────────────────────────────────

[INFO] serving_stack vllm=0.27.1
[INFO] uc_manager uc-manager-cuda-cu130=0.8.0
[PASS] accelerator_driver compute_cap=9.0
  threshold: compute_cap >= 8.0
[PASS] kernel 5.15.0
  threshold: >= 5.10
[PASS] memory_shm shm=1007.82 GiB (1007.56 GiB free)
  threshold: >= 512 GiB
[INFO] aio_resources max-nr=65536 nr=0 available=65536 max_aio_workers=16
[WARN] bandwidth dump=6.263 load=8.225 comprehensive=7.244 mixed=7.618 GB/s
  threshold: >= 8.0 GB/s
  dump best=6.263 GB/s (shard=184320B workers=16 engine=psync)
[WARN]
  load best=8.225 GB/s (shard=184320B workers=16 engine=psync)
[PASS]
  comprehensive best=7.244 GB/s (shard=184320B workers=16 engine=psync)
[WARN]
  mixed best=7.618 GB/s (shard=184320B workers=16 engine=psync)
[WARN]
  fix: metrics below threshold (dump, comprehensive, mixed): check the mount-point disk/NVMe throughput; try the aio engine, raise --workers, or confirm io_direct/O_DIRECT settings 
──────────────────────────────────────────────────────────────────────────────
 Result: PASSED (with warnings) | 3 pass, 1 warn, 0 fail, 0 skip, 3 info
```

### A.7 precheck：指定检查项

执行目录：`/workspace/storage`

```bash
ucm-toolkit run precheck --only kernel --only accelerator_driver
```

输出（按字段恢复换行）：

```text
UCM environment pre-check 
──────────────────────────────────────────────────────────────────────────────

[PASS] accelerator_driver compute_cap=9.0
  threshold: compute_cap >= 8.0
[PASS] kernel 5.15.0
  threshold: >= 5.10 
──────────────────────────────────────────────────────────────────────────────
 Result: PASSED | 2 pass, 0 warn, 0 fail, 0 skip, 0 info
```

### A.8 precheck：JSON 输出

执行目录：`/workspace/storage`

```bash
ucm-toolkit run precheck --skip-bandwidth --json
```

输出（按字段恢复换行，JSON 仅调整缩进）：

```json
{
  "checks": [
    {
      "name": "serving_stack",
      "severity": "INFO",
      "status": "INFO",
      "value": "vllm=0.27.1",
      "threshold": "",
      "detail": "installed serving stack",
      "remediation": "",
      "raw": {
        "vllm": "0.27.1"
      }
    },
    {
      "name": "uc_manager",
      "severity": "INFO",
      "status": "INFO",
      "value": "uc-manager-cuda-cu130=0.8.0",
      "threshold": "",
      "detail": "installed UCM packages",
      "remediation": "",
      "raw": {
        "distributions": {
          "uc-manager-cuda-cu130": "0.8.0"
        },
        "version": "0.8.0"
      }
    },
    {
      "name": "accelerator_driver",
      "severity": "WARN",
      "status": "PASS",
      "value": "compute_cap=9.0",
      "threshold": "compute_cap >= 8.0",
      "detail": "driver=580.173.02, 8 GPU(s): NVIDIA H100 80GB HBM3, NVIDIA H100 80GB HBM3, NVIDIA H100 80GB HBM3, NVIDIA H100 80GB HBM3, NVIDIA H100 80GB HBM3, NVIDIA H100 80GB HBM3, NVIDIA H100 80GB HBM3, NVIDIA H100 80GB HBM3",
      "remediation": "",
      "raw": {
        "platform": "nvidia",
        "min_cap": 9,
        "rows": [
          [
            "NVIDIA H100 80GB HBM3",
            "580.173.02",
            "9.0"
          ],
          [
            "NVIDIA H100 80GB HBM3",
            "580.173.02",
            "9.0"
          ],
          [
            "NVIDIA H100 80GB HBM3",
            "580.173.02",
            "9.0"
          ],
          [
            "NVIDIA H100 80GB HBM3",
            "580.173.02",
            "9.0"
          ],
          [
            "NVIDIA H100 80GB HBM3",
            "580.173.02",
            "9.0"
          ],
          [
            "NVIDIA H100 80GB HBM3",
            "580.173.02",
            "9.0"
          ],
          [
            "NVIDIA H100 80GB HBM3",
            "580.173.02",
            "9.0"
          ],
          [
            "NVIDIA H100 80GB HBM3",
            "580.173.02",
            "9.0"
          ]
        ]
      }
    },
    {
      "name": "kernel",
      "severity": "FAIL",
      "status": "PASS",
      "value": "5.15.0",
      "threshold": ">= 5.10",
      "detail": "kernel 5.15.0-185-generic (uname -r)",
      "remediation": "",
      "raw": {
        "release": "5.15.0-185-generic"
      }
    },
    {
      "name": "memory_shm",
      "severity": "WARN",
      "status": "PASS",
      "value": "shm=1007.82 GiB (1007.80 GiB free)",
      "threshold": ">= 512 GiB",
      "detail": "/dev/shm size OK",
      "remediation": "",
      "raw": {
        "shm_total_bytes": 1082136977408,
        "shm_avail_bytes": 1082122170368
      }
    },
    {
      "name": "aio_resources",
      "severity": "INFO",
      "status": "INFO",
      "value": "max-nr=65536 nr=32768 available=32768 max_aio_workers=8",
      "threshold": "",
      "detail": "each aio context needs 4096 events (queueDepth); 32768 available → 8 concurrent aio workers",
      "remediation": "",
      "raw": {
        "aio_max_nr": 65536,
        "aio_nr": 32768,
        "available": 32768,
        "aio_queue_depth": 4096,
        "max_aio_workers": 8
      }
    }
  ],
  "failed": false,
  "warned": false
}
```

原始粘贴记录：[precheck 完整输入输出](toolkit-self-test-precheck-output.txt)。

### A.9 posix-aio：DSA 模型 layerwise 测试

执行目录：`/workspace/storage`

```bash
ucm-toolkit run posix-aio --model /workspace/model/DeepSeek-V32 --tp 8 --input-len 4096 --worker-number 8 --layerwise --storage-backend /mnt
```

输出（按粘贴内容恢复换行）：

```text
UCM Store IO Info:
  model             : /workspace/model/DeepSeek-V32
  architecture      : DSA
  num_hidden_layers : 61
  head_dim          : 704
  dtype             : bfloat16 (2 bytes/elem)
  block_size(tokens): 128
  input_len         : 4096
  layerwise         : True
  per_layer/block   : 180224 bytes (176.00 KB)  |  (kv_lora_rank(512)+qk_rope_head_dim(64)+index_head_dim(128)) * block_size(128) * dtype(2)
  shard size        : 180224 bytes (176.00 KB)  |  align_up(per_layer/block, 4096)
  shards per file   : 61
  file size         : 10993664 bytes (10736.00 KB / 10.48 MB)  |  = shard size * shards per file
  block_number      : 32 (= ceil(4096/128))
  total data/worker : 351797248 bytes (~0.328 GiB)
epoch=000, worker=06, dump=[180224 x 32 x 61], avg_cost=9.919ms, p99_cost=11.288ms, total_cost=605.065ms, bw=0.581GB/s.
epoch=000, worker=02, dump=[180224 x 32 x 61], avg_cost=9.943ms, p99_cost=11.306ms, total_cost=606.494ms, bw=0.580GB/s.
epoch=000, worker=04, dump=[180224 x 32 x 61], avg_cost=9.965ms, p99_cost=11.132ms, total_cost=607.871ms, bw=0.579GB/s.
epoch=000, worker=07, dump=[180224 x 32 x 61], avg_cost=9.982ms, p99_cost=11.004ms, total_cost=608.877ms, bw=0.578GB/s.
epoch=000, worker=00, dump=[180224 x 32 x 61], avg_cost=9.998ms, p99_cost=11.049ms, total_cost=609.907ms, bw=0.577GB/s.
epoch=000, worker=05, dump=[180224 x 32 x 61], avg_cost=10.018ms, p99_cost=11.086ms, total_cost=611.125ms, bw=0.576GB/s.
epoch=000, worker=03, dump=[180224 x 32 x 61], avg_cost=10.040ms, p99_cost=11.083ms, total_cost=612.433ms, bw=0.574GB/s.
epoch=000, worker=01, dump=[180224 x 32 x 61], avg_cost=10.061ms, p99_cost=11.406ms, total_cost=613.738ms, bw=0.573GB/s.
epoch=001, worker=06, dump=[180224 x 32 x 61], avg_cost=9.668ms, p99_cost=12.068ms, total_cost=589.752ms, bw=0.597GB/s.
epoch=001, worker=01, dump=[180224 x 32 x 61], avg_cost=9.686ms, p99_cost=12.048ms, total_cost=590.866ms, bw=0.595GB/s.
epoch=001, worker=00, dump=[180224 x 32 x 61], avg_cost=9.706ms, p99_cost=11.728ms, total_cost=592.037ms, bw=0.594GB/s.
epoch=001, worker=07, dump=[180224 x 32 x 61], avg_cost=9.725ms, p99_cost=12.187ms, total_cost=593.233ms, bw=0.593GB/s.
epoch=001, worker=05, dump=[180224 x 32 x 61], avg_cost=9.743ms, p99_cost=12.008ms, total_cost=594.302ms, bw=0.592GB/s.
epoch=001, worker=03, dump=[180224 x 32 x 61], avg_cost=9.761ms, p99_cost=12.217ms, total_cost=595.422ms, bw=0.591GB/s.
epoch=001, worker=04, dump=[180224 x 32 x 61], avg_cost=9.784ms, p99_cost=12.418ms, total_cost=596.817ms, bw=0.589GB/s.
epoch=001, worker=02, dump=[180224 x 32 x 61], avg_cost=9.804ms, p99_cost=12.299ms, total_cost=598.066ms, bw=0.588GB/s.
epoch=002, worker=06, dump=[180224 x 32 x 61], avg_cost=9.661ms, p99_cost=11.623ms, total_cost=589.332ms, bw=0.597GB/s.
epoch=002, worker=02, dump=[180224 x 32 x 61], avg_cost=9.681ms, p99_cost=11.501ms, total_cost=590.531ms, bw=0.596GB/s.
epoch=002, worker=00, dump=[180224 x 32 x 61], avg_cost=9.700ms, p99_cost=11.371ms, total_cost=591.700ms, bw=0.595GB/s.
epoch=002, worker=07, dump=[180224 x 32 x 61], avg_cost=9.720ms, p99_cost=11.242ms, total_cost=592.936ms, bw=0.593GB/s.
epoch=002, worker=01, dump=[180224 x 32 x 61], avg_cost=9.739ms, p99_cost=11.117ms, total_cost=594.087ms, bw=0.592GB/s.
epoch=002, worker=03, dump=[180224 x 32 x 61], avg_cost=9.760ms, p99_cost=11.387ms, total_cost=595.372ms, bw=0.591GB/s.
epoch=002, worker=05, dump=[180224 x 32 x 61], avg_cost=9.779ms, p99_cost=11.684ms, total_cost=596.529ms, bw=0.590GB/s.
epoch=002, worker=04, dump=[180224 x 32 x 61], avg_cost=9.796ms, p99_cost=11.712ms, total_cost=597.529ms, bw=0.589GB/s.
epoch=003, worker=06, dump=[180224 x 32 x 61], avg_cost=9.660ms, p99_cost=11.350ms, total_cost=589.269ms, bw=0.597GB/s.
epoch=003, worker=04, dump=[180224 x 32 x 61], avg_cost=9.680ms, p99_cost=11.729ms, total_cost=590.456ms, bw=0.596GB/s.
epoch=003, worker=00, dump=[180224 x 32 x 61], avg_cost=9.696ms, p99_cost=11.860ms, total_cost=591.460ms, bw=0.595GB/s.
```

### A.10 nic-monitor：前台监控

执行目录：`/mnt`

```bash
sudo ucm-toolkit run nic-monitor fg 5
```

输出（按粘贴内容恢复换行）：

```text
=================== 物理网卡实时性能监控 ===================
时间         网卡       驱动     接收速率    发送速率    端口速率 利用率
-------------------------------------------------------------------------
07:37:30       enp25s0f0np0            195.0 B/s       0.0 B/s         ---        N/A
07:37:30       enp25s0f1np1            75.6 B/s        0.0 B/s         ---        N/A
07:37:30       enp57s0f0np0            194.8 B/s       0.0 B/s         ---        N/A
07:37:30       enp57s0f1np1            64.8 B/s        0.0 B/s         ---        N/A
07:37:30       enp73s0f0np0            205.8 B/s       0.0 B/s         ---        N/A
07:37:30       enp73s0f1np1            64.8 B/s        0.0 B/s         ---        N/A
07:37:30       enp90s0f0np0            205.8 B/s       0.0 B/s         ---        N/A
07:37:30       enp90s0f1np1            86.3 B/s        0.0 B/s         ---        N/A
07:37:30       ens5f0np0               0.0 B/s         0.0 B/s         ---        N/A
07:37:30       ens5f1np1               86.4 B/s        0.0 B/s         ---        N/A
07:37:30       ens6f0np0               270.7 B/s       97.1 B/s        ---        N/A
07:37:30       ens6f1np1               86.5 B/s        0.0 B/s         ---        N/A
07:37:30       ens7f0np0               0.0 B/s         0.0 B/s         ---        N/A
07:37:30       ens7f1np1               86.5 B/s        0.0 B/s         ---        N/A
07:37:30       ens8f0np0               206.2 B/s       0.0 B/s         ---        N/A
07:37:30       ens8f1np1               86.6 B/s        0.0 B/s         ---        N/A
07:37:30       ens10f0                 10.40 KB/s      42.27 KB/s      ---        N/A
07:37:30       ens10f1                 81.9 B/s        7.6 B/s         ---        N/A
============================================================
按 Ctrl+C 退出 | 采样间隔: 5s | 更新于: 07:37:30
^C
监控已停止。
```

### A.11 metrics-view：时间窗口查询失败

执行目录：`/workspace`

```bash
ucm-toolkit run metrics-view query \
  --window 10m \
  --aggr-by 1m \
  --config metrics_lite \
  --config-param tp_size=8
```

输出（按粘贴内容恢复换行）：

```text
Traceback (most recent call last):
  File "/usr/local/bin/ucm-toolkit", line 6, in <module>
    sys.exit(main())
             ^^^^^^
  File "/usr/local/lib/python3.12/dist-packages/ucm_toolkit/cli.py", line 38, in main
    return run_cmd.handle(argv[1:])
           ^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/dist-packages/ucm_toolkit/commands/run.py", line 24, in handle
    return tool.run(tool_args)
           ^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/dist-packages/ucm_toolkit/tools/metrics_view/adapter.py", line 29, in run
    return int(metrics_main(tool_args))
               ^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/dist-packages/ucm_toolkit/tools/metrics_view/terminal_view_metrics/cli.py", line 41, in main
    return int(args.func(args))
               ^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/dist-packages/ucm_toolkit/tools/metrics_view/terminal_view_metrics/cli.py", line 271, in _cmd_query
    rows = _query_rows(args)
           ^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/dist-packages/ucm_toolkit/tools/metrics_view/terminal_view_metrics/cli.py", line 296, in _query_rows
    config = apply_config_param_overrides(
             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/dist-packages/ucm_toolkit/tools/metrics_view/terminal_view_metrics/config.py", line 47, in apply_config_param_overrides
    raise ValueError(f"Config parameter not found: {key}")
ValueError: Config parameter not found: tp_size
```
