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

## 本次第四次修正：删除 `platforms/common.txt` 文件本身

目标环境连续报告同一个文件同一行错误，而当前主加载链已经不再 include 该文件。为避免任何外部启动器、索引器或旧脚本仍按文件名扫描/读取它，本次直接从仓库删除 `platforms/common.txt`，并同步从 `index.md` 移除该文件索引。

现在 `fighter_rcs` 定义只存在于 `platforms/air/fighter.txt`，主加载链为：`setup.txt` -> `platforms/air/fighter.txt` / `platforms/facility/airbase.txt` / sensors。若实际运行仍报 `/home/fzxs/.../platforms/common.txt`，则说明运行目录没有更新到本次提交，或启动器读取的是另一个副本，而不是当前仓库内容。

## 本次第五次修正：绕开无效武器模型 `WSF_AAM_WEAPON`

新的实测错误是 `weapons/medium_range_aam.txt` 第 5 行找不到 `WSF_AAM_WEAPON`。这说明目标 AFSIM 2.9 环境没有名为 `WSF_AAM_WEAPON` 的武器模型，或该模型不是当前安装包可用插件。为了先解决空地球并让设施、平台、航线、区域加载，本次将武器定义从最小显示加载链中移除：

1. 已从 `setup.txt` 删除 `include_once weapons/medium_range_aam.txt`。
2. 已从 `platforms/air/fighter.txt` 删除引用 `MRM_AAM` 的武器挂载块，避免平台类型引用一个未加载/不可用的武器定义。
3. 已从 `index.md` 移除该武器文件的 include 索引。

注意：`phases.json` 中仍有“发射中距空空导弹”的阶段语义，后续应使用目标 AFSIM 2.9 实际支持的 `WSF_EXPLICIT_WEAPON` / `WSF_IMPLICIT_WEAPON` 或现场插件模型重新实现；但在最小显示验证阶段，不应让武器模型阻断平台、设施、航线和区域显示。

## 本次第六次修正：把部署、航线和区域纳入 `setup.txt`

当前已经没有解析错误，但 GUI 仍为空，最可能的原因是实际启动时只加载了 `setup.txt`，而旧结构中 `setup.txt` 只包含配置和类型定义，真正创建实体的 `scenarios/laydown_*.txt`、航线 `routes_*.txt` 和区域 `zones_*.txt` 只在 `scenario.txt` 中 include。如果启动器入口选成 `setup.txt`，就会“无错误但没有任何实体”。

本次将主加载链调整为：

1. `setup.txt` 负责加载配置、平台类型、传感器、红蓝部署、航线、区域和空的网络占位文件。
2. `scenario.txt` 只保留 `include setup.txt`，避免同一批平台/航线/区域被重复 include。
3. 这样无论 AFSIM 启动器入口选择 `scenario.txt` 还是 `setup.txt`，都能读到设施、平台、航线和区域。

当前仍暂不 include 任务脚本和武器文件，目的是先完成最小可视化：显示红蓝飞机、机场、航线和区域。

## 本次第七次修正：将区域 `point` 改为闭合边界航线

新的实测错误是 `scenarios/zones_red.txt` 中 `point` 值不被目标 AFSIM 2.9 接受。该解析器的 `zone/point` 语法显然不是当前自动生成的经纬度点格式。为了继续推进最小可视化，本次不再使用 `zone ... point ... end_zone`，而是把红方交战区域和蓝方防御区域改写为闭合的 `route` 边界：

1. `scenarios/zones_red.txt` 现在定义 `route red_combat_zone_boundary`，按原 JSON 坐标依次连接四个角点，并回到起点闭合。
2. `scenarios/zones_blue.txt` 现在定义 `route blue_median_line_zone_boundary`，同样按原 JSON 坐标闭合。
3. 这样不改变区域的经纬度语义，并且可以绕开当前不可用的 `point` 区域语法，让 GUI 至少以边界线形式显示区域。

后续如果需要真正的 AFSIM 区域对象，应根据目标环境的官方 `zone` 示例恢复，而不是使用本次已证实会报错的 `point <lat> <lon>` 写法。

## 本次第八次修正：飞机从机场起飞并绑定航线

当前编译和启动已成功，但两架飞机悬停在台海不动，原因是之前的最小显示版本把飞机直接放在台海巡逻点，并且没有把平台绑定到航线。为了符合“从各自基地起飞，进入台湾海峡对抗”的预期，本次做了如下调整：

1. 红方 `red_j20_patrol` 初始位置改为漳州机场，航向向东，并内嵌红方起飞/巡逻 route 块。
2. 蓝方 `blue_f16v_patrol` 初始位置改为嘉义机场，航向向西，并内嵌蓝方起飞/巡逻 route 块。
3. 红蓝航线都从各自机场 0 米高度开始，经爬升点进入台海 10000 米巡逻高度，再向台海中线附近收敛，形成对抗态势。
4. `setup.txt` 中航线 include 已移动到平台部署 include 之前，同时顶层航线仍先于平台部署加载，用于 GUI 航线显示。

这些修改保留 JSON 中机场、巡逻点和高度语义，同时把原来“已经悬停在巡逻点”的静态展示改为“从基地出发进入台海”的动态推演。

## 本次第九次修正：移除平台级 `speed` 命令

新的实测错误是 `scenarios/laydown_red.txt` 中 `speed` 为未知命令，说明目标 AFSIM 2.9 不支持在 `platform` 块内直接写 `speed 450 knots`。本次已从红蓝飞机部署中移除该命令，保留初始位置、航向和 `route ...` 绑定，让运动由航线绑定和平台 mover 负责。

如果后续仍能编译但飞机不动，下一步应根据目标环境的官方示例查找正确的航线任务/速度控制语法，而不是使用平台级 `speed`。

## 本次第十次修正：将平台航线引用改为平台内嵌 route 块

新的实测错误是 `route red_patrol_route` 在 `platform` 块内被解析为未知命令 `red_patrol_route`。这说明目标 AFSIM 2.9 的平台内 `route` 语法不是“引用已有 route 名称”的单行写法，而更像是无参数 `route ... end_route` 块。

本次已将红蓝飞机部署中的单行 `route red_patrol_route` / `route blue_patrol_route` 改为平台内嵌 `route` 块，直接把起飞、爬升、巡逻和中线收敛航路点写入飞机平台内部。顶层 `scenarios/routes_*.txt` 仍保留，用于 GUI 显示航线边界；平台运动则使用内嵌 route 块，避免单行 route 引用触发解析错误。

## 本次环境限制

当前容器中未发现 AFSIM/Warlock 可执行程序，因此无法在本环境直接运行 AFSIM 2.9 编译器验证完整加载链。本次修改针对用户提供的明确错误进行修复，并尽量减少 TXT 中可能被解析器误读的说明文字。
