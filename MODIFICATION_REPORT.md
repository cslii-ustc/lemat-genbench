# 氧化态分配、BVlain/HamGNN readiness 与 ESW 集成修改说明

本文档说明当前程序相对最初程序的主要改动。修改目标是：在 validity 阶段加入元素有效性检查，过滤掉本任务不接受的元素；在 comprehensive run 请求 `migration_barrier` 或包含 migration barrier 的 `property` 时，提前完成氧化态分配和 BVlain readiness 检查；在请求 HamGNN `band_gap` 或包含 HamGNN band gap 的 `property` 时，从当前服务器的 OpenMX `DFT_DATA19` 目录读取支持元素并做 HamGNN readiness 检查；新增 Li-exchange-only ESW 性质计算；不支持请求性质计算的结构会被过滤，支持的结构继续执行后续计算。

## 1. 总体行为变化

最初程序的流程是：

```text
load structures
-> validity preprocessing
   -> forbidden-element validity check
-> filter overall_valid structures
-> remaining preprocessors
-> remaining benchmarks
```

当前程序在 comprehensive runner 中增加了一个可选 property gating 步骤：

```text
load structures
-> validity preprocessing
   -> structural validity
   -> forbidden-element validity check
   -> optional oxidation-state assignment and BVlain readiness check
-> filter overall_valid structures
-> optional property_gating
   -> optional HamGNN/OpenMX element-support readiness check
   -> keep structures ready for the requested property calculations
   -> filter out non-ready structures
-> remaining preprocessors
   -> optional ESW preprocessing for esw/property runs
-> remaining benchmarks
```

当前默认配置是：

```yaml
property_gating:
  enabled: true
  required_families: ["migration_barrier", "band_gap", "property"]
  failure_policy: filter
  check_hamgnn_elements: true
  hamgnn_dft_data: null
```

因此，只有当 `--families` 包含需要 readiness 的 property families 时，才启用 property gating。默认 comprehensive run 没有包含 `migration_barrier`、`band_gap` 或 `property`，因此不受影响。

## 2. 语义说明

这次改动扩展了 `overall_valid` 的定义。

- `overall_valid` 现在由 charge neutrality、interatomic distance、physical plausibility、element validity 共同决定。
- `element_valid=False` 表示结构含有本任务禁用的元素。
- 氧化态分配、BVlain readiness 和 HamGNN readiness 不会写入 `overall_valid`。
- readiness 只作为后续 property 计算前的门控条件。
- 只有 `overall_valid=True` 的结构才会做 property readiness 检查，避免在已经 invalid 的结构上浪费计算。

当前默认 `failure_policy: filter` 的含义是：

- 支持请求性质计算的结构继续跑后续所有 preprocessors/benchmarks。
- 不支持请求性质计算的结构被跳过。
- 整个 run 不会因为单个结构不支持 BVlain 或 HamGNN/OpenMX 而停止。

如果改成 `abort_run`，则只要有一个 valid 结构不 ready，就会停止后续所有计算。如果改成 `warn`，则只记录问题，不过滤结构。

## 3. 主要文件改动

| 文件 | 改动说明 |
|---|---|
| `src/lemat_genbench/utils/oxidation_state.py` | 新增氧化态分配和 BVlain readiness 工具函数。 |
| `src/lemat_genbench/utils/hamgnn_readiness.py` | 新增 HamGNN/OpenMX readiness 工具函数，从当前 `DFT_DATA19` 安装目录读取支持元素。 |
| `src/lemat_genbench/preprocess/validity_preprocess.py` | 增加 forbidden-element validity 检查，并给 `ValidityPreprocessor` 增加可选氧化态/readiness 记录能力。 |
| `src/lemat_genbench/properties/migration_barrier.py` | migration barrier 计算优先使用 validity 阶段保存的氧化态记录。 |
| `src/lemat_genbench/properties/esw.py` | 新增 Li-exchange-only ESW 核心计算，包括 MACE relaxation、MP entries cache、phase diagram 和 Li chemical-potential scan。 |
| `src/lemat_genbench/preprocess/esw_preprocess.py` | 新增 ESW preprocessor，将 ESW、氧化/还原电位、Ehull 和错误信息写入 structure properties。 |
| `src/lemat_genbench/metrics/esw_metric.py` | 新增 ESW 聚合指标。 |
| `src/lemat_genbench/benchmarks/esw_benchmark.py` | 新增单独 `esw` benchmark family。 |
| `src/lemat_genbench/benchmarks/property_benchmark.py` | `property` benchmark 新增可选 `include_esw`。 |
| `scripts/run_benchmarks.py` | comprehensive runner 新增 `property_gating` 解析、过滤和结果元数据输出。 |
| `src/lemat_genbench/cli.py` | 轻量 CLI 支持 `esw` benchmark，并生成默认 `esw.yaml`。 |
| `src/config/comprehensive.yaml` | 默认启用 property gating，并设置 `failure_policy: filter`；新增 `esw_settings`。 |
| `src/config/esw.yaml` | 新增单独 ESW benchmark 配置。 |
| `src/config/property.yaml` | `property` 配置新增 `include_esw` 和 ESW 参数。 |
| `EVALUATION.md` | 在配置参考中说明 `property_gating` 是 property-readiness filtering。 |
| `tests/test_validity_preprocess.py` | 新增 readiness record 的基础测试。 |
| `tests/test_hamgnn_readiness.py` | 新增 HamGNN/OpenMX 元素支持检查测试。 |
| `tests/test_esw_metric.py` | 新增 ESW metric/benchmark 的轻量聚合测试。 |

## 4. 元素有效性检查

validity 阶段新增 forbidden-element 检查。默认禁用元素为：

```python
{
    "He", "Ne", "Ar", "Kr", "Xe", "Rn", "Og",
    "Tc", "Pm", "Po", "At", "Fr", "Ra",
    "Ac", "Th", "Pa", "U", "Np", "Pu", "Am", "Cm", "Bk",
    "Cf", "Es", "Fm", "Md", "No", "Lr",
    "Hg",
}
```

如果结构含有上述任意元素：

```python
structure.properties["element_valid"] = False
structure.properties["overall_valid"] = False
```

同时会写入：

```python
structure.properties["element_check_details"]
structure.properties["validity_details"]["element_filter"]
```

示例：

```json
{
  "element_valid": false,
  "element_check_details": {
    "valid": false,
    "elements": ["Hg", "O"],
    "forbidden_elements_present": ["Hg"],
    "forbidden_elements": ["He", "Ne", "...", "Hg"]
  }
}
```

因此，`overall_valid` 现在表示“结构有效性 + 元素适用域”共同成立。

## 5. 氧化态分配记录

新增的核心记录键是：

```python
OXIDATION_STATE_RECORD_KEY = "oxidation_state_record"
MIGRATION_BARRIER_READY_KEY = "migration_barrier_ready"
```

每个 ready check 成功或失败的结构会在 `Structure.properties` 中写入：

```python
structure.properties["oxidation_state_record"]
structure.properties["migration_barrier_ready"]
structure.properties["oxidation_state_status"]
structure.properties["oxidation_state_confidence"]
```

`oxidation_state_record` 中主要包含：

```json
{
  "status": "success",
  "selected_source": "bvanalyzer",
  "confidence_label": "medium_confidence",
  "mobile_ion": "Li1+",
  "total_charge": 0,
  "mean_abs_mismatch": 0.12,
  "max_abs_mismatch": 0.45,
  "oxidation_states_by_element": {
    "Li": 1,
    "O": -2
  },
  "oxidation_states_by_site": [
    {
      "site_index": 0,
      "element": "Li",
      "oxi_state": 1
    }
  ],
  "bvlain_parameters_complete": true,
  "migration_barrier_ready": true,
  "warnings": [],
  "failure_reasons": [],
  "candidate_summary": []
}
```

失败时会记录失败原因，例如：

```json
{
  "status": "failed",
  "mobile_ion": "Li1+",
  "migration_barrier_ready": false,
  "failure_reasons": [
    "mobile ion Li not present"
  ]
}
```

## 6. 氧化态候选来源

当前氧化态分配会按以下来源生成候选：

| 来源 | 说明 |
|---|---|
| `cif` | 如果输入结构本身带 site oxidation states，则优先作为候选。 |
| `bvanalyzer` | 使用 Pymatgen `BVAnalyzer` 分配氧化态。 |
| `composition_guess` | 使用已有 LeMat ICSD 氧化态先验做 composition-level charge balance guess。 |
| `composition_guess_all_states` | 常规候选失败时，允许更宽的 oxidation-state 搜索。 |

候选会检查：

- 总电荷是否接近中性。
- mobile ion 是否存在。
- 元素种类数是否满足 `min_num_elements`。
- mobile ion 氧化态是否匹配，例如 `Li1+` 要求 Li 为 +1。
- 常见化学约束，例如 alkali 为 +1、alkaline-earth 为 +2、F 为 -1、O 不为正价。
- BVS mismatch 统计。
- 如果启用 `check_bvlain_parameters`，尝试调用 BVlain 做参数/readiness 检查。

## 7. HamGNN/OpenMX readiness

新增的核心记录键是：

```python
HAMGNN_READINESS_RECORD_KEY = "hamgnn_readiness_record"
HAMGNN_READY_KEY = "hamgnn_ready"
```

HamGNN readiness 不使用固定的全局支持元素表，而是在运行时读取当前服务器配置的 OpenMX `DFT_DATA19` 目录：

- 优先使用 `property_gating.hamgnn_dft_data`。
- 如果未配置，则读取 `band_gap_settings.backend_kwargs.dft_data`。
- 如果是 `property` family，则也会读取 `property_settings.band_gap_backend_kwargs.dft_data`。
- 如果配置里都没有，则读取环境变量 `HAMGNN_DFT_DATA`。

检查逻辑是：

```text
DFT_DATA19/VPS 中出现的元素
∩ DFT_DATA19/PAO 中出现的元素
-> 当前 OpenMX 安装支持的元素集合
```

如果结构中有元素不在该集合中，则：

```python
structure.properties["hamgnn_ready"] = False
structure.properties["hamgnn_readiness_record"]["unsupported_elements"] = [...]
```

如果 `DFT_DATA19` 路径没有配置、路径不存在，或无法从 `VPS`/`PAO` 读出支持元素，也会记为 not ready，并在 `failure_reasons` 中写明原因。

## 8. ValidityPreprocessor 的变化

`ValidityPreprocessor` 新增可选参数：

```python
forbidden_elements: list[str] | tuple[str, ...] | None = None
assign_oxidation_states: bool = False
mobile_ion: str = "Li1+"
require_mobile_ion: bool = False
migration_min_num_elements: int | None = None
check_bvlain_parameters: bool | str = False
oxidation_charge_tolerance: float = 1e-3
bvlain_settings: Dict[str, Any] | None = None
```

氧化态/readiness 默认保持关闭，因此直接使用 `ValidityPreprocessor()` 时，不会额外调用 BVlain。forbidden-element 检查默认启用；如需禁用，可传入 `forbidden_elements=[]`。

当 comprehensive runner 需要 migration/property gating 时，会传入：

```python
assign_oxidation_states=True
mobile_ion="Li1+"
require_mobile_ion=True
migration_min_num_elements=2
check_bvlain_parameters=True
```

生成的 validity final scores 会额外包含：

```json
{
  "element_validity_count": 95,
  "element_validity_ratio": 0.95,
  "oxidation_state_success_count": 82,
  "oxidation_state_success_ratio": 0.82,
  "migration_barrier_ready_count": 82,
  "migration_barrier_ready_ratio": 0.82
}
```

其中 `element_validity_*` 始终反映 validity 元素检查；`oxidation_state_*` 和 `migration_barrier_ready_*` 只在实际启用氧化态/readiness 检查时出现。

## 9. Comprehensive runner 的变化

`scripts/run_benchmarks.py` 新增两个主要函数：

```python
get_property_gating_settings(...)
apply_property_gating(...)
```

`get_property_gating_settings` 负责从配置中解析：

- 是否启用 gating。
- 哪些 families 会触发 gating。
- 本次运行是否需要 migration/BVlain readiness。
- 本次运行是否需要 HamGNN/OpenMX readiness。
- mobile ion。
- readiness 所需最小元素种类数。
- BVlain 参数。
- HamGNN `DFT_DATA19` 路径。
- `failure_policy`。

`apply_property_gating` 在 validity 之后、其他 preprocessors 之前执行：

```text
valid_structures
-> keep structures ready for all requested property calculations
-> record blocked structures and reasons
-> return filtered structures
```

默认 `filter` 策略下，被过滤的结构不会进入后续：

- fingerprint/distribution/stability preprocessors
- `migration_barrier`
- `band_gap`
- `property`
- 其他 remaining benchmarks

这满足“只跳过不支持请求性质计算的结构，其他正常结构继续计算”的需求。

如果本次只请求 HamGNN `band_gap`，不会启用 BVlain readiness，也不会要求结构含 Li。如果本次只请求 `migration_barrier`，不会启用 HamGNN/OpenMX 元素支持检查。`property` family 会根据 `include_band_gap`、`include_migration_barrier` 和 `band_gap_backend` 决定需要哪些 readiness。

## 10. 输出变化

结果 JSON 中新增 `validity_filtering.property_gating`：

```json
{
  "enabled": true,
  "migration_enabled": true,
  "hamgnn_enabled": true,
  "failure_policy": "filter",
  "mobile_ion": "Li1+",
  "check_bvlain_parameters": true,
  "require_mobile_ion": true,
  "min_num_elements": 2,
  "check_hamgnn_elements": true,
  "hamgnn_dft_data": "/path/to/DFT_DATA19",
  "required_families": ["band_gap", "migration_barrier", "property"],
  "input_valid_structures": 100,
  "ready_structures": 82,
  "blocked_structures": 18,
  "migration_ready_structures": 90,
  "hamgnn_ready_structures": 88,
  "ready_structure_ids": [0, 1, 4],
  "blocked_structure_details": [
    {
      "structure_id": 7,
      "original_source": "bad.cif",
      "oxidation_state_status": "failed",
      "hamgnn_status": "success",
      "failure_reasons": ["bvlain parameter check failed: ..."],
      "warnings": []
    }
  ]
}
```

结果 JSON 中还会新增 `validity_filtering.oxidation_state_records`：

```json
[
  {
    "structure_id": 0,
    "original_source": "LiFePO4.cif",
    "record": {
      "status": "success",
      "selected_source": "bvanalyzer",
      "mobile_ion": "Li1+",
      "migration_barrier_ready": true,
      "oxidation_states_by_site": []
    }
  }
]
```

当启用 HamGNN readiness 时，还会新增 `validity_filtering.hamgnn_readiness_records`：

```json
[
  {
    "structure_id": 0,
    "original_source": "LiFePO4.cif",
    "record": {
      "status": "success",
      "hamgnn_ready": true,
      "dft_data": "/path/to/DFT_DATA19",
      "elements": ["Fe", "Li", "O", "P"],
      "unsupported_elements": [],
      "supported_element_count": 80,
      "failure_reasons": [],
      "warnings": []
    }
  }
]
```

终端摘要中会显示：

```text
Property-ready structures: 82 / 100
Migration-barrier ready: 90 / 100
HamGNN/OpenMX ready: 88 / 100
```

日志中会显示：

```text
Property gating: 82/100 valid structures are property-ready
Property gating filtered 18 structures before remaining benchmarks.
```

## 11. Migration barrier 计算变化

最初 `migration_barrier.py` 的 BVlain 计算主要依赖：

- `oxi_check=True`
- `add_oxidation_state_by_guess`

当前程序新增优先路径：

```text
1. 如果结构 properties 里已有 oxidation_state_record，则用记录里的 site-wise 氧化态装饰结构。
2. 如果没有记录，则临时调用 assign_oxidation_states_for_bvlain。
3. 如果前两步失败，再走原来的 BVlain fallback chain。
```

这样 comprehensive run 中 validity 阶段分配好的氧化态可以被后面的 migration barrier 复用，避免重复和不一致。

## 12. 配置变化

`src/config/comprehensive.yaml` 新增：

```yaml
validity_settings:
  forbidden_elements:
    - He
    - Ne
    - Ar
    - Kr
    - Xe
    - Rn
    - Og
    - Tc
    - Pm
    - Po
    - At
    - Fr
    - Ra
    - Ac
    - Th
    - Pa
    - U
    - Np
    - Pu
    - Am
    - Cm
    - Bk
    - Cf
    - Es
    - Fm
    - Md
    - No
    - Lr
    - Hg

property_gating:
  enabled: true
  required_families: ["migration_barrier", "band_gap", "property"]
  failure_policy: filter
  mobile_ion: Li1+
  require_mobile_ion: true
  min_num_elements: 2
  check_bvlain_parameters: true
  oxidation_charge_tolerance: 0.001
  check_hamgnn_elements: true
  hamgnn_dft_data: null

migration_barrier_settings:
  mobile_ion: Li1+
  dimensionality: 3d
  r_cut: 10.0
  resolution: 0.2
  k: 100
  encut: 5.0
  fast_threshold: 0.6
  n_jobs: 1
  timeout: 30
```

如果后续希望改成其他策略，只需要改：

```yaml
failure_policy: warn
```

或：

```yaml
failure_policy: abort_run
```

`hamgnn_dft_data: null` 表示不在配置中写死路径，运行时使用 `band_gap_settings.backend_kwargs.dft_data`、`property_settings.band_gap_backend_kwargs.dft_data` 或环境变量 `HAMGNN_DFT_DATA`。

## 13. 测试和验证

新增测试文件：

```text
tests/test_validity_preprocess.py
tests/test_hamgnn_readiness.py
tests/test_esw_metric.py
```

覆盖内容：

- 开启氧化态分配时，validity preprocessor 能写入 `oxidation_state_record`。
- LiF 这类含 Li 结构会得到 `migration_barrier_ready=True`。
- 不含 Li 的 Si 结构会得到 `migration_barrier_ready=False` 并记录失败原因。
- 含 forbidden element 的 Hg 结构会得到 `element_valid=False` 和 `overall_valid=False`。
- 纯 Li 结构虽然含 Li，但因为元素种类数不足 2，会得到 `migration_barrier_ready=False`。
- HamGNN readiness 会从模拟的 `DFT_DATA19/VPS` 和 `DFT_DATA19/PAO` 目录读取支持元素。
- 当结构含有当前 OpenMX 安装不支持的元素时，会得到 `hamgnn_ready=False` 并记录 `unsupported_elements`。
- ESW metric 能正确聚合有效/缺失 ESW 值。
- ESW target window 会将 `fraction_in_target_window` 作为 primary metric。
- ESW benchmark 在 `preprocess=False` 时能读取已有 `structure.properties["esw"]`。

已执行的静态验证：

```text
python -m py_compile ...
git diff --check
```

这些检查已通过。

已执行的 pytest：

```text
uv run pytest tests/test_esw_metric.py tests/test_validity_preprocess.py tests/test_hamgnn_readiness.py
```

结果为 `9 passed`。测试中没有运行真实 MACE relaxation 或 MP API 调用，只覆盖轻量聚合和 readiness 逻辑。

## 14. 兼容性说明

- 直接运行普通 validity benchmark 时，默认不启用氧化态分配，但会启用 forbidden-element validity 检查。
- 如需完全恢复最初元素行为，可构造 `ValidityPreprocessor(forbidden_elements=[])`。
- 单独运行 `lemat-genbench ... migration_barrier` 时，没有 comprehensive validity 阶段，但 migration barrier 内部仍会临时分配氧化态并保留原 fallback。
- comprehensive run 中只有请求 `migration_barrier`、`band_gap` 或 `property` 且配置要求时才启用 property gating。
- HamGNN readiness 只在请求 HamGNN backend 的 band gap 时启用；如果 band gap backend 改成 `alignn`，不会做 OpenMX `DFT_DATA19` 元素支持检查。
- `overall_valid` 会受 forbidden-element validity 影响，但不受 BVlain/HamGNN readiness 影响，避免把“元素适用域”和“某个性质是否可计算”混在一起。

## 15. ESW 集成

新增 `esw` benchmark family，用于计算 Li-exchange-only electrochemical stability window。该实现沿用 `esw_ea_calc.py` 的定义：

```text
ESW = widest continuous relative Li chemical-potential interval
      where abs(Li_exchange) <= gpd_stability_tol
```

当前 ESW 不加入 reaction-energy / grand-potential e_above_hull 阈值，因此应解释为 Li-exchange-only ESW。

ESW 计算流程是：

```text
input structure
-> MACE relaxation
-> relaxed structure + MACE energy
-> MP entries for chemical system
-> PhaseDiagram + target ComputedStructureEntry
-> GrandPotentialPhaseDiagram Li chemical-potential scan
-> esw / reduction_potential / oxidation_potential / ehull
```

默认设备策略是：

```yaml
device: auto
```

即如果 `torch.cuda.is_available()` 为 true，则使用 `cuda`，否则使用 `cpu`。

每个结构会写入：

```python
structure.properties["esw"]
structure.properties["reduction_potential"]
structure.properties["oxidation_potential"]
structure.properties["ehull"]
structure.properties["esw_error"]
structure.properties["esw_used_relaxed_structure"]
structure.properties["esw_mace_energy"]
structure.properties["esw_relaxed_structure_path"]
structure.properties["esw_stable_mu_min"]
structure.properties["esw_stable_mu_max"]
```

聚合输出包括：

```text
mean_esw
median_esw
min_esw
max_esw
fraction_esw_valid
fraction_in_target_window  # 仅配置 target_min/target_max 后出现
```

默认配置位于：

```yaml
esw_settings:
  preprocess: true
  cache_dir: data/esw_cache
  api_key: null
  mace_model: null
  device: auto
  default_dtype: float64
  fmax: 0.05
  max_steps: 500
  mu_min: -5.0
  mu_max: 0.0
  mu_step: 0.01
  gpd_stability_tol: 0.0001
```

其中：

- `api_key: null` 表示运行时读取环境变量 `MP_API_KEY`。
- `mace_model: null` 表示运行时读取环境变量 `MACE_MODEL_PATH`，若未设置则使用原脚本中的默认 MACE model 路径。
- MP entries 和 MACE relaxation 结果默认缓存到 `data/esw_cache`。
- MACE relaxation、MP entries 获取或 ESW 计算失败时，不会让整个 run 崩溃；该结构写入 `esw=None` 和 `esw_error`，聚合时由 `fraction_esw_valid` 反映成功比例。

`property` benchmark 新增：

```yaml
include_esw: false
```

因此默认 property 行为仍保持 band gap + migration barrier；需要同时跑 ESW 时显式设为 true。
