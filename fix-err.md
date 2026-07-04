# AFSIM 2.9 工程编译失败与“空地球”问题排查记录

## 结论摘要

本工程当前的 `json` 原始信息与 `txt` 翻译结果在“兵力、航线、区域、设施、阶段动作”的业务语义上基本对应，但 `txt` 不是一个可被 AFSIM 2.9 直接编译并执行出实体行为的完整场景。现象“代码无法编译”和“启动后引擎上是一个空地球”不是单一问题，而是由以下几类问题叠加造成：

1. **启动日志只加载到配置文件，未继续加载平台、传感器、武器和部署文件**，说明解析/编译很可能在早期配置段中断；一旦主入口没有成功走到 `scenarios/laydown_*.txt`，仿真中就不会创建红蓝飞机和机场，因此界面表现为空地球。
2. **`config/dis_interface.txt` 使用了非标准或不完整的 DIS 配置块**。AFSIM 2.9 的 DIS/HLA/GUI 接口配置通常依赖安装包示例中的插件和接口语法，当前仅写 `dis_interface / side_mapping`，很可能被解析器判为未知关键字或无效块，从而阻断后续 include。
3. **`processors/assignments_*.txt` 中使用了伪 API**，例如 `PLATFORM.FollowRoute(...)`、`PLATFORM.SetRadarMode(...)`、`PLATFORM.EngageClosestHostile()`。这些不是可直接在 AFSIM 2.9 场景脚本里调用的通用内置方法；如果解析到这些文件，仍会导致脚本编译失败。
4. **任务阶段语义没有落成可执行的 AFSIM 行为模型**。`phases.json` 记录了“起飞、巡逻、雷达探测、发射导弹”等意图，但当前 `txt` 只是用注释和伪函数表达，并没有配置可执行的任务处理器、行为树/脚本处理器、交战规则、武器发射条件或目标选择逻辑。
5. **平台部署未绑定航线**。`routes.json` 已翻译为 `route red_patrol_route` 和 `route blue_patrol_route`，但 `laydown` 中创建的飞机没有可靠地在平台定义阶段绑定路径；后续依赖伪函数 `FollowRoute`，所以即使平台能被创建，也不会自动按航线巡逻。
6. **网络定义重复**。`platforms/common.txt` 和 `scenarios/nets.txt` 都定义了 `default_net`，在部分 AFSIM 版本/配置下可能引发重复定义或覆盖风险。
7. **部分雷达、武器和事件输出参数可能不是 AFSIM 2.9 对应组件支持的精确语法**。例如雷达 `mode_template`、`attenuation_model itu end_attenuation_model`、武器内嵌 `WSF_RADAR_SENSOR` 的简化写法、`event_output` 的事件名等都需要对照本机 AFSIM 2.9 示例逐条校验。

## JSON 与 TXT 语义一致性检查

### 已保持一致的内容

- `forces.json` 中红方歼-20巡逻机位于经度 `119.2`、纬度 `24.5`、高度 `10000`，`scenarios/laydown_red.txt` 翻译为 `24:30:00.00n 119:12:00.00e altitude 10000 m msl`，数值一致。
- `forces.json` 中蓝方 F-16V 巡逻机位于经度 `119.8`、纬度 `24.5`、高度 `10000`，`scenarios/laydown_blue.txt` 翻译为 `24:30:00.00n 119:48:00.00e altitude 10000 m msl`，数值一致。
- `facilities.json` 中漳州机场 `118.5/24.8`、嘉义机场 `120.9/23.5` 均已翻译到对应机场平台。
- `routes.json` 中红方航线 `119.2E, 24.0N -> 119.2E, 25.0N` 和蓝方航线 `119.8E, 24.0N -> 119.8E, 25.0N` 均已翻译到 `scenarios/routes_*.txt`。
- `areas.json` 中红方交战区域和蓝方中线防御区域均已翻译到 `scenarios/zones_*.txt`。
- `phases.json` 的三个阶段已被拆成 `processors/assignments_red.txt` 与 `processors/assignments_blue.txt` 中的注释和定时执行块。

### 需要修正但不能改变业务语义的点

- 不能把红蓝双方、经纬度、高度、航线、机场和阶段时间随意改掉；这些来自 JSON 的原始信息。
- 可以把伪函数替换为 AFSIM 2.9 可编译的任务处理器/脚本处理器/行为配置，但替换后仍要表达相同语义：
  - `0.0 hr` 开始起飞/进入巡逻；
  - `1.0 hr` 进入雷达探测/跟踪阶段；
  - `1.5 hr` 进入超视距交战阶段；
  - `2.0 hr` 左右完成命中判定与战损评估；
  - 总仿真时间仍为 `4 hr`。

## 根因定位

### 1. 主入口 include 链未完整执行

`scenario.txt` 的 include 顺序是：先 include `setup.txt`，再 include 部署、航线、区域、网络和任务分配文件。`warlock.log` 只显示读取到了：

- `scenario.txt`
- `setup.txt`
- `config/simulation.txt`
- `config/output.txt`
- `config/dis_interface.txt`

日志没有显示继续读取 `platforms/common.txt`、`platforms/air/fighter.txt`、`scenarios/laydown_red.txt`、`scenarios/laydown_blue.txt` 等文件。这与“空地球”现象高度吻合：平台部署文件没有被成功加载，仿真世界里自然没有任何实体。

优先怀疑点是 `config/dis_interface.txt`，因为它正好是日志中最后一个被读取的文件。

### 2. DIS 配置块可能阻断编译

当前文件内容为：

```afsim

dis_interface
   side_mapping
      side red  force_id 2
      side blue force_id 1
   end_side_mapping
end_dis_interface
```

该块过于简化，缺少 AFSIM 2.9 DIS 接口通常需要的协议、应用 ID、站点 ID、端口、广播/组播地址、实体映射、PDU 配置等信息。若当前工程只是本地仿真验证，建议先临时移除或注释 `include_once config/dis_interface.txt`，待场景可编译并能显示实体后，再参考本机 AFSIM 2.9 官方示例恢复 DIS 配置。

### 3. 任务分配文件不是有效的 AFSIM 行为脚本

当前 `processors/assignments_red.txt` 与 `processors/assignments_blue.txt` 在已结束的平台定义外又写了同名 `platform` 块，并在其中写：

```afsim
PLATFORM.FollowRoute("red_patrol_route");
PLATFORM.SetRadarMode("TWS");
PLATFORM.EngageClosestHostile();
```

这些更像从 JSON 阶段动作直译而来的伪代码，并非 AFSIM 2.9 通用场景语法。解决时应改为 AFSIM 支持的方式之一：

- 在平台初始定义中直接配置路径/任务；
- 使用 WSF 脚本处理器，并调用 AFSIM 文档中实际存在的方法；
- 使用任务处理器、交战处理器、武器控制处理器和规则集表达巡逻、搜索、跟踪和发射；
- 若仅需快速验证实体显示，可先删除任务分配 include，只保留平台部署、平台类型、传感器和武器，确保场景先能编译和显示实体。

### 4. 平台没有可靠绑定航线

红蓝航线文件已经存在，但飞机平台没有在部署阶段通过 AFSIM 2.9 支持的路径/任务语法引用这些 route。当前唯一的引用在伪函数 `FollowRoute` 中，因此编译通过前不会执行，编译通过后也未必能执行。

建议将航线语义前移到平台部署或任务处理器中。例如按 AFSIM 2.9 示例使用合法的 `route`/`path`/`task` 绑定语法，确保仿真一启动飞机就有可执行运动任务。

### 5. 网络重复定义

`platforms/common.txt` 定义了 `network default_net`，`scenarios/nets.txt` 又定义了一次 `network default_net`。建议只保留一处定义：

- 若 `default_net` 是全局基础设施，保留在 `platforms/common.txt` 或单独 `scenarios/nets.txt` 均可；
- 更推荐只在 `scenarios/nets.txt` 定义网络，`platforms/common.txt` 只保留通用签名、类型和组件定义。

## 建议修复顺序

### 第一阶段：让场景可以编译并显示实体

1. 临时从 `setup.txt` 中注释或移除 `include_once config/dis_interface.txt`。
2. 临时从 `scenario.txt` 中注释或移除：
   - `include_once processors/assignments_red.txt`
   - `include_once processors/assignments_blue.txt`
3. 去掉重复网络定义，只保留一个 `default_net`。
4. 运行 AFSIM 2.9 编译/加载命令，确认日志继续读取到：
   - `platforms/common.txt`
   - `platforms/air/fighter.txt`
   - `platforms/facility/airbase.txt`
   - `sensors/aesa_radar.txt`
   - `weapons/medium_range_aam.txt`
   - `scenarios/laydown_red.txt`
   - `scenarios/laydown_blue.txt`
5. 启动 GUI，确认至少能看到：
   - `red_j20_patrol`
   - `blue_f16v_patrol`
   - `red_zhangzhou_airbase`
   - `blue_chiayi_airbase`

### 第二阶段：恢复运动与巡逻

1. 保持 `routes.json` 与 `scenarios/routes_*.txt` 的经纬度和高度不变。
2. 用 AFSIM 2.9 支持的路径任务语法绑定：
   - `red_j20_patrol` -> `red_patrol_route`
   - `blue_f16v_patrol` -> `blue_patrol_route`
3. 验证飞机在仿真开始后沿台海中线两侧南北方向运动。

### 第三阶段：恢复雷达探测与交战

1. 将 `SetRadarMode("TWS")` 替换为 AFSIM 2.9 支持的传感器模式切换语法或脚本处理器方法。
2. 将 `EngageClosestHostile()` 替换为明确的目标选择、武器分配、发射条件和交战规则。
3. 若使用脚本处理器，必须把脚本 API 改为本机 AFSIM 2.9 文档中真实存在的方法。
4. 验证事件输出中出现传感器检测、航迹建立、武器发射、命中/脱靶等事件。

### 第四阶段：恢复 DIS 输出

1. 在场景实体、运动、探测、交战均可运行后，再恢复 DIS。
2. 使用 AFSIM 2.9 安装目录中的 DIS 示例作为模板，不要保留当前过度简化的 `dis_interface` 块。
3. 明确配置：
   - exercise/application/site ID；
   - UDP 端口；
   - 广播或组播地址；
   - red/blue 到 DIS force id 的映射；
   - 平台类型到 DIS entity type 的映射；
   - 输出 PDU 类型和刷新频率。

## 推荐的最小可验证改动

为了避免一次性修改过多导致无法定位问题，建议先做一个最小可验证版本：

```text
scenario.txt
  include setup.txt
  include scenarios/laydown_red.txt
  include scenarios/laydown_blue.txt
  include_once scenarios/routes_red.txt
  include_once scenarios/routes_blue.txt
  include_once scenarios/zones_red.txt
  include_once scenarios/zones_blue.txt
  include_once scenarios/nets.txt
  # 暂停 processors/assignments_*.txt

setup.txt
  include_once config/simulation.txt
  include_once config/output.txt
  # 暂停 config/dis_interface.txt
  include_once platforms/common.txt
  include_once platforms/air/fighter.txt
  include_once platforms/facility/airbase.txt
  include_once sensors/aesa_radar.txt
  include_once weapons/medium_range_aam.txt
```

该版本的目标不是完成交战，而是确认平台可被加载和显示。只要 GUI 不再是空地球，就说明“空地球”的主因是 include 链在平台部署前中断，后续再逐项恢复 DIS 和任务脚本。

## 本次环境限制

当前容器中未发现 AFSIM/Warlock 可执行程序，因此无法在本环境直接运行 AFSIM 2.9 编译器验证具体报错行。以上排查基于工程文件、现有 `warlock.log` 的 include 轨迹，以及 AFSIM 场景工程常见加载顺序问题。最终修复应在安装了 AFSIM 2.9 的目标环境中逐步验证。
