# 项目文档索引（PROJECT_DOCS_ZH）

本目录是 **你自己项目的本地说明文档**，与上游 `docs/`（mdbook 文档站）**刻意分开**。

> ⚠️ **不要把这些文件搬进 `docs/`**。
> 仓库的 CI 有一个校验 job：`.github/workflows/build.yml` → `docs_validate` 会跑
> `.github/docs_validate/inclusive_summary.py`，规则是「`docs/**/*.md` 必须全部登记在 `docs/SUMMARY.md`」。
> 放进去而没登记 = CI 直接失败。

## 四份文档

| 文件 | 内容 | 什么时候看 |
|---|---|---|
| [01-项目结构与文件说明.md](./01-项目结构与文件说明.md) | 完整文件树、每个目录/关键文件的职责、编译流水线、"我要改 X 去哪"速查表 | 接手项目第一天；忘了某个东西在哪时 |
| [02-GitHub-Actions线上编译.md](./02-GitHub-Actions线上编译.md) | 现有 3 个 workflow 逐段解析、启用 Actions、可直接抄的"编译并下载 ROM"YAML、发 Release、失败排查、耗时与计费 | 想在网页上编译、不想在本机装 WSL 时 |
| [03-脚本功能与语法手册.md](./03-脚本功能与语法手册.md) | 5 类脚本体系：构建/工具脚本、事件脚本（401 宏）、战斗脚本（424 宏）、队伍脚本、JSON 与配置宏；每类都给 demo 与能力边界 | 写剧情、写 NPC、写战斗、批量改文本时 |
| [04-修改项目工作流.md](./04-修改项目工作流.md) | 软件清单、WSL 环境、7 步流程、12 个场景操作手册、变更影响矩阵、删除内容的四步法、踩坑清单、端到端示例 | 每次真要动手改东西时 |

## 项目速览

- 仓库：<https://github.com/ruina3s/pokeemerald-expansion>（fork of <https://github.com/rh-hideout-chinese/pokeemerald-expansion>）
- 版本：pokeemerald-expansion **1.17.3**（`include/constants/expansion.h`）
- 产物：`pokeemerald.gba`（`make`）/ `pokeemerald-release.gba`（`make release`）
- 规模：939 张地图、803 个布局、10,937 张 PNG、530 首 MIDI、385 个 `.c`、968 个测试文件

## 三条最省时间的经验

1. **先判层级再动手**：配置（`include/config/`）→ 数据（`src/data/`）→ 脚本（`data/`）→ 资源（`graphics/`、`sound/`）→ 引擎（`src/*.c`），顺序越靠前越安全、编译越快。
2. **生成文件不手改**：`trainers.h`、`wild_encounters.h`、`map_groups.h`、`*.4bpp`、`*.lz`、`*.smol` 全是编译产物，改输入（`.party` / `.json` / `.png`）。
3. **改 `include/config/*.h` 必须 `make clean && make`**：它被几百个编译单元包含，增量编译会给你假象。
