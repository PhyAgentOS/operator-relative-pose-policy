# 相对位姿策略节点

[English](README.md) | [简体中文](README_zh.md)

独立 Dora 策略节点，为 `motion.relative_pose/resolve` 提供 Forge Query。
本地接收 `forge_msgs.JointState`，经 `forge-kinematics` 计算正运动学，
通过 Tool 线协议返回起始位姿和绝对目标位姿；不依赖 `forge-motion-server`。

与 Motion Action provider 相同，进程按 Endpoint 与领域服务分层：

```text
Forge Query Tool Wire
        -> RelativePoseQueryEndpoint
        -> RelativePoseResolver
             -> JointStateCache
             -> ForwardKinematics
             -> frame transform
```

`RelativePoseQueryEndpoint` 只负责 Forge 截止时间、严格参数解析、`ToolResult` 序列化和错误转换。
`RelativePoseResolver` 不依赖 `forge_tool`，负责状态快照、正运动学、坐标系校验与相对位姿数学。
解析是短时单次查询，两者在同一个 Dora 进程内组合；另起 `RelativeQueryServer` 不增加独立的执行或故障边界。

## Node、Endpoint 与 Tool API

| 项目 | 当前实现 |
| --- | --- |
| 生产 Node | 1：`relative_pose_policy`；源码命令 `forge-relative-motion-policy`，打包入口见 `scripts/build_pyinstaller.sh` |
| Tool 接入方式 | Dora `ToolMessage` provider；通过 Gateway 暴露调用入口 |
| 直接 HTTP Tool API | 0；本 Node 不监听 HTTP，调用地址属于 Gateway |
| Endpoint 数 | 1：`motion.relative_pose` |
| operation 数 | 1：`resolve`（Query，最大并发 1） |
| Gateway Tool 数 | 由 `tools.specs` 决定；仓库不固定外部 Tool ID，也不自动增加 robot-control-sim 的 Tool |

调用映射为 `<tool_id from tools.specs> -> motion.relative_pose / resolve -> relative_pose_policy`。
该 Query 读取 `joint_state`，把相对位姿请求解析成绝对目标；它不发送运动目标。
需要运动时，由上层编排另行调用 Motion Action；Query 与后续 Action 不构成原子操作。

完整参数、返回值、frame/单位、状态新鲜度及错误见 [TOOL_CONTRACTS.md](TOOL_CONTRACTS.md)，
实现见 [query.py](src/relative_pose_policy_node/query.py)。

兼容性：当前依赖声明为 `dora-rs==0.4.1`、`forge-kinematics==1.0.1`、`forge-msgs==1.0.1`；以 [pyproject.toml](pyproject.toml) 与 [uv.lock](uv.lock) 为准。 Tool 协议实现为仓库内 vendored 代码。 这些声明不构成跨版本互操作保证；使用制品时还需核对 build revision 和 Skill lock。

后续修改 Node 入口、Endpoint 或 Tool API 时，请同步更新本节及对应配置/契约，遵循 [Node README 规范](https://github.com/PhyAgentOS/forge/blob/master/CONTRIBUTING.md#node-readme-contract)。

## 安装与运行

Forge 包从公共索引解析（`forge-msgs==1.0.1`、`forge-kinematics==1.0.1`），Tool 协议内置于 `src/forge_tool`：

```bash
uv sync
uv run forge-relative-motion-policy --config relative_pose_policy.yaml
```

配置支持原有单组格式和多组格式。不设置顶层 `groups` 时使用原单组格式：

```yaml
group_name: arm
joint_names: [joint1, joint2, joint3, joint4, joint5, joint6]
urdf_path: ../robots/piper.urdf
base_frame: base_link
tip_frame: link6
```

存在顶层 `groups` 映射时，每个键是组名，每个值严格包含
`joint_names`、`urdf_path`、`base_frame`、`tip_frame`。
所有组共享输入 `joint_state`，按 `group_name` 分发解析：

```yaml
groups:
  xlerobot_left_arm:
    joint_names: [left_arm_shoulder_pan, left_arm_shoulder_lift, left_arm_elbow_flex, left_arm_wrist_flex, left_arm_wrist_roll]
    urdf_path: ../assets/urdf/xlerobot_kinematics.urdf
    base_frame: base_link
    tip_frame: left_arm_tcp
  xlerobot_right_arm:
    joint_names: [right_arm_shoulder_pan, right_arm_shoulder_lift, right_arm_elbow_flex, right_arm_wrist_flex, right_arm_wrist_roll]
    urdf_path: ../assets/urdf/xlerobot_kinematics.urdf
    base_frame: base_link
    tip_frame: right_arm_tcp
```

拒绝混用顶层单组字段与 `groups`，也拒绝空组、重复/空关节列表、组间共享关节及未知字段。
两种格式中的相对 `urdf_path` 都按配置文件目录解析；未知或缺失字段会报错。

## Dora 端口

输入：

- `joint_state`：`forge_msgs.JointState`，仅在进程内缓存，以策略接收时刻计时。
- `tool_in`：Forge `ToolMessage`，包含注册响应与 Query 请求。
- 任意周期输入（例如 `tick`）可驱动租约续期。

输出：

- `tool_out`：注册/注销、Query 响应与 Tool 错误。

运行器组合 `RelativePoseQueryEndpoint`、`RelativePoseResolver`、Forge 公共
`ToolEndpointHandler` 和 `DoraToolEndpointBinding`，不重复实现线协议分发。
Endpoint 级 `EndpointLease` 负责注册与续期。
严格请求/结果契约见 [TOOL_CONTRACTS.md](TOOL_CONTRACTS.md)。

PAOS 维护的 MuJoCo 集成图位于 PhyAgentOS Skill 集合中的 `move-arm-by-ee`。


## 开发

使用 Python 3.12 与 `uv`：

```bash
uv sync --frozen --all-groups
uv run pytest -q
```

`src/forge_tool` 内置协议会自动解析，来源见 `THIRD_PARTY_NOTICES.md`。
可选独立可执行文件通过 `scripts/build_pyinstaller.sh` 构建。

## 许可证

本仓库使用 [Apache License 2.0](LICENSE)。运行时依赖不随源码仓库分发，
保留各自许可证，详见 [NOTICE](NOTICE) 与 [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md)。
发布独立可执行文件前审查完整打包制品。
