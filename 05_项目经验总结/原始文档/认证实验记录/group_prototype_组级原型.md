# 关键件组聚合原型报告（只读）

> 输入：`runs/merge_folder/orig.json` + `plm.json`；参照：诊断 JSON。
> 组键：`(表号, 分组标题, 关键件名称)`。选型键：`(型号, 位号, 规格材料, 证书号)`。

## 1. 组级差异分布

| 状态 | 组数 |
|---|---:|
| ONLY_PLM_GROUP | 15 |
| ONLY_ORIG_GROUP | 11 |
| REPLACED_GROUP | 8 |
| SAME_GROUP | 6 |
| SHRUNK_BY_PLM | 3 |

## 2. 未覆盖行按类别 → 组级视角覆盖率

命中口径：该未覆盖行的 `(表, 关键件名称)` 能落到至少一个非 `SAME_GROUP` 的组，说明组级视角识别为业务操作。

| 类别 | 未覆盖行 | 落在非 SAME_GROUP | 未映射到任何组 |
|---|---:|---:|---:|
| demo_identity_only_content_diff+strict_input_match | 17 | 17 | 0 |
| input_name_model_only | 15 | 15 | 0 |
| demo_identity_only_content_diff | 6 | 5 | 0 |
| input_identity_but_not_demo+strict_input_match | 2 | 2 | 0 |
| not_in_input_identity | 1 | 1 | 0 |

## 3. `input_name_model_only` 落到的组

- 表1 行15 `机内电源单元` / `53772-A` → [REPLACED_GROUP] (无分组) orig=1 plm=1
- 表1 行37 `天线隔离器(内置)` / `内置于主板RSAG7.820.51943*结构1上` → [REPLACED_GROUP] (无分组) orig=1 plm=1
- 表1 行38 `天线隔离器电容(内置)` / `GR431BR7LA471KW01L` → [REPLACED_GROUP] (无分组) orig=7 plm=8
- 表1 行39 `天线隔离器电容(内置)` / `C3216X7R471KQPS` → [REPLACED_GROUP] (无分组) orig=7 plm=8
- 表1 行40 `天线隔离器电容(内置)` / `GRM31CR73A103KW03L` → [REPLACED_GROUP] (无分组) orig=7 plm=8
- 表1 行41 `天线隔离器电容(内置)` / `C3216X7R103KPP` → [REPLACED_GROUP] (无分组) orig=7 plm=8
- 表1 行42 `天线隔离器电容(内置)` / `GRM31BR73A472KW01` → [REPLACED_GROUP] (无分组) orig=7 plm=8
- 表1 行43 `天线隔离器电容(内置)` / `C3216X7R472KPPS` → [REPLACED_GROUP] (无分组) orig=7 plm=8
- 表1 行44 `天线隔离器电容(内置)` / `TCC1206X7R472K102FT` → [REPLACED_GROUP] (无分组) orig=7 plm=8
- 表1 行45 `天线隔离器电容(内置)` / `GRM31BR73A222KW75L` → [REPLACED_GROUP] (无分组) orig=7 plm=8
- 表1 行178 `X电容` / `MKP系列` → [ONLY_ORIG_GROUP] 53772-a型机内电源单元上元器件 orig=5 plm=0; [ONLY_PLM_GROUP] 55431-f内电源单元上元器件 orig=0 plm=7
- 表1 行189 `压敏电阻` / `CNR-14V681K` → [ONLY_ORIG_GROUP] 53772-a型机内电源单元上元器件 orig=4 plm=0; [ONLY_PLM_GROUP] 55431-f内电源单元上元器件 orig=0 plm=3
- 表1 行191 `压敏电阻` / `TVR14681-V` → [ONLY_ORIG_GROUP] 53772-a型机内电源单元上元器件 orig=4 plm=0; [ONLY_PLM_GROUP] 55431-f内电源单元上元器件 orig=0 plm=3
- 表1 行192 `压敏电阻` / `TVR14681-V` → [ONLY_ORIG_GROUP] 53772-a型机内电源单元上元器件 orig=4 plm=0; [ONLY_PLM_GROUP] 55431-f内电源单元上元器件 orig=0 plm=3
- 表1 行193 `继电器` / `HF46FA/12-HT` → [ONLY_ORIG_GROUP] 53772-a型机内电源单元上元器件 orig=1 plm=0; [ONLY_PLM_GROUP] 55431-f内电源单元上元器件 orig=0 plm=1

## 4. 非 SAME_GROUP 组一览（前 20）

| 表 | 分组 | 关键件名称 | 状态 | orig 行 | plm 行 | 共同选型 | 仅 orig | 仅 plm |
|---|---|---|---|---:|---:|---:|---:|---:|
| 1 | (无) | pcb(主板,电源板) | REPLACED_GROUP | 22 | 22 | 21 | 1 | 1 |
| 1 | (无) | 内部布线(电源到屏驱动) | SHRUNK_BY_PLM | 39 | 33 | 33 | 6 | 0 |
| 1 | (无) | 内部布线(电源板到主板连接线,主板到扬声器的连接 | SHRUNK_BY_PLM | 36 | 31 | 31 | 5 | 0 |
| 1 | (无) | 后壳材料 | REPLACED_GROUP | 5 | 11 | 1 | 4 | 10 |
| 1 | (无) | 天线隔离器(内置) | REPLACED_GROUP | 1 | 1 | 0 | 1 | 1 |
| 1 | (无) | 天线隔离器电容(内置) | REPLACED_GROUP | 7 | 8 | 6 | 1 | 2 |
| 1 | (无) | 机内电源单元 | REPLACED_GROUP | 1 | 1 | 0 | 1 | 1 |
| 1 | (无) | 电线组件 | ONLY_ORIG_GROUP | 1 | 0 | 0 | 1 | 0 |
| 1 | (无) | 电线组件(0.5平方毫米线径的电线组件,长度小于 | ONLY_PLM_GROUP | 0 | 11 | 0 | 0 | 11 |
| 1 | (无) | 绝缘垫片 | REPLACED_GROUP | 2 | 5 | 1 | 1 | 4 |
| 1 | 53772-a型机内电源单元上元器件 | icx及关联电阻器 | ONLY_ORIG_GROUP | 1 | 0 | 0 | 1 | 0 |
| 1 | 53772-a型机内电源单元上元器件 | x电容 | ONLY_ORIG_GROUP | 5 | 0 | 0 | 5 | 0 |
| 1 | 53772-a型机内电源单元上元器件 | y电容 | ONLY_ORIG_GROUP | 4 | 0 | 0 | 4 | 0 |
| 1 | 53772-a型机内电源单元上元器件 | 光电耦合器 | ONLY_ORIG_GROUP | 3 | 0 | 0 | 3 | 0 |
| 1 | 53772-a型机内电源单元上元器件 | 压敏电阻 | ONLY_ORIG_GROUP | 4 | 0 | 0 | 4 | 0 |
| 1 | 53772-a型机内电源单元上元器件 | 开关变压器 | ONLY_ORIG_GROUP | 3 | 0 | 0 | 3 | 0 |
| 1 | 53772-a型机内电源单元上元器件 | 开关电源变压器 | ONLY_ORIG_GROUP | 3 | 0 | 0 | 3 | 0 |
| 1 | 53772-a型机内电源单元上元器件 | 泄放电阻 | ONLY_ORIG_GROUP | 2 | 0 | 0 | 2 | 0 |
| 1 | 53772-a型机内电源单元上元器件 | 熔断器 | ONLY_ORIG_GROUP | 3 | 0 | 0 | 3 | 0 |
| 1 | 53772-a型机内电源单元上元器件 | 继电器 | ONLY_ORIG_GROUP | 1 | 0 | 0 | 1 | 0 |
