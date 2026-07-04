# AFSIM 2.9 工程编译失败与“空地球”问题排查记录

## 本次再次修正的错误

用户仍然遇到：

```text
***** ERROR: Could not find network end_network
'/home/fzxs/workspaces/test/zhiji-test/scenario_cf784ef3_0194_4e6d_bbbf_0df470df0e0f/platforms/common.txt', line 11, near column 12
```

上一次虽然删除了真正的网络定义块，但仍在 `platforms/common.txt` 的说明文字中保留了 `network ... end_network` 这样的字样。目标 AFSIM 2.9 解析器仍然把这些字样当作输入 token 处理，所以继续报同样的错误。

## 已做的实际修复

1. `platforms/common.txt` 现在只保留真正需要的雷达散射截面积定义，不再包含任何说明文字或通信相关字样。
2. `scenarios/nets.txt` 已改为空文件，避免该 include 后续再次触发同类解析问题。
3. `platforms/air/fighter.txt` 与 `platforms/facility/airbase.txt` 中的通信组件已经移除，并且也删除了包含通信/网络关键字的说明文字。
4. 以上改动不改变 JSON 原始语义中的红蓝双方、经纬度、高度、机场、航线、区域和阶段时间；只移除了当前 JSON 中本来没有定义的通信层内容。

## 为什么这样能解决当前空地球问题

当前报错发生在 `setup.txt` include `platforms/common.txt` 阶段。只要这里失败，AFSIM 就不会继续读取平台类型、机场类型、传感器、武器、红蓝部署、航线和区域，因此 GUI 中仍然会是空地球。现在 `platforms/common.txt` 已极简化为：

```afsim
radar_signature fighter_rcs WSF_RADAR_SIGNATURE
   constant 0 dbsm
end_radar_signature
```

这会避免当前已知的解析阻断点，让加载流程继续向后执行。

## JSON 与 TXT 语义一致性检查

- 红方歼-20巡逻机仍位于经度 `119.2`、纬度 `24.5`、高度 `10000`，TXT 中为 `24:30:00.00n 119:12:00.00e altitude 10000 m msl`。
- 蓝方 F-16V 巡逻机仍位于经度 `119.8`、纬度 `24.5`、高度 `10000`，TXT 中为 `24:30:00.00n 119:48:00.00e altitude 10000 m msl`。
- 漳州机场、嘉义机场、红蓝航线、红方交战区域和蓝方中线防御区域仍保持原始 JSON 坐标语义。
- 阶段动作的时间语义仍保留在 `processors/assignments_red.txt` 与 `processors/assignments_blue.txt` 中，但其中的伪 API 后续仍可能需要替换成 AFSIM 2.9 的真实任务/脚本语法。

## 下一步验证顺序

1. 重新加载 `scenario.txt`，确认不再出现 `Could not find network end_network`。
2. 查看日志是否能继续读取到 `platforms/air/fighter.txt`、`platforms/facility/airbase.txt`、`sensors/aesa_radar.txt`、`weapons/medium_range_aam.txt`。
3. 查看日志是否能继续读取到 `scenarios/laydown_red.txt`、`scenarios/laydown_blue.txt`、`scenarios/routes_red.txt`、`scenarios/routes_blue.txt`、`scenarios/zones_red.txt`、`scenarios/zones_blue.txt`。
4. 如果平台、设施、航线、区域仍不显示，请继续提供下一条新的 AFSIM 报错。下一条报错所在文件就是新的阻断点。
5. 如果实体已经显示，再逐步恢复通信、DIS 和任务脚本，不要在最小显示验证阶段恢复这些非必要模块。

## 本次第三次修正：绕开 `platforms/common.txt` include

如果目标环境仍然报告 `platforms/common.txt` 第 11 行存在 `network/end_network` 错误，而当前仓库中的 `platforms/common.txt` 实际只有 3 行，说明运行环境很可能仍在读取旧版本文件或缓存副本。为避免该文件继续成为阻断点，本次进一步做了结构性规避：

1. 已从 `setup.txt` 删除 `include_once platforms/common.txt`。
2. 已将 `fighter_rcs` 雷达散射截面积定义移动到 `platforms/air/fighter.txt` 顶部，在战斗机平台类型使用它之前完成定义。
3. `platforms/common.txt` 虽保留在仓库中，但不再参与主加载链；因此即使目标环境中该文件存在旧内容，只要使用本次更新后的 `setup.txt`，AFSIM 也不会再读取它。

请确认实际运行目录中的 `setup.txt` 已更新，并且其中不再包含 `include_once platforms/common.txt`。

## 本次环境限制

当前容器中未发现 AFSIM/Warlock 可执行程序，因此无法在本环境直接运行 AFSIM 2.9 编译器验证完整加载链。本次修改针对用户提供的明确错误进行修复，并尽量减少 TXT 中可能被解析器误读的说明文字。
