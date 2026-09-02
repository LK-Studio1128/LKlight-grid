# LKlight-grid

**LKlight CPU 网格版（v4 最终版）** —— 全功能分子对接引擎，纯 CPU、跨平台换机即用。

LKlight 是 Python LightDock（GSO 群智能对接）的 Rust 高性能实现；本目录是 **CPU 网格
加速最终版**的独立发布项目：打分把"全原子 30Å 逐对"拆成 **≤10Å cell-list 逐对精确 +
10–30Å 受体静电场网格查表**，全部全原子评分函数（dna/vdw/pydock/cpydock）均获
12–48× 加速，其余 8 族查表/统计势函数天然 <0.2s 无需加速。**同 seed 收敛与官方
exact 完全一致**（top-5 解 ±2Å 100% 重合）。

> 需要 NVIDIA GPU 批量加速？见同级项目 **`../LKlight-GPU`**（Linux CUDA 版，无 GPU
> 自动回退本 CPU 网格路径，功能等价）。

## 一、功能清单（与官方 LightDock 对齐，无遗漏）

- **评分函数 12 族**：dfire / dfire2 / dna / ddna / mj3h / pydock / cpydock / sd /
  pisa / sipper / tobi / vdw（`setup/run/rank/generate/score` 全命令统一入口）
- **搜索**：GSO（glowworm 群智能），邻居空间分箱 + 并行移动；swarm 独立可并行
  （`tools/run_parallel.py` 多进程并发 swarm）
- **高级功能**：ANM 柔性变形（按模态坐标）、restraints 约束、膜环境（dna/pydock/
  cpydock）——约束/膜自动走网格路径；ANM 走精确路径（每 pose 受体变形使场缓存失效，
  属设计取舍，非功能缺失）
- **输出**：gso 轨迹、ranking.list、pose PDB（`generate`）、`rank_by_rmsd.list`；
  `tools/clash_analyze.py` pose 体检、`scan_all.py` 批量扫描
- **数值契约**：vdw 与精确逐位一致；dna/pydock/cpydock 远距场误差 ≤0.5%（有效位姿）

## 二、换机即用（无需任何编译工具链）

| 文件 | 平台 | 依赖 | 用法 |
|---|---|---|---|
| `release_bin/LKlight-mac-arm64` | macOS（Apple Silicon/Intel via Rosetta）| 仅系统库 | 拷走 `chmod +x` 即跑 |
| `release_bin/LKlight-win64.exe` | Windows 10/11 x64 | 仅系统自带 UCRT | 拷走即跑（Windows Server 2022 真机 MSVC 原生编译）|
| `release_bin/LKlight-linux-x64` | Linux x86-64 | **零动态依赖（static-pie）** | 拷走即跑 |

验证：`release_bin/LKlight-linux-x64 score <rec.pdb> <lig.pdb> dna --tx 1 --ty 2 --tz 3`
应输出一行 `Score (DNA): ...`。

## 三、快速上手

```bash
BIN=release_bin/LKlight-mac-arm64          # 换成你平台的二进制

# 1) 前处理：6 swarm × 20 glowworm（普通蛋白-核酸对接推荐 25-50 swarm × 200 glow × 100 步）
$BIN setup rec.pdb lig.pdb dna -s 6 -g 20 --seed 42 --noxt --now

# 2) 逐 swarm 跑（或并行跑，见下）
for i in 0 1 2 3 4 5; do $BIN run setup.json initial_positions_$i.dat 100 dna; done
python3 tools/run_parallel.py $BIN 100 dna 6 6    # 等价，6 进程并行更快

# 3) 排序 + 生成 pose
$BIN rank 6 100
for i in 0 1 2 3 4 5; do $BIN generate lightdock_rec.pdb lightdock_lig.pdb swarm_$i/gso_100.out 20; done
```

常用参数：`--noxt` 跳过加氢（输入自带 H）/ `--now` 去水；`--restraints f` 启用约束；
`--seed N` 复现；评分函数把 `dna` 换成 `pydock` 等任一族即可。

## 四、从源码构建（可选）

```bash
cargo build --release          # 需要 Rust stable；产物 target/release/lklight (LKlight)
cargo test --release           # 34 项测试（含 grid-vs-exact 一致性）
```

## 五、性能参考（RNA 大体系 8218+12625 原子，单 swarm 20 glow × 100 步）

| 评分函数 | v4 耗时 | 相对原版 exact |
|---|---|---|
| vdw | ~1.0 s | 48× |
| dna | ~3 s（Mac）| 19× |
| pydock / cpydock | ~7.6 / ~9.7 s（服务器）| 15× / 12× |

完整基准：`PERF_COMPARE_20260902.md`；部署/换机矩阵：`DEPLOY_PLAYBOOK.md`；
引擎版本说明：`docs/engines/`；源码信息见 `docs/README_source.md`。

## 六、发布信息

- 版本：**v4**（含 2026-09-03 bug 审计修复：约束场景 GPU 批量禁用、musl 全静态产物）
- 测试：34/34 通过；原仓库 git 基准 `d4667c6`（本目录为整理后的独立发布快照）
