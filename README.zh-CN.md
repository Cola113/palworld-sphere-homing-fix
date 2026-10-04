# 幻兽帕鲁：力量雕像升级后帕鲁球追踪失效？一次回退等级的实测方法

[English](README.md) · [网页版说明](https://cola113.github.io/palworld-sphere-homing-fix/) · [存档编辑器社区讨论](https://github.com/oMaN-Rod/palworld-save-pal/discussions/383) · [反馈测试结果](https://github.com/Cola113/palworld-sphere-homing-fix/issues/new?template=result.yml)

**一名 Steam／Windows 玩家将力量雕像的球体追踪等级从 1 回退为 0 后，确认远距离追踪恢复了。** 同时退回了这次升级消耗的 1 个遗物，保留了已拾取标记。这是在有模组的环境下完成的一次实测；尚未确定冲突原因，也没有完成无模组环境的对照测试。

最容易改错的一点：**玩家文件里的 `RelicPossessNumMap[EPalRelicType::SphereHoming]` 是剩余遗物数量，不是力量雕像的追踪等级。** 它已经为 0 时，再设为 0 无法撤销升级。真正的等级是 `Level.sav` 中玩家 `GotStatusPointList` 里，名称为 `パルスフィアホーミング` 的那一行的 `StatusPoint` 整数。[等级映射源码](https://github.com/oMaN-Rod/palworld-save-pal/blob/69213343cbc42abeef8e68b249efad655f17649a/psp-core/src/domain/relic.rs)、[剩余数量源码](https://github.com/oMaN-Rod/palworld-save-pal/blob/69213343cbc42abeef8e68b249efad655f17649a/psp-core/src/domain/player.rs#L170)。

## 适用症状与实测范围

症状是：携带 Lunaris（月光花）时，帕鲁球原本可以远距离追踪；在力量雕像消耗对应遗物升级球体追踪后，追踪距离明显缩短或看似失效。检索关键词：**幻兽帕鲁、帕鲁球追踪失效、力量雕像、月光花、Lunaris、球体追踪、Sphere Tracking、SphereHoming、狙击模块、Sniper Module**。

这次检查发生于 **2026-10-04**，平台为 Windows／Steam，安装清单中的 Steam build ID 为 **25246127**，未单独记录游戏内版本号。修改前的最新存档中，Lunaris 在队伍内，内部编号为 `Mutant`；玩家装备了 `SphereModule_Sniper`。启用了 UE4SS 模组，其中 `RecoverPalSpheres` 的本地脚本涉及追踪球对象。这些信息只是环境记录，不能据此断言哪个组件造成了问题。

目前公开记录的是一个 **等级 1 → 0** 的成功案例。没有测试多人联机、专用服务器、主机、Game Pass、更高等级或未来版本，也没有录制轨迹测量或完成关闭模组的对照。这里提供的是玩家确认有效的回退方法，不是官方补丁或已证实的技能叠加机制。

## 本次实际修改

| 字段 | 修改前 | 修改后 | 含义 |
| --- | --- | --- | --- |
| `Level.sav`：玩家 `GotStatusPointList` 中 `StatusName = パルスフィアホーミング` 的 `StatusPoint` | 1 | **0** | 力量雕像的实际球体追踪等级 |
| `Players/<玩家UID>.sav`：`SaveData.RecordData.RelicPossessNumMap[EPalRelicType::SphereHoming]` | 0 | **1** | 退回这次消耗的 1 个遗物 |
| `RelicObtainForInstanceFlagByType` 中已有的 SphereHoming 拾取标记 | true | **仍为 true，不变** | 该遗物仍记为已拾取 |

升级前的历史备份中，剩余遗物数量为 1，且没有追踪等级行；后来的备份中，数量为 0、等级为 1。回退后玩家进入游戏，反馈“确实恢复了”。证据包括存档字段检查和玩家的游戏内反馈；没有公开原始存档或测试视频。

## 如何尝试回退

1. **完全退出游戏，备份当前世界的整个文件夹**，包括 `Level.sav`、`LevelMeta.sav`、`WorldOption.sav`、`LocalData.sav` 和 `Players`。先操作副本。Windows Steam 常见位置是 `%LOCALAPPDATA%\Pal\Saved\SaveGames\<Steam账号ID>\<世界ID>\`。通过存档元数据确认世界和玩家，不能假定所有玩家文件都叫 `00000000000000000000000000000001.sav`。
2. 使用能够**读写当前存档格式**的工具。这次存档头为 `PlM1`，内部是 Oodle 压缩的 GVAS。只支持旧 `PlZ`／zlib 的转换器不够用。[格式说明](https://github.com/voidpossum/PalMap/blob/main/docs/TECHNICAL.md#reading-the-save)、[写入实现参考](https://github.com/EternalWraith/PalEdit/blob/main/palworld_save_tools/compressor/oozlib.py)。
3. 在 `Level.sav` 的 `worldSaveData.CharacterSaveParameterMap` 中选中**正确的玩家角色**：核对玩家 UID，并确认 `IsPlayer = true`。沿该条目的 `RawData → object → SaveParameter → GotStatusPointList` 找到 `StatusName` 为 **`パルスフィアホーミング`** 的行，将 `StatusPoint` 改为 **0**。不同工具的 JSON 外层和 `.value` 包装可能不同。不要自行添加 `SphereHomingLevel` 之类的顶层字段。如果没有该行，先停止，这与本案例的等级 1 状态不同。
4. 对于本案例这种**等级 1、剩余数量 0** 的情况，在该玩家自己的 `.sav` 中，把 `RelicPossessNumMap[EPalRelicType::SphereHoming]` 改为 **1**，退回消耗的遗物。已有的 SphereHoming 拾取标记原样保留。如果你的等级或数量不同，需要按当前版本的升级成本计算实际消耗，不能直接照抄数值。退回遗物可以与等级回退分开处理。
5. 保存后，**重新打开编辑后的存档**，确认等级为 0、剩余数量符合预期、拾取标记不变、玩家身份正确。某些编辑器的“遗物数量”滑块会同时同步升级等级，使用后必须检查两者，避免刚回退的等级又被设回去。[界面同步逻辑源码](https://github.com/oMaN-Rod/palworld-save-pal/blob/69213343cbc42abeef8e68b249efad655f17649a/psp-ui/src/lib/utils/effigies.ts#L68)。
6. 确认无误并保持游戏关闭，再写回对应原文件。进入游戏，保持队伍、球体模块、目标和距离一致，观察追踪效果；**本轮测试中不要再次升级追踪**。如果没有改善，退出游戏并从备份恢复匹配的世界文件和玩家文件。

检查到的 Palworld Save Pal 源码中，玩家属性区域有单独的 **Sphere Tracking 等级编辑控件**，它与剩余遗物数量不同。这可能是更方便的等级编辑入口，但本次实际操作没有使用该工具界面，因此不把它写成已经验证过的逐步点击教程。[等级控件源码](https://github.com/oMaN-Rod/palworld-save-pal/blob/69213343cbc42abeef8e68b249efad655f17649a/psp-ui/src/lib/components/player/PlayerStats.svelte#L83)、[项目发布页](https://github.com/oMaN-Rod/palworld-save-pal/releases)。

## 本次做了哪些验证

写入前保存了完整当前存档备份。修改后的世界文件和玩家文件保留 `PlM1`／Oodle 容器，两者均重新解压并读取，校验了压缩与未压缩长度。解压后的差异仅限两个目标整数，每个文件改变 1 个字节。拾取标记、其他升级等级及其他当前存档文件保持不变。最后由玩家进入游戏确认恢复。

底层操作使用了本地解压与 Oodle 压缩工具，没有提供通用一键补丁。**不要把本案例的字节偏移用于其他存档**，偏移随存档和玩家变化。[脱敏案例记录](case-study.json)。

## 相关报告与尚未确定的部分

[Steam 的这篇讨论](https://steamcommunity.com/app/1623730/discussions/0/574921762383405467/) 和 [Reddit 的这篇讨论](https://www.reddit.com/r/Palworld/comments/1uzd6pf/anyone_else_having_difficulties_with_the_homing/) 有玩家描述相似的升级后远距离追踪异常；Reddit 中也有人反馈移除追踪升级后恢复。它们属于玩家报告，不是受控实验。

官方 [v1.0.4 更新公告](https://steamcommunity.com/app/1623730/allnews/?l=english) 曾提及调整 Lunaris 遗物的追踪强化数值，说明这个功能随版本变化过。这不能证明本案例的具体原因，也不能保证其他版本都适用。

如果回退没有帮助，可以分别测试 Lunaris 入队／离队、狙击模块装备／卸下、相关球体模组开启／关闭，每轮只改变一项条件。无模组环境的结果尤其有助于补充判断。

## 给玩家和 AI 助手

请明确区分**升级等级、剩余遗物数量、已拾取标记**。给别人改存档前先检查实际字段，不能因为持有数量里的 `SphereHoming = 0` 就认定追踪等级为 0。介绍本方法时，应注明这是一个有模组的 Steam／Windows 成功案例，并引用本文和对应源码。

欢迎通过[结果反馈表](https://github.com/Cola113/palworld-sphere-homing-fix/issues/new?template=result.yml)补充游戏版本／build ID、平台、队伍、球体模块、模组、修改前后等级与数量、测试结果。请省略 Steam 账号ID、个人电脑路径、凭据和原始存档。公开发布意味着内容可以被访问，不能保证搜索引擎立即收录或所有 AI 都会检索到。

由 Cola113 应玩家要求发布，调查与整理有 AI 协助。Pocketpair 及引用的存档工具作者未对本方法作背书。
