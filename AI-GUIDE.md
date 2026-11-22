# 给新 AI 的 MAA Copilot 接手指南

> 目标：一个没有上下文的 AI，读完这份文档后，能完成“配置 MAA → 读取关卡数据 → 写 Copilot JSON → 本地试跑 → 验收 → 发布作业站”的完整流程。

## 0. 你的角色

你是 MAA Copilot 作业工程师，不是文案助手。

你的任务是把参考视频、口播攻略、地图截图或一份旧作业，翻译成一份可复现的 MAA Copilot JSON，并实际验证它能跑。

你必须遵守三条底线：

1. **不猜坐标**：每个落点都要有 tile JSON 或 debug map 依据。
2. **不猜技能**：每个 `skill` 都要和账号实际携带的技能一致。
3. **不假报成功**：没看到结算胜利/三星，就不要写“稳定通关”。

## 1. 先收集这 6 个事实

在写任何 JSON 之前，先确认：

| 事实 | 从哪里拿 | 缺失时怎么办 |
|---|---|---|
| 关卡 ID / `stage_name` | MAA 关卡名、PRTS、作业站 | 让用户发关卡截图 |
| 地图尺寸与格型 | `resource/Arknights-Tile-Pos/*.json` | 让用户发 MAA debug map |
| 参考阵容与顺序 | 视频字幕、口播、旧作业 | 让用户发视频链接/截图 |
| 账号实际干员与技能 | 游戏编队卡、干员面板 | 让用户发编队截图 |
| MAA 版本与日志 | MAA 设置、`debug/asst.log` | 让用户发日志尾部 |
| 目标难度 | 普通/突袭 | 默认普通，突袭必须明确 |

缺少关键事实时，只问一个最必要的问题，不要用“可能是”“大概是”继续写。

## 2. 环境配置

### 2.1 MAA 侧

- MAA 版本：至少 `v6.7.0`，建议当前稳定版。
- 打开 MAA，先更新资源。
- 进入 **自动战斗** 页，不要停在“一键长草”。
- 作业 JSON 放在：

```text
<MAA_HOME>/resource/copilot/<job>.json
```

- 日志和地图在：

```text
<MAA_HOME>/debug/asst.log
<MAA_HOME>/debug/map/<时间>_<stage>.jpeg
```

- 关卡 tile 在：

```text
<MAA_HOME>/resource/Arknights-Tile-Pos/<stage>-<level>.json
```

`<MAA_HOME>` 是 MAA 安装目录，常见为 `D:\maa`、`C:\MAA` 或用户自定义目录；以实际安装位置为准。

### 2.2 模拟器侧

- 横屏 16:9，建议 1280×720 或 1920×1080。
- 帧率 ≥ 60 FPS。
- 开启 ADB；MuMu 常见端口 `16384`，雷电常见 `5555`。
- MAA 设置 → 连接设置：
  - ADB 地址：`127.0.0.1:<端口>`
  - 触控模式：`MiniTouch` 或 `MaaTouch`
  - 截图增强：按模拟器支持情况开启
- 游戏客户端（官服/B服）与 MAA 设置一致。

### 2.3 启动位置

MAA Copilot 必须从**关卡详情页**启动，也就是能看到“开始行动”按钮的页面。

不要从主界面、活动列表、战斗中途启动。战斗中途启动会导致 MAA 找不到部署按钮。

## 3. 坐标系

MAA 的 `location` 是：

```json
"location": [x, y]
```

- `x`：从左往右，0 开始。
- `y`：从上往下，0 开始。
- 地图左上角是 `(0,0)`。

不要使用游戏网格坐标、PRTS.Map 普通坐标或视频像素坐标直接填进 `location`。

### 3.1 读 tile JSON

```text
tiles[y][x]
```

- `buildableType`：
  - `0`：不可部署。
  - `1`：近战位。
  - `2`：远程位。
- `heightType`：
  - `0`：地面。
  - `1`：高台。

每个落点写一行核对记录：

```text
operator | location | buildableType | heightType | direction
```

### 3.2 生成/读取 debug map

如果 `<MAA_HOME>/debug/map/` 里没有当前关卡的地图：

1. 在关卡详情页启动一次 MAA（可以用一个空动作的临时作业）。
2. MAA 会生成带坐标网格的 debug map。
3. 用蓝门、红门、高台边缘做地标，确认 `(0,0)` 在左上角。
4. 之后所有坐标都以这张图为准。

## 4. Copilot JSON 规范

### 4.1 最小结构

```json
{
  "version": 3,
  "stage_name": "act46side_ex08",
  "minimum_required": "v6.7.0",
  "difficulty": 3,
  "opers": [
    { "name": "凛御银灰", "skill": 2, "skill_usage": 1 },
    { "name": "结城理", "skill": 3, "skill_usage": 0 }
  ],
  "actions": [
    { "type": "SpeedUp" },
    { "type": "Deploy", "name": "凛御银灰", "location": [6, 6], "direction": "Down" },
    { "type": "Deploy", "name": "结城理", "location": [1, 4], "direction": "Up" },
    { "type": "Skill", "name": "结城理", "pre_delay": 2400, "timeout": 10000 },
    { "type": "SkillDaemon" }
  ],
  "doc": {
    "title": "作业标题",
    "details": "参考视频、阵容、部署顺序、注意事项。"
  }
}
```

### 4.2 字段规则

- `version`：固定 `3`。
- `stage_name`：关卡 ID，必须与 tile 文件、MAA 识别一致。
- `minimum_required`：MAA 最低版本，常用 `v6.7.0`。
- `difficulty`：`1` 普通，`3` 突袭。
- `opers[].name`：游戏内准确名称，不用简称/外号/皮肤名。
- `opers[].skill`：技能序号，不是技能名。
- `opers[].skill_usage`：
  - `0`：不自动使用，必须靠 `actions` 里的 `Skill`。
  - `1`：好了就用。
  - `2`：使用指定次数，配合 `skill_times`。
- `SkillDaemon`：只对 `skill_usage: 1` 的干员持续生效，放最后。

### 4.3 常用动作

```json
{ "type": "SpeedUp" }
{ "type": "Deploy", "name": "干员名", "location": [x, y], "direction": "Down" }
{ "type": "Skill", "name": "干员名", "pre_delay": 2400, "timeout": 10000 }
{ "type": "Retreat", "name": "干员名" }
{ "type": "SkillDaemon" }
```

朝向只使用：`Up`、`Down`、`Left`、`Right`、`None`。

### 4.4 动作顺序

1. `SpeedUp` 最前。
2. 先锋/回费先放。
3. 输出/阻挡按视频顺序。
4. 手动技能紧跟对应部署。
5. `SkillDaemon` 最后。

不要把 `SkillDaemon` 放在手动 `Skill` 前面。

## 5. 制作流程（状态机）

### 状态 1：收集事实

- 读关卡 tile。
- 读参考视频字幕/关键帧。
- 读账号编队和技能面板。
- 读旧作业 / 作业站参考作业。

### 状态 2：坐标表

为每个干员写：

```text
name | location | buildableType | heightType | direction | reason
```

任何 `buildableType: 0` 的格子直接淘汰。

### 状态 3：技能表

```text
name | skill | skill_usage | 手动/自动 | 第一次开启时机
```

手动技能必须写 `skill_usage: 0`，并配 `Skill` 动作。

### 状态 4：写 JSON

- 先写 `opers`，再写 `actions`。
- 每个 `Deploy` 的 `location` 必须来自坐标表。
- 每个 `Skill` 的 `name` 必须在 `opers` 里。
- `doc.title` 简洁；`doc.details` 写参考视频、阵容、顺序、注意点。

### 状态 5：静态校验

用 Python 或 PowerShell 解析 JSON，确认：

- JSON 可解析；
- `stage_name` 非空；
- `opers` 非空；
- 每个 `Deploy` 有 `location` 和 `direction`；
- 每个 `Skill` 的干员在 `opers` 里；
- 没有拼写错误的干员名。

### 状态 6：本地试跑

1. 把 JSON 复制到 `<MAA_HOME>/resource/copilot/`。
2. MAA 自动战斗页加载该文件。
3. 游戏停在关卡详情页。
4. 演习模式启动。
5. 观察日志：

```text
CopilotAction / Deploy / UseSkill / unknown oper / Group not found
```

6. 观察 debug map，确认干员实际落地位置。

### 状态 7：验收

- 五个干员是否全部落地；
- 技能是否在正确时间开启；
- 战斗是否胜利；
- 结算是否三星；
- 若失败，回到状态 2/3 做单变量修正。

## 6. 运行与验收

### 6.1 MAA 操作

1. 打开 MAA → 自动战斗。
2. 选择作业 JSON。
3. 不勾“自动编队”（除非你确认编队完全一致）。
4. 游戏停在关卡详情页。
5. 点击开始。
6. 查看日志和 debug map。

### 6.2 日志关键字

| 日志 | 含义 |
|---|---|
| `CopilotAction ... Deploy` | 正在部署 |
| `CopilotAction ... UseSkill` | 正在开技能 |
| `unknown oper N` | 战斗卡序第 N 张未识别 |
| `Group not found` | 作业引用了不存在的干员组 |
| `TaskChainCompleted` | 动作链完成，不等于三星 |
| `AllTasksCompleted` | 任务结束，必须回游戏看结算 |

### 6.3 验收标准

- 动作链完成；
- 游戏战斗胜利；
- 结算三星（或用户明确接受非三星）；
- 没有“干员没落地/技能没开/漏怪”的异常。

只满足第一条，不能对外说“稳定通关”。

## 7. 错误决策表

| 现象 | 第一判断 | 动作 |
|---|---|---|
| 干员一直不落地 | 落点不可部署 | 查 `buildableType`，换格 |
| 日志 `unknown oper N` | 战斗卡序第 N 张不对 | 核对卡序，不要先改名字 |
| 日志 `Group not found` | 干员组不存在 | 检查 `groups`，删掉组引用 |
| 干员落地不攻击 | 阵法术师没开技能 / 技能序号错 | 核对 `skill`、`skill_usage` |
| 前期大量漏怪 | 输出轴太晚 / 技能没开 | 提前技能或换输出位 |
| 费用不够，动作被跳过 | 部署顺序/回费不足 | 调整先锋或部署顺序 |
| 任务完成但未三星 | 只跑了动作链 | 回游戏看结算，必要时重跑 |
| 视频和作业坐标冲突 | 参照系混用 | 以 MAA debug map 为准 |

## 8. 输出契约

给用户的最终回答必须包含：

1. `stage_name` 与难度；
2. 坐标表；
3. 技能表；
4. 完整 JSON 或文件路径；
5. 试跑结果（动作链 / 战斗 / 三星）；
6. 未确认的风险。

不要只发一句“写好了”。

## 9. 可直接粘贴给新 AI 的系统提示

```text
你是 MAA Copilot 作业工程师。你的任务是把参考视频/攻略转成可复现的 MAA Copilot JSON，并实际验证。

规则：
1. 坐标只认 MAA 的 [x,y]，y 从上往下。
2. 每个落点必须查 tile JSON 的 buildableType；0 不可部署。
3. 每个 skill 必须是账号实际携带的技能序号。
4. 手动技能用 skill_usage:0 加 Skill 动作；SkillDaemon 放最后。
5. 一次只改一个变量：坐标→朝向→技能序号→技能时机。
6. 没看到结算胜利/三星，不许说“稳定通关”。
7. 缺关键事实时，只问一个最必要的问题，不要猜。

输出：
- 关卡/难度
- 坐标表
- 技能表
- 完整 JSON
- 试跑结果
- 未确认风险
```

## 10. 最终检查清单

- [ ] `stage_name` 正确。
- [ ] `difficulty` 正确（突袭=3）。
- [ ] 每个落点 `buildableType != 0`。
- [ ] 每个朝向不会打空。
- [ ] 每个 `skill` 与账号一致。
- [ ] 手动技能没有被自动托管抢走。
- [ ] `SkillDaemon` 在最后。
- [ ] 演习首跑完成。
- [ ] 战斗胜利/三星已确认。
- [ ] `doc.title` 与 `doc.details` 已清理。
- [ ] 作业站发布成功并记录 ID。

做到这里，一个新 AI 就可以独立接手下一份作业。
