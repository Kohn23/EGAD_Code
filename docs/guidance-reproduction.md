下面是从「刚 clone 完」到「复现全部结果」的完整流水线。每一步都标注了**命令、产生什么、落在哪里**。核心原则是:**先装依赖 → 构建四种二进制 → preflight 校准机器 → 跑测试(产 log)→ 渲染(产图)→ 单独跑恢复验证**。

---

## 第 0 步:前置条件(无法跳过的硬件/软件)

| 要求 | 说明 |
|---|---|
| CUDA GPU | 32 GB 显存(编译架构 80/86/89),仅用 GPU 0 |
| 主机内存 | ~128 GB(参考机 384 GB) |
| 系统/工具链 | Ubuntu 24.04、Linux 6.8、gcc 13、CUDA 12.9、CMake 3.28、Python3 + matplotlib |
| 已知 GPU→NUMA 映射 | GPU 0 挂在 NUMA node 0,harness 会把 EGAD 进程 pin 到该 node |
| passwordless sudo | 每个 cell 之间要 drop page cache(`echo 3 > /proc/sys/vm/drop_caches`) |

---

## 第 1 步:克隆(含子模块)

```bash
git clone --recurse-submodules https://github.com/ChrisJTab/EGAD_Code.git
cd EGAD_Code
```

**产生:** 完整仓库树,包括 `third_party`(cuco、thrust 等子模块)、`EGAD`(系统本体)、`EGAD_plotting`(图形 harness)、`validation`、`patches`、`setup_epic_stock.sh` 等。

> ⚠️ 必须带 `--recurse-submodules`,否则第三方依赖(尤其 cuco)是空的,构建会失败。

---

## 第 2 步:安装系统依赖(`install_dependencies.sh`,需 sudo,结尾会重启)

```bash
sudo ./install_dependencies.sh
```

**产生/改动:**

| 位置 | 内容 |
|---|---|
| 系统 | apt 更新 + NVIDIA 驱动 575 + CUDA Toolkit 12.9 |
| 系统 | 内核头文件、cmake、build-essential、libnuma-dev、libjemalloc-dev、ninja、boost、libc++ 等 |
| `~/.profile` | 追加 `PATH=/usr/local/cuda/bin` 和 `LD_LIBRARY_PATH=/usr/local/cuda/lib64` |
| `/etc/security/limits.conf` | `* - memlock unlimited` |
| `/etc/sysctl.conf` | `vm.nr_hugepages = 50000`(锁定 100 GB 大页!) |
| Python | 安装 numpy、pandas、matplotlib |

**结尾会自动重启。** 注意:**这一步故意启用了 50000 个大页(100 GB)**,这是 Epic 原始设计所需;但 EGAD 的 hybrid 模式要求大页释放,所以重启后大页又会被保留——这正是第 5 步 preflight 要处理的对象。

---

## 第 3 步(可选,但论文级复现建议做):搭建上游 stock Epic

```bash
./setup_epic_stock.sh                          # 1 KB 版本 → ./epic_stock
./setup_epic_stock.sh --small-records ./epic_stock_120b   # 120 B 版本 → ./epic_stock_120b
```

**产生:** `./epic_stock/`(和 120 B 变体)——这是**未修改的上游 Epic**(`ShujianQian/epic-eval` @ `5a7dc90` = EGAD fork 的基准提交),打了两个仅用于本代硬件构建的补丁(CC 8.9 架构 + cuco Thrust guard),120 B 版本额外打 `epic_stock_120b.patch`。

> 说明:这是 `PLOTS.md` 里「CPU baseline provenance」所指的**上游二进制来源**。`run_all.sh` 驱动的 `tests/*.sh` 实际用的是 fork 自带的 OFF-layout 二进制(`build-off`/`build-small-off`),它已实测与上游吞吐一致(0.99x–1.00x)。所以**这一步不是 `run_all.sh` 的必要前提**,但若你要复现「用纯上游 Epic 跑出的 citable CPU 数字」,就需要它。

---

## 第 4 步:构建四种二进制(`build_binaries.sh`)

**必须在仓库根目录、通过包装脚本构建,不要手跑 cmake**(脚本每次显式设置两个 CMake flag,防止缓存串配置)。

```bash
./build_binaries.sh --hybrid                    # ON, 1 KB
./build_binaries.sh --hybrid --small-records    # ON, 120 B
./build_binaries.sh                             # OFF, 1 KB
./build_binaries.sh --small-records             # OFF, 120 B
```

**产生:** 四个互相隔离的构建树,各有一个 `epic_driver` 可执行文件:

| 命令 | 产物路径 | 布局 | 用途 |
|---|---|---|---|
| `--hybrid` | `EGAD/build/epic_driver` | ON, 1 KB | hybrid_staging(全部 TPC-C + 1 KB YCSB) |
| `--hybrid --small-records` | `EGAD/build-small/epic_driver` | ON, 120 B | 120 B YCSB hybrid |
| (无参数) | `EGAD/build-off/epic_driver` | OFF(stock) | TPC-C cpu_only + gpu_only 基线 |
| `--small-records` | `EGAD/build-small-off/epic_driver` | OFF, 120 B | 120 B YCSB 基线 |

**为什么分四份:** record layout 是编译期 flag;cpu_only 在 ON 布局上值错误(64-bit 原子假设两个版本 tag 相邻,ON 把它们拉开 ~128 B),gpu_only 在 ON 布局上 padding 翻倍小记录导致虚假 OOM 上限。所以 cpu/gpu 基线必须用 OFF,hybrid 必须用 ON。

---

## 第 5 步:Box preflight(跑任何 benchmark 之前必须做)

每次重启后都要重做。`run_all.sh` 会**强制前两项**,不满足就拒绝启动。

```bash
./deallocate_hugepages.sh    # 把 HugePages_Total 归零
# CPU governor 设为 performance(48 个逻辑核全部):
for c in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do echo performance | sudo tee "$c" >/dev/null; done
# THP defrag 设为 defer:
echo defer | sudo tee /sys/kernel/mm/transparent_hugepage/defrag
```

**检查清单(全部满足才开跑):**

| 检查 | 要求 | 为什么 |
|---|---|---|
| Hugepages | `HugePages_Total: 0` | 重启会重新套用 sysctl.conf 的 50000 大页(100 GB,node 0 少 34 GB),`membind=0` 的 hybrid 会随 epoch 深度衰减——曾悄悄毁掉三周数据 |
| Governor | 全部 CPU = `performance` | 重启重置为 schedutil,120 B hybrid 损失 ~19% 吞吐 |
| GPU 0 空闲 | 无 compute 应用、显存 < 100 MiB | 残留显存会撑大 footprint、饿死 hybrid autosizer |
| THP enabled | `THP_enabled: 1` | 容器/tuned 会设 `PR_SET_THP_DISABLE`,子进程继承后 madvise 静默失效,TPC-C 大页不生成(-6~12%) |
| THP defrag | `defer` | `madvise` 默认值在碎片化后触发同步 direct compaction,deep-run 会 8.3→2.9 MTxn/s 断崖 |
| 静默 + drop cache | 每个 cell 前 `drop_caches` | pageable Primary Store 首次触达一致性 |

---

## 第 6 步:跑图形 harness(`run_all.sh`)—— 这是「复现所有论文图」的主入口

```bash
cd EGAD_plotting
./run_all.sh
```

**执行顺序:** preflight(第 5 步的前两项)→ 按字典序跑 `tests/*.sh`(15 个)→ 按字典序跑 `plots/*.py|.sh`(15 个)。

**产生(都在 `EGAD_plotting` 下):**

| 目录 | 内容 |
|---|---|
| `logs/<test_name>/...` | 每个 cell 一个日志(如 `record_size/120B_hybrid_staging_skew0.2_rep1.log`)。每个 cell **skip-if-log-exists**,中断后自动续跑;`FORCE_RERUN=1` 强制重跑 |
| `figures/<NN_name>.png` | 渲染出的图 |
| `figures/<NN_name>.csv` | 每 cell 中位数 CSV,供不改跑数据重新排版 |

**各测试产什么(来自 `RUNNING_TESTS.md` §4):**

| 测试 | cell 数 | 产出 |
|---|---|---|
| 01 tpcc_overlap | 1 | deck W=128 e50 hybrid 日志(overlap 时间线) |
| 02 ycsb_overlap | 1 | YCSB-F 120B rF-fT e300 hybrid 日志 |
| 03 record_size | 72 | 120B vs 1KB × hybrid/cpu_only × 6 skew |
| 04 three_way | 144 | 5M 记录(装入 HBM)× hybrid/gpu_only × 4 workload |
| 05 beyond_hbm | 144 | 20M @ 33.6% × hybrid/cpu_only × 4 workload |
| 06 config_sensitivity | 108 | rT-fF/rT-fT/rF-fT × hybrid/cpu_only |
| 07 cache_sensitivity | 24 | cache 比例 5–100% 扫描 + gpu/cpu 参考线 |
| 08 writeback_breakdown | 24 | async vs sync 逐阶段分解(YCSB) |
| 09 tpcc_warehouse_sweep | 153 | 头号 TPC-C 图;deck W{4..128}、NP W{4..240} × 3 模式 |
| 10 tpcc_writeback_breakdown | 6 | TPC-C async/sync 分解,deck W=128 |
| 11 tpcc_cliff | 7 | 深跑断崖演示,e=400,reserve 6 GB |
| 12–14 | 0(仅绘图) | 复用 03/04/05/08/10 的日志画 warmup / 组合扫描 / 组合分解 |
| 15 recovery_cost | 54 | 恢复开/关/cpu 成本,durable store 在 `/dev/shm/egad_rec_cost` |

**注意:** 12–14 是 plot-only,需先跑完 03/04/05/08/10。完整 campaign 在参考机上要跑**好几天**(仅 test 09 就 153 个 cell)。

---

## 第 7 步:恢复正确性验证(`validation`,独立于图形 harness)

需要带 `EGAD_VALIDATION` 编译开关的专用二进制(发布版二进制不含这些 hook):

```bash
cmake -S EGAD -B EGAD/build-val -DCMAKE_BUILD_TYPE=Release \
      -DHYBRID_RECORD_LAYOUT=ON -DEGAD_VALIDATION=ON
cmake --build EGAD/build-val -j

BIN=EGAD/build-val/epic_driver bash validation/crash_recover_sweep.sh
BIN=EGAD/build-val/epic_driver bash validation/ycsb_crash_recover_sweep.sh
BIN=EGAD/build-val/epic_driver bash validation/negative_control.sh
```

**产生:**

| 步骤 | 产物 |
|---|---|
| cmake 构建 | `EGAD/build-val/epic_driver`(带故障注入、state-hash、negative-control hook) |
| `crash_recover_sweep.sh` | 每个 (crash epoch, phase) 注入**真实 GPU 故障**(CUDA err 700 → 进程死亡),新进程重映射 durable store、回滚到 E-2、重放 E-1/E,比较 `[STATE-HASH-VALMS]` 与无 crash 基线 → PASS/FAIL |
| `ycsb_crash_recover_sweep.sh` | YCSB 版本,gate 是 `[STATE-HASH-VALONLY]` |
| `negative_control.sh` | 证明 hash gate 敏感:1 字节损坏 / 两记录交换会**发散** hash,而非盲过 |

> 需要 GPU + 可写 durable store 目录(用 `/dev/shm`),脚本前后会清理。

---

## 完整流程速览

```
clone --recurse-submodules
  → install_dependencies.sh (sudo, 重启)
  → [可选] setup_epic_stock.sh (上游 stock Epic 溯源)
  → build_binaries.sh ×4 (4 个 epic_driver)
  → preflight (deallocate_hugepages + governor + THP defrag)
  → EGAD_plotting/run_all.sh  →  logs/ (测试日志) → figures/ (图 + CSV)
  → validation/ ×3 (恢复正确性, 需 build-val 二进制)
```

需要我针对其中某一步(比如 test 09 的 TPC-C warehouse sweep 细节、或某个具体 figure 的 claim)再展开吗?