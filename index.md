# Scenario Project Index

**项目目录**: /home/fzxs/workspaces/test/zhiji-test/scenario_cf784ef3_0194_4e6d_bbbf_0df470df0e0f/
**生成时间**: 2026-07-04T10:14:00Z
**AFSIM 版本**: WSF 2.9

> 想定：台海1v1空战对抗-2030 | 歼-20 vs F-16V | 台海中线巡逻遭遇

## Side 清单

| Side 名称 | 角色 | DIS Force ID | 部署文件 |
|-----------|------|-------------|---------|
| red | 红方/PLA | 2 | scenarios/laydown_red.txt |
| blue | 蓝方/ROCAF | 1 | scenarios/laydown_blue.txt |

## JSON 文件清单（地图渲染）

| 文件路径 | 职责 | 实体数 |
|----------|------|--------|
| scenario.json | 顶层元数据 | - |
| forces.json | 兵力部署 | 2 |
| routes.json | 航路航线 | 2 |
| facilities.json | 设施 | 2 |
| areas.json | 作战区域 | 2 |
| phases.json | 作战阶段 | 3 |

## AFSIM 文件清单（推演执行）

| 文件路径 | 职责 | Include 类型 |
|----------|------|-------------|
| scenario.txt | 主入口 | - |
| setup.txt | 公共设置 | - |
| config/simulation.txt | 仿真配置 | include_once |
| config/output.txt | 输出配置 | include_once |
| config/dis_interface.txt | DIS映射 | include_once |
| platforms/air/fighter.txt | 战斗机类型 | include_once |
| platforms/facility/airbase.txt | 机场类型 | include_once |
| sensors/aesa_radar.txt | AESA雷达 | include_once |
| weapons/medium_range_aam.txt | 中距空空导弹 | include_once |
| scenarios/laydown_red.txt | 红方部署 | include |
| scenarios/laydown_blue.txt | 蓝方部署 | include |
| scenarios/routes_red.txt | 红方航线 | include_once |
| scenarios/routes_blue.txt | 蓝方航线 | include_once |
| scenarios/zones_red.txt | 红方区域 | include_once |
| scenarios/zones_blue.txt | 蓝方区域 | include_once |
| scenarios/nets.txt | 通信网络 | include_once |
| processors/assignments_red.txt | 红方任务 | include_once |
| processors/assignments_blue.txt | 蓝方任务 | include_once |

## JSON ↔ AFSIM 映射

| JSON 实体 | AFSIM 实体 | 映射关系 |
|-----------|-----------|---------|
| forces[0] 红方歼-20巡逻机 | platform red_j20_patrol (RED_FIGHTER) | 歼-20 @ 119.2E,24.5N |
| forces[1] 蓝方F-16V巡逻机 | platform blue_f16v_patrol (BLUE_FIGHTER) | F-16V @ 119.8E,24.5N |
| facilities[0] 红方漳州机场 | platform red_zhangzhou_airbase (AIRBASE) | 118.5E,24.8N |
| facilities[1] 蓝方嘉义机场 | platform blue_chiayi_airbase (AIRBASE) | 120.9E,23.5N |
| routes[0] 红方歼-20巡逻航线 | route red_patrol_route | 24.0N→25.0N @ 119.2E |
| routes[1] 蓝方F-16V巡逻航线 | route blue_patrol_route | 24.0N→25.0N @ 119.8E |
| areas[0] 台海交战区域 | zone red_combat_zone | 118E/23N → 121.5E/26N |
| areas[1] 台海中线 | zone blue_median_line_zone | 119.5E-119.6E/23N-26N |
| phases | processors/assignments_*.txt | 时间触发行为 |
