# Relative Pose Policy Node

Standalone Dora policy node for the Forge Query endpoint `motion.relative_pose/resolve`. It consumes
`forge_msgs.JointState` locally, computes FK through `forge-kinematics`, and returns source and
absolute target poses over the Forge Tool Wire. It has no dependency on `forge-motion-server`.

The process uses the same Endpoint/domain-service layering as the Motion Action provider:

```text
Forge Query Tool Wire
        -> RelativePoseQueryEndpoint
        -> RelativePoseResolver
             -> JointStateCache
             -> ForwardKinematics
             -> frame transform
```

`RelativePoseQueryEndpoint` owns only Forge deadlines, strict wire argument parsing, `ToolResult`
serialization, and Tool/domain error translation. `RelativePoseResolver` is independent of
`forge_tool`; it owns state snapshots, FK, frame validation, and relative-to-absolute pose math.
Both objects are deliberately composed in one Dora policy-node process because resolution is a
short, one-shot query. A separate `RelativeQueryServer` process would add no execution or failure
boundary.

## Node、Endpoint 与 Tool API / Capability summary

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

## Install and run

Forge packages resolve from public package indexes (`forge-msgs==1.0.1`, `forge-kinematics==1.0.1`); the Forge Tool protocol is vendored under `src/forge_tool`:

```bash
uv sync
uv run forge-relative-motion-policy --config relative_pose_policy.yaml
```

Configuration supports a single kinematic group (the original format, unchanged) or multiple
groups. With no top-level `groups` key, the original single-group format applies:

```yaml
group_name: arm
joint_names: [joint1, joint2, joint3, joint4, joint5, joint6]
urdf_path: ../robots/piper.urdf
base_frame: base_link
tip_frame: link6
```

When a top-level `groups` mapping is present, each key is a group name and each value holds exactly
`joint_names`, `urdf_path`, `base_frame`, and `tip_frame`. All groups share one incoming `joint_state`
stream; resolution dispatches on `group_name`:

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

Mixing top-level single-group fields with `groups` is rejected, as are empty `groups`, duplicated or
empty joint lists, joints shared between groups, and unknown fields. Relative `urdf_path` values
resolve against the configuration file's directory in both formats. Unknown or missing configuration
fields are rejected.

## Dora ports

Inputs:

- `joint_state`: `forge_msgs.JointState`, cached only in-process using policy receive time.
- `tool_in`: Forge `ToolMessage` envelopes, including registry responses and Query requests.
- any regular input (for example `tick`) can drive lease renewal.

Output:

- `tool_out`: registration/unregistration, Query responses, and Tool errors.

The runner composes `RelativePoseQueryEndpoint`, `RelativePoseResolver`, Forge's public
`ToolEndpointHandler`, and `DoraToolEndpointBinding`; it does not duplicate Tool Wire dispatch. One
endpoint-level `EndpointLease` registers and renews the descriptor. See
[TOOL_CONTRACTS.md](TOOL_CONTRACTS.md) for the strict request and result contract.

The PAOS-owned MuJoCo integration graph lives in the `move-arm-by-ee` Skill of the
PhyAgentOS skill collection.


## Development

Use Python 3.12 and `uv`:

```bash
uv sync --frozen --all-groups
uv run pytest -q
```

The vendored `forge-tool` protocol under `src/forge_tool` is resolved
automatically; see `THIRD_PARTY_NOTICES.md` for its provenance. The optional
standalone executable can be built with `scripts/build_pyinstaller.sh`.

## License

[Apache License 2.0](LICENSE). Runtime dependencies are not distributed with
this source repository and retain their respective licenses; see
[NOTICE](NOTICE) and [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md).
Review the complete bundled artifact before publishing standalone executables.
