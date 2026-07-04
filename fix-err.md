# AFSIM 2.9 工程编译失败与“空地球”问题排查记录

## 本次已处理的实报错误

用户实测报错：

```text
***** ERROR: Could not find network end_network
'/home/fzxs/workspaces/test/zhiji-test/scenario_cf784ef3_0194_4e6d_bbbf_0df470df0e0f/platforms/common.txt', line 11, near column 12
Reading of simulation input failed
```

该错误的直接原因是自动生成的 TXT 中写入了如下顶层网络块：

```afsim
network default_net
end_network
```

在目标 AFSIM 2.9 解析器中，这种写法不是可接受的网络定义块；解析器把后续 `end_network` 当成另一个 `network` 命令的参数继续解析，于是报出 `Could not find network end_network`。因为错误发生在 `setup.txt` 读取平台通用定义阶段，解析在进入 `scenarios/laydown_red.txt`、`scenarios/laydown_blue.txt`、航线和区域文件之前已经失败，所以 GUI 中仍然是空地球，看不到设施、平台、区域和航线。

## 已做的代码修复

1. 已从 `platforms/common.txt` 删除非法的顶层 `network default_net / end_network` 块，只保留战斗机雷达散射截面积签名 `fighter_rcs`。
2. 已从 `scenarios/nets.txt` 删除同类非法网络块，保留文件作为 JSON/TXT 语义占位，并记录逻辑网络名 `default_net` 与 `weapons_subnet`。
3. 已临时禁用战斗机和机场平台中的 `WSF_COMM_TRANSCEIVER` 组件，避免网络块删除后又因 `network_name default_net` 触发新的网络解析/引用错误；该场景 JSON 未定义通信行为，所以此改动不改变原始兵力、设施、区域、航线或阶段语义。
4. 未改变 JSON 原始语义中的红蓝双方、经纬度、高度、机场、航线、区域和阶段时间。

> 说明：当前修复的目标是先消除已确认的编译阻断点，让 AFSIM 能继续读到部署、航线和区域文件，从而解决“空地球”的第一层原因。通信网络与 transceiver 的精确 AFSIM 2.9 写法应在实体可见后，再根据目标安装环境中的官方示例恢复。

## JSON 与 TXT 语义一致性检查

- `forces.json` 中红方歼-20巡逻机位于经度 `119.2`、纬度 `24.5`、高度 `10000`，`scenarios/laydown_red.txt` 翻译为 `24:30:00.00n 119:12:00.00e altitude 10000 m msl`，数值一致。
- `forces.json` 中蓝方 F-16V 巡逻机位于经度 `119.8`、纬度 `24.5`、高度 `10000`，`scenarios/laydown_blue.txt` 翻译为 `24:30:00.00n 119:48:00.00e altitude 10000 m msl`，数值一致。
- `facilities.json` 中漳州机场 `118.5/24.8`、嘉义机场 `120.9/23.5` 均已翻译到对应机场平台。
- `routes.json` 中红方航线 `119.2E, 24.0N -> 119.2E, 25.0N` 和蓝方航线 `119.8E, 24.0N -> 119.8E, 25.0N` 均已翻译到 `scenarios/routes_*.txt`。
- `areas.json` 中红方交战区域和蓝方中线防御区域均已翻译到 `scenarios/zones_*.txt`。
- `phases.json` 的三个阶段已被拆成 `processors/assignments_red.txt` 与 `processors/assignments_blue.txt` 中的注释和定时执行块。

## 后续仍需逐项验证的问题

当前已修正明确报出的 `network/end_network` 编译错误。如果目标环境继续报错，建议按以下顺序处理，不要一次性大改，以免破坏 JSON 与 TXT 的语义对应关系。

### 1. DIS 配置可能仍会阻断编译

`setup.txt` 当前仍 include `config/dis_interface.txt`。如果下一次日志停在该文件，应先临时注释该 include。当前 DIS 块过于简化，缺少目标 AFSIM 2.9 环境通常需要的站点 ID、应用 ID、端口、地址、实体映射和 PDU 配置。建议在平台、设施、航线和区域可见后，再用目标安装目录中的 DIS 示例恢复。

### 2. 任务分配文件仍是伪行为脚本

`processors/assignments_red.txt` 与 `processors/assignments_blue.txt` 中的：

```afsim
PLATFORM.FollowRoute("red_patrol_route");
PLATFORM.SetRadarMode("TWS");
PLATFORM.EngageClosestHostile();
```

不是可保证被 AFSIM 2.9 识别的通用内置 API。若解析到任务文件时报错，应临时注释 `scenario.txt` 中两个 `processors/assignments_*.txt` include，先验证实体、航线和区域显示；随后再用 AFSIM 2.9 支持的任务处理器、脚本处理器、交战处理器或规则集重新表达相同阶段语义。

### 3. 航线显示与飞机运动是两个问题

`scenarios/routes_red.txt` 和 `scenarios/routes_blue.txt` 已保留原始 JSON 航线语义。GUI 能否显示航线取决于前面的编译是否能走到 route 文件；飞机是否沿航线运动还取决于平台是否通过 AFSIM 支持的任务/路径语法绑定该 route。当前自动生成代码只在伪函数 `FollowRoute` 中引用航线，因此后续仍需替换为目标 AFSIM 2.9 可执行的路径绑定方式。

## 推荐验证顺序

1. 重新加载 `scenario.txt`，确认不再出现 `Could not find network end_network`。
2. 检查加载日志是否继续读到：
   - `platforms/air/fighter.txt`
   - `platforms/facility/airbase.txt`
   - `sensors/aesa_radar.txt`
   - `weapons/medium_range_aam.txt`
   - `scenarios/laydown_red.txt`
   - `scenarios/laydown_blue.txt`
   - `scenarios/routes_red.txt`
   - `scenarios/routes_blue.txt`
   - `scenarios/zones_red.txt`
   - `scenarios/zones_blue.txt`
3. 如果仍是空地球，优先看日志最后一个成功读取的文件；最后一个文件后的语法块就是下一个阻断点。
4. 如果平台和设施出现，但航线/区域未出现，检查 GUI 图层显示开关以及 route/zone 语法是否被当前 AFSIM 2.9 可视化工具支持。
5. 如果平台、设施、航线、区域都出现，再恢复 DIS 和任务脚本。

## 本次环境限制

当前容器中未发现 AFSIM/Warlock 可执行程序，因此无法在本环境直接运行 AFSIM 2.9 编译器验证完整加载链。本次修改针对用户提供的明确错误进行修复，并保留了 JSON/TXT 的业务语义对应关系。
