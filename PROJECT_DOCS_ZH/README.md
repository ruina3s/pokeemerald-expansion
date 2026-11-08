# 项目文档索引（PROJECT_DOCS_ZH）

本目录是 **你自己项目的本地说明文档**，与上游 `docs/`（mdbook 文档站）**刻意分开**。

> ⚠️ **不要把这些文件搬进 `docs/`**。
> 仓库的 CI 有一个校验 job：`.github/workflows/build.yml` → `docs_validate` 会跑
> `.github/docs_validate/inclusive_summary.py`，规则是「`docs/**/*.md` 必须全部登记在 `docs/SUMMARY.md`」。
> 放进去而没登记 = CI 直接失败。

## 编译口径（先看这一条）

**所有编译一律走 GitHub Actions，本机不装 WSL、不装 `arm-none-eabi`、不执行 `make`。**

```
改文件（VS Code / Porymap / Aseprite）
   ↓
git push（master 自动编；分支要手动 Run workflow）
   ↓
GitHub Actions: Build ROM（8–12 分钟）
   ↓
Artifacts 下载 rom-emerald-dev-<ref>.zip
   ↓
本机 mGBA 打开 .gba 验证
```

本机只装 4 样：**Git、VS Code、mGBA、（按需）Porymap / Aseprite**。
配置方式见 `02-GitHub-Actions线上编译.md`，日常流程见 `04-修改项目工作流.md`。

## 四份文档

| 文件 | 内容 | 什么时候看 |
|---|---|---|
| [01-项目结构与文件说明.md](./01-项目结构与文件说明.md) | 完整文件树、每个目录/关键文件的职责、编译流水线（CI 视角）、"我要改 X 去哪"速查表 | 接手项目第一天；忘了某个东西在哪时 |
| [02-GitHub-Actions线上编译.md](./02-GitHub-Actions线上编译.md) | 仓库 4 个 workflow 逐段解析、启用 Actions、**逐行注释的 `build-rom.yml`**（master 触发 + 5 个开关）、**三种构建模式与游戏内调试菜单的开启方法**、tag 发 Release、失败排查、耗时与计费、线上/本机职责边界 | 第一次配置线上编译时；想知道调试菜单怎么开时；改了 workflow 要排错时 |
| [03-脚本功能与语法手册.md](./03-脚本功能与语法手册.md) | 5 类脚本体系：构建/工具脚本、事件脚本（401 宏 + 11 个带注释 demo）、战斗脚本（424 宏 + 逐行讲解）、队伍脚本、JSON 与配置宏；含注释符号规则与能力边界 | 写剧情、写 NPC、写战斗、批量改文本时 |
| [04-修改项目工作流.md](./04-修改项目工作流.md) | 零工具链环境、9 步线上工作流、**调试菜单造测试条件**、12 个场景操作手册、CI 耗时矩阵、线上能力对照表、删除内容四步法、16 个坑、端到端示例 | 每次真要动手改东西时 |

## 项目速览

- 仓库：<https://github.com/ruina3s/pokeemerald-expansion>（fork of <https://github.com/rh-hideout-chinese/pokeemerald-expansion>）
- 版本：pokeemerald-expansion **1.17.3**（`include/constants/expansion.h`）
- 产物：`pokeemerald.gba`（`dev`/`debug`，**带调试菜单**）/ `pokeemerald-release.gba`（`release`，**无调试菜单**）—— 均由 CI 产出
- 规模：939 张地图、803 个布局、10,937 张 PNG、530 首 MIDI、385 个 `.c`、968 个测试文件

## 四条最省时间的经验

1. **先判层级再动手**：配置（`include/config/`）→ 数据（`src/data/`）→ 脚本（`data/`）→ 资源（`graphics/`、`sound/`）→ 引擎（`src/*.c`），顺序越靠前越安全。
2. **生成文件不手改**：`trainers.h`、`wild_encounters.h`、`map_groups.h`、`*.4bpp`、`*.lz`、`*.smol` 全是编译产物，改输入（`.party` / `.json` / `.png`）。
3. **攒够改动再 push**：线上编译固定成本 8–12 分钟，改一行和改一百行耗时差不多。别为一个错别字等一次构建。
4. **改 `include/config/*.h` 要勾全量**：动作在 Actions 页面手动触发时勾 `clean_build`，否则容易出现"改了宏但行为没变"。

## 注释符号速记（写脚本时最容易踩）

| 文件 | 注释 | 备注 |
|---|---|---|
| `.inc` / `.s` | `@` | 事件脚本、战斗脚本、移动脚本都用它；`#` 不是注释 |
| `.c` / `.h` | `//`、`/* */` | 标准 C |
| `.party` | `/* */` | **`//` 会导致编译失败** |
| `.json` | 无 | 加注释会让 `mapjson` / `jsonproc` 解析失败 |
