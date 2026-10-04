# syscall-anomaly-detection

> 本科毕业设计：**基于 Linux 系统调用序列的 CAD 软件异常行为检测**
> （仓库原名 `syssyssys`，2026-10-04 改名）

## 一、这是什么

在 Linux 上用 **eBPF / BCC** 采集进程的系统调用行为，抽取特征、训练检测模型，
识别 CAD 类软件（FreeCAD、KiCad）的**异常操作**（例如批量导出、项目目录扫描、源码外泄、BOM 外泄）。

目标不是"防病毒"，而是**用系统调用这一层的行为信号，把正常使用与异常操作区分开**。

## 二、方法流程（按代码结构整理）

```
① 采集          linux/scripts/bcc_linux_capture.py（BCC 挂载，抓 syscall 事件）
                linux/scripts/batch_collect_linux.py（批量跑、汇总 raw csv）
② 特征          src/syscall_anomaly/features.py
③ 模型          src/syscall_anomaly/models.py（含隔离森林 baseline：models/isolation/）
④ 多特征融合     models/fusion_advantage/（置信度 / 重要性 / LOUO / 多分类 / 噪声鲁棒性）
⑤ 评估          models/section_7_1/、models/ablation/、models/param_sweep/
⑥ 部署开销      models/deployment/（采集开销 + 推理基准 + 单会话吞吐）
```

## 三、目录结构

| 路径 | 内容 |
|---|---|
| `src/syscall_anomaly/` | 核心库：`features.py` 特征工程 / `models.py` 检测模型 / `schema.py` 数据结构定义 |
| `linux/scripts/` | 采集与评估脚本（BCC 采集、批量收集、消融、逐软件评估、融合分析、开销与吞吐测量、§7.1 复现） |
| `linux/workloads/` | **正常与异常负载的定义**（见下表） |
| `models/ablation/` | 消融实验（freecad / kicad 各一套：结果 csv + summary + 热力图/折线/柱状图） |
| `models/case_study/` | 个案分析（含多轮 revision：`p2_revision` / `p2_v2_probe` / `p2_v3_sc_probe` / `p3_v2`，逐窗口预测 + 会话投票） |
| `models/deployment/` | 部署侧测量：采集开销、推理基准、每会话吞吐 |
| `models/fusion_advantage/` | 多特征融合的优势验证：置信度、特征重要性、LOUO、多分类、噪声鲁棒性 |
| `models/isolation/` | 隔离森林基线（异常召回 / 特征重要性 / 结果汇总） |
| `models/param_sweep/` | 参数扫描（准确率 / 假正率曲线） |
| `models/section_7_1/` | 论文 §7.1 的评估复现（双层级混淆矩阵、逐窗口 dump） |

### 负载定义（`linux/workloads/`）

| 类型 | FreeCAD | KiCad |
|---|---|---|
| 正常 | `freecad_normal_linux.py` | `kicad_normal_linux.py` |
| 异常·批量导出 | `freecad_abnormal_bulk_export_linux.py` | — |
| 异常·项目扫描 | `freecad_abnormal_project_scan_linux.py` | `kicad_abnormal_project_scan_linux.py` |
| 异常·源码外泄 | `freecad_abnormal_source_copy_linux.py` | `kicad_abnormal_source_copy_linux.py` |
| 异常·BOM 外泄 | — | `kicad_abnormal_bom_exfil_linux.py` |

## 四、环境与运行

> ⚠️ **本节待补**：以下需要我（作者）确认后填写，README 里不猜。
>
> - [ ] 操作系统与内核版本（eBPF 对内核有要求）
> - [ ] 依赖安装（BCC / bcc-python / Python 三方库清单）
> - [ ] 复现一条完整链路的最小命令序列（采集 → 特征 → 训练 → 评估）
> - [ ] 各脚本的入参说明（尤其是 `batch_collect_linux.py` 与 `run_7_1_evaluation.py`）
> - [ ] 原始数据是否随仓库发布（`models/` 下目前只有结果，无原始 syscall trace）

## 五、说明

- 仓库内 `models/` 是**实验结果与图表**，可直接查看；复现需自备实验环境。
- 相关论文与实验细节以毕设论文为准，本 README 只描述代码结构。
