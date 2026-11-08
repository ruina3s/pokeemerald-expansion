# 02 用 GitHub Actions 线上编译（详细流程）

> 结论先行：
> **1）你仓库里已经有 CI（`.github/workflows/build.yml`），但它只"验证能编译"，不产出任何可下载的 ROM。**
> **2）你的 fork 目前 `Actions` 是关闭状态——API 显示 workflow 运行记录为 0 条，所以现在推代码不会触发任何构建。**
> **3）本项目约定：所有编译一律走线上。`build-rom.yml` 已就位——只在 `master` 上自动编译，分支改动靠网页手动触发；产出 `.gba/.elf/.map/.sym`。**
> **4）本机只保留 mGBA，用来跑下载回来的 ROM。全文逐行注释，可直接改。**
> **5）游戏内调试菜单默认就是开的**（`dev` / `debug` 模式都有），只有 `release` 会关掉——详见 4.3 节。
>
> 依据：`https://api.github.com/repos/ruina3s/pokeemerald-expansion/actions/runs` 返回 `total_count: 0`；`build.yml` 全文无 `upload-artifact`。

## 1. 为什么这个项目适合线上编译

| 因素 | 说明 |
|---|---|
| 仓库是 **public** | GitHub 对公开仓库的 Actions 分钟数**不限额**，用不完也不用担心账单 |
| 不需要原版 ROM | 现代构建（modern build）不依赖 `baserom.gba`；CI 里 `COMPARE=0` 即可。`make compare` 才需要原版 ROM 的 sha1 |
| 依赖干净 | 只需要 apt 装的交叉工具链 + Python3 + libpng，全是标准包 |
| 工具链可缓存 | `tools/` 下 14 个二进制用 `actions/cache` 缓存，key 按工具源码哈希 |
| 本机零配置 | 你这台是 Windows，本地编译要装 WSL2 + 交叉工具链（1–2 GB），且跨文件系统访问再慢 10 倍。CI 直接给一台 Ubuntu，**本机只需 Git + VS Code + mGBA** |
| 结果可复现 | 每次都是干净检出、固定镜像，不会出现"我这能编你那不能编" |

GitHub 官方文档：<https://docs.github.com/actions/using-github-hosted-runners/about-github-hosted-runners>

## 2. 现状盘点：仓库里的 4 个 workflow

### 2.1 `build.yml`（CI 主流程）

```yaml
on:
  push:
    branches: [master, upcoming]
  pull_request:

env:
  GAME_REVISION: 0
  GAME_LANGUAGE: ENGLISH
  COMPARE: 0          # 不比对原版 ROM（CI 没有 baserom.gba）
  UNUSED_ERROR: 1     # 未使用变量当错误
  DEPRECATED_ERROR: 1 # 弃用 API 当错误
```

8 个 job：

| job | 做什么 | 命令 |
|---|---|---|
| `build-emerald` | 编译绿宝石 ROM | `make -j$(nproc) -O all`（env `GAME_VERSION=EMERALD`） |
| `build-firered` | 编译火红 | `make firered -j$(nproc) -O` |
| `build-leafgreen` | 编译叶绿 | `make leafgreen -j$(nproc) -O` |
| `release` | Release 构建（LTO + NDEBUG） | `make -j$(nproc) release` |
| `test` | 跑测试套件 | `make -j$(nproc) check`（env `TEST=1`） |
| `build` | **门禁 job**，`needs` 前面 5 个，全过才绿 | `echo` 占位 |
| `docs_validate` | 校验 `docs/**/*.md` 是否都登记在 `SUMMARY.md` | `.github/docs_validate/inclusive_summary.py` |
| `allcontributors` | 机器人提交直接放行 | `exit 0` |

5 个编译 job 各自重复一遍 "apt 装依赖 + 恢复 tool 缓存"。这是**冗余但稳妥**的写法：任一 job 失败都能独立看出原因。代价是每个 job 都要 checkout 一次 500MB+ 的仓库。

### 2.2 `docs.yml`

`master` push 时用 mdbook 构建 `docs/` 并部署到 GitHub Pages。**没开 Pages 时这一步会失败**（脚本用 `octokit/request-action` 查 Pages 状态做了 `continue-on-error` 保护，但仍会红一次）。

### 2.3 `labels.yml`

PR 必须有至少一个标签才允许合并（`mheap/github-action-required-labels`）。

### 2.4 `build-rom.yml`（本项目新增，唯一能拿到 ROM 的入口）

`CI` 只回答"能不能编过"，不给文件。`build-rom.yml` 负责"编出来 + 上传 artifact"：

- **触发**：`master` push、`v*` tag、网页手动
- **输入**：版本（emerald/firered/leafgreen）、构建模式（dev/debug/release）、全量重编、跑测试、生成符号表
- **产物**：`rom-<版本>-<模式>-<ref>.zip`，里面是 `.gba` / `.elf` / `.map`（可选 `.sym`）
- **附带**：推 tag 时自动发 Release

全文与逐行解释见第 4 节。

## 3. 第一步：把 Actions 打开（不做这步，后面全是白费）

fork 出来的仓库默认不会自动跑 workflow，必须手动确认一次：

1. 打开 `https://github.com/ruina3s/pokeemerald-expansion/actions`
2. 顶部会有一条黄色横幅：*"Workflows aren't being run on this forked repository"*
3. 点 **"I understand my workflows, go ahead and enable them"**
4. 刷新页面，左侧应出现 `CI`、`Docs`、`Labels`、`Build ROM` 四个 workflow

如果左侧列表是空的，去 `Settings → Actions → General`，确认 **Actions permissions** 选的是 *"Allow all actions and reusable workflows"*，然后 `Save`。

另外建议（可选）：`Settings → Actions → General → Workflow permissions` 勾选 **Read and write permissions**——后面用 tag 自动发 Release 时需要写权限。

## 4. 第二步：`build-rom.yml`（核心，仓库里已就位）

文件：`.github/workflows/build-rom.yml`（**已经建好了，这份是它的当前内容**；改的时候直接改文件，再把下面这段同步过来）。

设计前提：**只在 `master` 上自动编译**，分支上的改动靠网页手动触发，避免每个 `feature/*` push 都烧一次 8–12 分钟。`build_mode` 一个下拉框同时管住了"优化级别"和"调试菜单开关"——因为它们本来就是同一个 `RELEASE` 宏决定的（见 4.3）。

```yaml
# ============================================================================
# Build ROM —— 本项目的唯一编译入口（不做本地编译，全部在 GitHub 上构建）
# 触发：push 到 master / 推 v* tag / 网页手动 Run workflow
# 产物：运行页最下方 Artifacts（含 .gba / .elf / .map，可选 .sym）
# ============================================================================
name: Build ROM

on:
  push:
    branches: [master]        # 只在 master 上自动编译；分支开发用网页手动触发
    tags: ['v*']              # 推 v* tag 也编译，编完自动发 Release（见 job: release）
    paths-ignore:             # 只改文档就不编译，省一次 8-20 分钟
      - '**.md'
      - 'docs/**'
      - 'PROJECT_DOCS_ZH/**'
  workflow_dispatch:          # 网页手动触发：可选版本、构建模式、全量/测试/符号表
    inputs:
      version:
        description: '游戏版本'
        type: choice
        options: [emerald, firered, leafgreen]
        default: emerald
      build_mode:
        description: 'dev=带调试菜单 / debug=带调试菜单+调试信息 / release=关闭调试菜单'
        type: choice
        options: [dev, debug, release]
        default: dev
      clean_build:
        description: '先 make tidy 再全量重编（改了 include/config/*.h 必须勾）'
        type: boolean
        default: false
      run_tests:
        description: '编译后跑 make check 测试套件（仅 emerald，首次约 +20 分钟）'
        type: boolean
        default: false
      make_syms:
        description: '额外产出 .sym 符号表（mGBA 调试器显示函数名）'
        type: boolean
        default: false

# 同一个 ref 连续触发时，取消上一次还在跑的构建，只保留最新一次
concurrency:
  group: build-rom-${{ github.ref }}
  cancel-in-progress: true

env:
  COMPARE: 0            # 必须 0：设 1 需要 baserom.gba（版权问题，CI 里也没有）
  UNUSED_ERROR: 0       # 0 = 未使用变量只警告；1 = 当错误（-Werror）
  DEPRECATED_ERROR: 0   # 0 = 弃用 API 只警告；同上

jobs:
  build:
    name: Build (${{ inputs.version || 'emerald' }} / ${{ inputs.build_mode || 'dev' }})
    runs-on: ubuntu-latest
    timeout-minutes: 90     # 兜底：正常 8-22 分钟，超 90 分钟一定是卡住了
    # inputs 只在 job 级 env 里能用，放工作流级 env 会拿不到值
    env:
      BUILD: ${{ inputs.version || 'emerald' }}       # Makefile 读 BUILD 决定编哪一版
      BUILD_MODE: ${{ inputs.build_mode || 'dev' }}   # push 触发时默认 dev
    outputs:
      artifact: ${{ steps.meta.outputs.name }}   # 传给下面 release job
    steps:
      # ---- 1. 取代码 ------------------------------------------------------
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 1     # 只取最新提交，省 30-60 秒（check_history.sh 要求 .git 存在）

      # ---- 2. 装交叉工具链 ------------------------------------------------
      - name: Install toolchain
        run: |
          sudo apt-get update
          sudo apt-get install -y --no-install-recommends \
            build-essential \
            binutils-arm-none-eabi \
            gcc-arm-none-eabi \
            libnewlib-arm-none-eabi \
            libpng-dev \
            python3

      # ---- 3. 缓存构建工具 ------------------------------------------------
      # tools/ 下 14 个二进制每次重编要 1.5-2.5 分钟，缓存后接近 0
      # key 带 tools 源码哈希：工具代码一改 key 就变，自动重编，不会误用旧缓存
      - name: Cache build tools
        uses: actions/cache@v4
        with:
          path: |
            tools/bin2c/bin2c
            tools/gbafix/gbafix
            tools/gbagfx/gbagfx
            tools/jsonproc/jsonproc
            tools/mapjson/mapjson
            tools/mid2agb/mid2agb
            tools/preproc/preproc
            tools/ramscrgen/ramscrgen
            tools/rsfont/rsfont
            tools/scaninc/scaninc
            tools/trainerproc/trainerproc
            tools/compresSmol/compresSmol
            tools/compresSmol/compresSmolTilemap
            tools/wav2agb/wav2agb
          key: tools-${{ runner.os }}-${{ hashFiles('tools/*/Makefile', 'tools/**/*.c', 'tools/**/*.cpp', 'tools/**/*.h') }}

      # 跑测试要额外两个工具（mGBA 无头 + patchelf），单独缓存，普通构建不背这个包袱
      - name: Cache test tools
        if: inputs.run_tests
        uses: actions/cache@v4
        with:
          path: |
            tools/patchelf/patchelf
            tools/mgba-rom-test-hydra/mgba-rom-test-hydra
          key: tools-check-${{ runner.os }}-${{ hashFiles('tools/**/Makefile', 'tools/**/*.c', 'tools/**/*.cpp', 'tools/**/*.h') }}

      # ---- 4. 可选全量重编 ------------------------------------------------
      # 改了 include/config/*.h、合并了上游大版本、或增量构建结果诡异时必须清
      - name: Clean
        if: inputs.clean_build
        run: make tidy

      # ---- 5. 编译 --------------------------------------------------------
      # dev     → 不加参数：-O2，RELEASE 未定义 → 调试菜单全部可用
      # debug   → DEBUG=1 ：-Og -g，同样带调试菜单，崩溃时有函数名+行号
      # release → RELEASE=1：LTO + NDEBUG，调试菜单被 DISABLED_ON_RELEASE 关掉
      - name: Build ROM
        run: |
          set -eo pipefail
          FLAGS=()
          case "$BUILD_MODE" in
            debug)   FLAGS+=("DEBUG=1") ;;
            release) FLAGS+=("RELEASE=1") ;;
          esac
          echo "::notice::BUILD=$BUILD MODE=$BUILD_MODE"
          make -j"$(nproc)" -O all "${FLAGS[@]}"

      # ---- 6. 可选：测试套件 ----------------------------------------------
      - name: Run tests
        if: inputs.run_tests && (inputs.version || 'emerald') == 'emerald'
        run: make -j"$(nproc)" -O check

      # ---- 7. 可选：符号表 ------------------------------------------------
      - name: Symbols
        if: inputs.make_syms
        run: make syms

      # ---- 8. artifact 名 -------------------------------------------------
      # ref 名里可能有 /（分支）或 v1.0.0（tag），拼进 artifact 名
      - name: Compute artifact name
        id: meta
        run: |
          REF="$(echo '${{ github.ref_name }}' | tr '/' '-')"
          echo "name=rom-$BUILD-$BUILD_MODE-$REF" >> "$GITHUB_OUTPUT"

      # ---- 9. 上传产物 ----------------------------------------------------
      - name: Upload ROM
        uses: actions/upload-artifact@v4
        with:
          name: ${{ steps.meta.outputs.name }}
          path: |
            poke*.gba
            poke*.elf
            poke*.map
            poke*.sym
          if-no-files-found: error
          retention-days: 30      # 30 天后自动删除；要长期保存就发 Release

      # ---- 10. 构建摘要 ---------------------------------------------------
      - name: Summary
        if: always()
        run: |
          {
            echo "### 构建结果"
            echo ""
            echo "| 项 | 值 |"
            echo "|---|---|"
            echo "| 版本 | \`$BUILD\` |"
            echo "| 模式 | \`$BUILD_MODE\` |"
            echo "| 提交 | \`${{ github.sha }}\` |"
            echo "| artifact | \`${{ steps.meta.outputs.name }}\` |"
            echo ""
            ls -lh poke*.gba 2>/dev/null || echo "未生成 .gba"
          } >> "$GITHUB_STEP_SUMMARY"

  # 只有推 v* tag 时才跑：把 artifact 附到 Release，永久可下载
  release:
    if: startsWith(github.ref, 'refs/tags/v')
    needs: build
    runs-on: ubuntu-latest
    permissions:
      contents: write        # 不加这行无法创建 Release
    steps:
      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: ${{ needs.build.outputs.artifact }}
      - name: Publish release
        uses: softprops/action-gh-release@v2
        with:
          files: |
            poke*.gba
            poke*.map
          generate_release_notes: true
```


### 4.1 逐段解释（知道每行在干什么，出问题才改得动）

| 片段 | 作用 | 什么时候要改 |
|---|---|---|
| `on.push.branches: [master]` | **只有 `master`** push 才自动编译 | 想让某个长期分支也自动编，在这里加；不建议用 `'**'`，分支一多会互相挤占排队 |
| `on.push.tags: ['v*']` | 推 `v1.0.0` 这类 tag 也编译，并触发下面的 Release job | 不需要自动发 Release 就删掉 |
| `paths-ignore` | 只改 `*.md` / `docs/` / `PROJECT_DOCS_ZH/` 不触发 | 新增其他纯文档目录时往里加一条 |
| `workflow_dispatch.inputs` | 网页手动触发时的 **5 个开关**（版本 / 构建模式 / 全量 / 测试 / 符号表） | 想加参数（如只跑某个测试）在这里加 |
| `type: choice / boolean` | `choice` 出下拉框，`boolean` 出复选框 | — |
| `concurrency` | 连推 3 次只跑最后一次，前两次自动取消 | 想保留每次构建就删掉这段 |
| `timeout-minutes: 90` | 兜底超时，正常 8–22 分钟 | 项目继续膨胀可调大 |
| `fetch-depth: 1` | 只取最新提交，省 30–60 秒 | `check_history.sh` 需要 `.git` 存在，不能设 0 |
| `COMPARE: 0` | 关闭与原版 ROM 的 sha1 比对 | **永远别设 1**，除非把 `baserom.gba` 弄进 runner（版权问题，不要做） |
| `UNUSED_ERROR: 0` | C 里没用到的变量只警告 | 想和官方仓库一样严格就设 1（代价是别人的 PR 很容易红） |
| `DEPRECATED_ERROR: 0` | 弃用 API 只警告 | 同上 |
| apt 那几行 | 装交叉工具链 + `libpng-dev`（`gbagfx` 依赖）+ `python3`（mapjson/learnset 脚本依赖） | Ubuntu 镜像换版本时一般不用改 |
| `actions/cache@v4` | 缓存 `tools/` 编好的 14 个二进制 | 改了 `tools/` 源码 → key 变 → 自动重编，无需手动清缓存 |
| `BUILD: ${{ inputs.version || 'emerald' }}` | 用 `BUILD` 变量告诉 Makefile 编哪一版，等价于 `make firered` | 加新版本（如 `ruby`）时改这里 |
| `BUILD_MODE` | 三选一，决定下面 `FLAGS` 加什么 | — |
| `Cache test tools` | 只在 `run_tests = true` 时恢复 mGBA 无头模拟器缓存 | 测试工具源码改动后自动失效 |
| `Compute artifact name` | 把 ref 名（分支/tag）里的 `/` 换成 `-`，否则 artifact 名非法 | 命名规则变了再改 |
| `set -eo pipefail` | 任一命令失败立刻停，不把半成品传上去 | 不要加 `-u`：`FLAGS` 空数组在旧 bash 上会报未绑定 |
| `FLAGS=()` + `"${FLAGS[@]}"` | 空数组 = 不加参数；`debug` 加 `DEBUG=1`；`release` 加 `RELEASE=1` | — |
| `make tidy`（`clean_build`） | 清掉所有 `.o` 与生成文件，等于全量重来 | 改 `include/config/*.h`、合并上游大版本时必勾 |
| `make -j$(nproc) -O` | `-j` 并行；`-O` 让各任务日志按行输出（不然并行日志会糊成一团） | — |
| `RELEASE=1` | 等价于 `make release`：LTO + 定义 `RELEASE` → 头文件里 `NDEBUG` 生效 → 调试菜单全关 | 只有正式发布才用 |
| `DEBUG=1` | 等价于 `make debug`：`-Og -g`，编出的 ROM 带函数名与行号 | 要 gdb / 崩溃回溯时用 |
| `make check`（`run_tests`） | 跑 `test/` 下的 ROM 内测试 | 改战斗/机制逻辑时勾上；仅 emerald |
| `make syms`（`make_syms`） | 额外产出 `.sym`，mGBA 调试器能显示函数名 | 要断点调试时勾 |
| `upload-artifact@v4` | 把 ROM 存成可下载 artifact | 想直接发 Release 见第 5 节 |
| `GITHUB_STEP_SUMMARY` | 在运行页顶部生成结果表 | 想加构建耗时统计就改这里 |

### 4.2 产物在哪、叫什么

文件名由 `BUILD`（版本）+ `RELEASE`（是否发布版）两个变量拼出来，规则在 `Makefile` 第 29 / 97 / 100 行：

| 触发方式 | ROM 文件名 | 说明 |
|---|---|---|
| `build_mode = dev` | `pokeemerald.gba` | 默认产物，**调试菜单全开** |
| `build_mode = debug` | `pokeemerald.gba`（对象在 `build/emerald-debug/`） | 文件名不变，但 ROM 里带 `-Og -g` 的调试信息 |
| `build_mode = release` | `pokeemerald-release.gba` | 名字多一个 `-release`，LTO + NDEBUG，**调试菜单全关** |
| `version = firered` | `pokefirered.gba` / `pokefirered-release.gba` | — |
| `version = leafgreen` | `pokeleafgreen.gba` / `pokeleafgreen-release.gba` | — |
| `run_tests = true` | `pokeemerald-test.elf` / `-test-headless.elf` | 测试用，不是给玩家玩的 |

> **一眼判断拿到的是不是带调试的版本**：文件名里带 `-release` 就是关掉调试菜单的；不带就是开的。

勾选 `make_syms = true` 时额外产出 `pokeemerald.sym`（mGBA 调试器据此显示函数名而不是地址）。
注意：`.gba` 里的符号信息来自 `.elf`，所以 artifact 一定要把 `.elf` 一起带上。

### 4.3 构建模式与游戏内调试菜单（怎么开、怎么按）

**核心结论：不用"开启"，默认就开着。** 只有 `release` 模式会把它关掉。

#### 为什么默认是开的

`include/constants/global.h` 第 12–17 行：

```c
#ifdef RELEASE                        // 只有定义了这个宏（即 RELEASE=1）才走第一支
#define ENABLED_ON_RELEASE  TRUE
#define DISABLED_ON_RELEASE FALSE     // ← 发布版：DISABLED_ON_RELEASE 变成 FALSE = 关闭
#else
#define ENABLED_ON_RELEASE  FALSE
#define DISABLED_ON_RELEASE TRUE      // ← 普通构建：DISABLED_ON_RELEASE 变成 TRUE = 开启
#endif
```

而 `include/config/debug.h` 里三个调试菜单全部写成 `DISABLED_ON_RELEASE`：

```c
#define DEBUG_OVERWORLD_MENU            DISABLED_ON_RELEASE  // 大地图调试菜单
#define DEBUG_BATTLE_MENU               DISABLED_ON_RELEASE  // 战斗内调试菜单
#define DEBUG_POKEMON_SPRITE_VISUALIZER DISABLED_ON_RELEASE  // 宝可梦图像预览
```

所以：**`dev` / `debug` 一定带菜单，`release` 一定没有**。CI 里 `build_mode` 默认 `dev`，你从 Artifacts 下载到的就是带菜单的版本。

#### 三种模式对照

| `build_mode` | CI 实际执行 | 优化 | 调试菜单 | 什么时候用 |
|---|---|---|---|---|
| `dev`（默认） | `make -j -O all` | `-O2` | ✅ 三个全开 | 日常：测剧情、给宝可梦、跳地图、验证数值 |
| `debug` | `make -j -O all DEBUG=1` | `-Og -g` | ✅ 三个全开 | 崩溃要看函数名/行号，或接 gdb / mGBA 调试器 |
| `release` | `make -j -O all RELEASE=1` | LTO + `NDEBUG` | ❌ 全关 | 只给最终发布；平时千万别用 |

> `make debug` 与 `DEBUG=1` 等价，`make release` 与 `RELEASE=1` 等价。CI 里用变量而不是 `make debug`，是因为 `make debug` 只能单独作为唯一目标（`Makefile` 第 55 行是 `ifeq (debug,$(MAKECMDGOALS))`），没法跟 `firered` 组合。

#### 三个菜单分别怎么按

| 菜单 | 打开方式 | 出现位置 |
|---|---|---|
| 大地图调试菜单 | **按住 `R`，再按 `START`**（`DEBUG_OVERWORLD_HELD_KEYS` + `DEBUG_OVERWORLD_TRIGGER_EVENT`） | 大地图上（不在菜单/对话/战斗中） |
| 战斗调试菜单 | 战斗中按 **`SELECT`** | 战斗指令选择界面 |
| 宝可梦图像预览 | 查看宝可梦能力页时按 **`SELECT`** | 队伍 → 宝可梦 → 能力页 |

> **mGBA 里 R 是哪个键？** mGBA 默认不把 `R` 绑在键盘 `R` 上。打开 `Tools → Settings → Input`，看 `R` 那一栏绑的是哪个键（常见是 `S`），或者直接改成你顺手的键。找不到就先按 `START` 开菜单确认键位没反。

#### 菜单里有什么（`src/debug.c` 的真实结构）

一级菜单 11 项，全部已汉化（这个 fork 用 `python_tools/translate_debug.py` 翻过）：

| 一级项 | 实际子项（摘录） | 干什么用 |
|---|---|---|
| 常用功能… | 飞往地图 / 瞬移到地图 / 设定天气 / 字体测试 / 时间功能 / 观看通关动画 / 开始作弊 / 树果相关 / EWRAM计数 / NPC跟随 / 测试宝可梦随机 / 测试道具随机 / 满充教程 / 大吾双打 | `飞往地图` 是跳图神器；`开始作弊` 一键给满资源 |
| 电脑/包包… | 使用电脑 / 填充… / 清空背包 / 清空寄放盒子 | 「填充…」能一键塞满背包、电脑、精灵球、招式学习器、树果、重要道具 |
| 同行宝可梦… | 回忆招式 / 孵化蛋 / 回复 / 编辑宝可梦 / 确认努力值 / 确认个体值 / 给予/清除病毒 / 清空 / 设置 / 开始Debug对战 | 「编辑宝可梦」里还能改异常状态、隐藏特性、亲密度、特性 |
| 获取… | 获得道具 / 获得宝可梦(简易) / 获得宝可梦(详细) / 获得蛋 / 获得装饰 / 最大零花钱 / 最大代币 / 最大对战点数 / 培育屋获得蛋 | 「详细」版可以指定等级、性格、个体、招式、异色、太晶属性 |
| 主人公… | 主人公名称 / 切换性别 / 刷新训练家ID | 没有改金钱——金钱走上面的「最大零花钱」 |
| 脚本… | 脚本1 ~ 脚本8 | 执行 `data/scripts/debug.inc` 里的 `Debug_EventScript_Script_1`…`_8`，**你写测试脚本就挂这里** |
| 训练家… | 从地图选择训练家 / 训练家1、2 / 搭档 / 双打开关 / 再战次数 / 再战准备完毕 / 尝试进入战斗 / 充能对战搜寻器 | 不用跑地图就能打任意训练家 |
| 遇敌… | 大量出现…（固定/动态、物种、地图、等级、招式、概率、剩余天数） | 测野生分布、测大量出现机制 |
| 标志与变量… | 设定某个标志 / 设定某个变量 / 图鉴标志全部设定·重置 / 各种开关 | 不止能改 flag，下面这些"作弊开关"极其好用（见下） |
| 声音… | 音效… / 音乐… | 试听任意 BGM / SE |
| ROM信息… | SaveBlock空间 / ROM空间 / Expansion版本 | 查存档与 ROM 容量，改数据结构后必看 |

「标志与变量…」里最值钱的几个开关（**测试时能省掉大量走图时间**）：

| 开关 | 效果 |
|---|---|
| 无视地形阻碍 | 穿墙，直接走到还没做完的地图区域 |
| 不再遭遇宝可梦 | 专心测 NPC / 剧情，不被野怪打断 |
| 对训练家隐身 | 反过来：测剧情时不被训练家视线拦住 |
| 无法捕捉宝可梦 | 测"必须打死"的剧情战 |
| 全部徽章 / 通关游戏 / 开拓通行证 | 直接解锁对应进度 |
| 飞翔标志 | 解锁飞行点 |

#### 想改按键 / 想放进开始菜单

改 `include/config/debug.h`：

```c
// 组合键：按住 L + R 再按 START（多个键用 | 连接，定义在 include/gba/io_reg.h）
#define DEBUG_OVERWORLD_HELD_KEYS       (L_BUTTON | R_BUTTON)

// 或者干脆不要组合键：在开始菜单里加一项（图鉴上方），触屏玩家友好
#define DEBUG_OVERWORLD_IN_MENU         TRUE
```

> 触发键 `DEBUG_OVERWORLD_TRIGGER_EVENT` 只能填 `include/field_control_avatar.h` 里那几个 `pressedXxxButton` 位域（`pressedStartButton` / `pressedAButton` / …）。**别用 `pressedSelectButton`**——大地图上 SELECT 已经被"注册道具"占用了，会打架。

改完推 `master`（或手动触发），等 CI 出 ROM。**这类改动改的是 `include/config/*.h`，勾上 `clean_build = true` 更稳**——配置宏被几百个 `.c` 包含，增量编译容易编出"改了一半"的行为。

#### printf 日志（`debug` 模式附带）

`include/config/general.h` 里 `LOG_HANDLER` 默认是 `LOG_HANDLER_MGBA_PRINT`，且只在 `NDEBUG` 未定义时生效。所以 `dev` / `debug` 模式下，C 代码里的 `DebugPrintf(...)` 会输出到 **mGBA 的 `Tools → View Logs`**；`release` 模式下全部编译掉，零开销。改逻辑时配合日志比猜快得多。

### 4.4 下载步骤

1. `https://github.com/ruina3s/pokeemerald-expansion/actions`
2. 左侧点 **Build ROM** → 点最新一次运行（标题是你 commit message 的前半句）
3. 页面底部 **Artifacts** 区域 → 点 `rom-emerald-dev-master`（格式：`rom-<版本>-<模式>-<ref>`）
4. 下载得到 zip，解压即得 `.gba` / `.elf` / `.map`（需要**登录 GitHub**；未登录看不到下载按钮）
5. artifact 默认保留 30 天（`retention-days` 控制的），过期就没了
6. 运行页顶部还有一张 **Summary** 表，写着本次编的是哪个版本、哪个 commit、`.gba` 多大

## 5. 打 tag 自动发 Release（长期可下载）

`build-rom.yml` 里已经带了 `release` job，**不用再追加任何 YAML**，打个 tag 推上去就行：

```bash
git tag v1.17.3-zh.1          # 必须以小写 v 开头，才会命中 tags: ['v*']
git push origin v1.17.3-zh.1  # 推 tag 会同时触发 build 和 release
```

两个 job 的分工：

1. `build` 先跑完整构建（`on.push.tags` 命中），产物名形如 `rom-emerald-dev-v1.17.3-zh.1`
2. `release` 用 `needs: build` + `needs.build.outputs.artifact` 把**同一个** artifact 取下来，挂到 Release 上（不重复编译，省 8–20 分钟）

三个必须知道的点：

- `permissions: contents: write` 不能省，否则创建 Release 会被 403 拒掉
- artifact 名只在 `build` 里算一次，通过 `outputs` 传给 `release`；两边各写一遍命名规则，早晚会不一致
- **tag 构建默认还是 `dev` 模式**（tag 触发时 `inputs` 全为空，`BUILD_MODE` 回落到 `dev`）。要发正式版，先手动触发选 `build_mode = release` 验证一遍再打 tag

> 想让 tag 构建自动切到 `release` 模式，把 env 里那行改成：
> `BUILD_MODE: ${{ inputs.build_mode || (startsWith(github.ref, 'refs/tags/') && 'release' || 'dev') }}`

> ⚠️ **版权提醒**：这是 ROM 改版工程。公开发布改版 ROM 属于灰色地带，官方仓库（pret / RHH）自己从不发布 `.gba`。建议把仓库保持为 **private**，或者只发布补丁（xdelta/ips patch）。这一点没得商量。
## 6. 常见失败与对策

| 现象 | 原因 | 修法 |
|---|---|---|
| push 后 Actions 页面什么都没发生 | fork 未启用 workflow / 只改了 md（被 `paths-ignore` 跳过） | 见第 3 节；确认改动不是纯文档 |
| `Process completed with exit code 2` 在 `make` 阶段 | `-Werror` 把警告当错误（常见：未使用变量、弃用 API） | 把 `UNUSED_ERROR` / `DEPRECATED_ERROR` 设为 0 |
| `unknown character` / `charmap` 相关报错 | 文本里有 `charmap.txt` 未收录的字符（中文特有坑） | 在 `charmap.txt` 补该字，或换字 |
| `make check` 红，但 ROM 能编出来 | 测试套件失败（改机制后常见） | 看日志里哪个 `TEST(...)` 失败；本地无法复现就临时不勾 `run_tests` |
| `No space left on device` | 仓库大 + 并行编译产物占空间 | 把 `-j"$(nproc)"` 改成 `-j2`，或只编 emerald |
| 卡在 `Install toolchain` 超时 | 网络波动 / apt 镜像慢 | 重跑一次；连续失败改用 `runs-on: ubuntu-22.04` 固定镜像 |
| `sha1sum: rom.sha1: No such file` 或校验失败 | 有人把 `COMPARE` 设成 1 | 保持 `COMPARE: 0` |
| Artifacts 里没有文件 / job 红在 Upload | 编译其实失败了，只是报错被日志淹没 | 往上翻，找第一个 `error:` |
| 改了 `config/*.h` 但游戏行为没变 | 忘了勾 `clean_build` | 手动 Run workflow 勾上重跑 |
| 进游戏按 R+START 没反应 | 编的是 `release`（文件名带 `-release`），菜单被 `DISABLED_ON_RELEASE` 关掉了 | 重编选 `build_mode = dev` |
| 菜单出现了但按不出来 | 不是在"大地图"状态（在菜单/对话/战斗中），或 mGBA 的 R 键映射不对 | 先按 START 确认键位；`Tools → Settings → Input` 改 R |
| 改了 `include/config/debug.h` 没生效 | 同配置宏的坑，增量编译不彻底 | 勾 `clean_build` |
| `Docs` workflow 红 | 未开启 GitHub Pages | `Settings → Pages → Source: GitHub Actions`，或直接删掉 `docs.yml` |
| `Build ROM` 一直排队不动 | 同时有多个运行，`concurrency` 在排队/取消 | 看是否有更晚的 commit 已取消它；或等额度释放 |

## 7. 性能与成本参考

| 项 | 数值 |
|---|---|
| 首次全量（无 tool 缓存 + `clean_build`） | 约 15–22 分钟（编 14 个工具 + 1.1 万张 PNG 转 4bpp + 385 个 C 文件） |
| 有 tool 缓存（最常见情况） | 约 8–12 分钟（主要耗时在资源转换，单线程，无法缓存） |
| 改 `include/config/*.h` + `clean_build` | 约 15–22 分钟 |
| 改一张 PNG / 一句对话 | 仍是 8–12 分钟——**CI 每次都是干净检出，没有本地增量那 10 秒的待遇** |
| 额外勾 `run_tests = true` | 首次 +15–20 分钟（要现编 mGBA 无头模拟器），之后有缓存 +3–6 分钟 |
| 选 `build_mode = debug` | 和 `dev` 差不多，`-Og` 反而编得更快，但 ROM 体积更大 |
| 公开仓库计费 | **免费，不占额度** |
| 私有仓库计费 | 占 2000 分钟/月（Free 计划），按 Ubuntu 2 vCPU 计 |

> 关键认知：**线上编译的耗时几乎与你改了多少无关**，固定成本就在 8 分钟左右（checkout + 装依赖 + 编工具 + 资源转换 + 编 385 个 C）。
> 所以正确用法是「**攒一批改动 → 一次 push → 一次验证**」，而不是"改一行看一次"。

想更快只有三条路：

1. 收窄触发范围，避免无意义的构建（`paths-ignore` 已默认开启）。
2. 缓存 `build/`（**不推荐**——缓存体积上 GB、`.map_version` 与资源生成容易不一致，反而制造奇怪的增量错误）。
3. 上 self-hosted runner（自己的机器，但那就变成本地编译了，与"全部线上"的约定冲突）。

## 8. 线上编译的边界：哪些必须回到本机

CI 只能给你"编译是否通过"和"一个 ROM 文件"，**不能替你看画面、不能替你按手柄**。所以完整闭环是「**云端编译 + 本地运行**」：

| 环节 | 在哪做 | 具体做法 |
|---|---|---|
| 写代码/脚本/数据 | 本机 VS Code（或 GitHub 网页） | 编辑 `data/`、`src/`、`include/` |
| 编译 | **GitHub Actions** | push 即触发，或网页 Run workflow |
| 拿 ROM | 本机浏览器 | Actions → Artifacts → 下载 zip |
| 验证玩法 | **本机 mGBA** | 打开 `.gba`，标题按 SELECT 用 Quickstart 直接进游戏；大地图按住 R 再按 START 开调试菜单 |
| 断点调试 | 本机 mGBA + `.elf`/`.sym` | 需要 `build_mode = debug` + `make_syms = true` |
| 跑单元测试 | GitHub Actions | 手动触发时勾 `run_tests` |
| 批量改文本（汉化） | 本机 Python | `python_tools/translate_*.py`，只改文本不编译 |
| 版本迁移 | 本机 Python | `migration_scripts/<ver>/*.py`，跑完 push 让 CI 验证 |

**明确做不到的三件事**（不要浪费时间尝试）：

1. 在 CI 里"玩"游戏——runner 无显示输出，ROM 只能被测试框架无头驱动。
2. 在 CI 里用 mGBA 的 GUI 调试器——只能跑无头测试。
3. 在 CI 里做 `make compare`——需要 `baserom.gba`，版权与体积都不允许。

## 9. 完整验证清单

- [ ] Actions 已启用（页面左侧能看到 `CI`、`Docs`、`Labels`、`Build ROM`）
- [ ] `build-rom.yml` 已在 `master` 上
- [ ] 推一次 `master`，`Build ROM` 自动跑起来且变绿
- [ ] Artifacts 里能下载到 `rom-emerald-dev-master.zip`
- [ ] 解压出的 `pokeemerald.gba` 能在 mGBA 里启动
- [ ] 大地图按住 R 再按 START，**调试菜单能弹出**（弹出的中文菜单说明你拿到的是带调试的版本）
- [ ] 手动 Run workflow 试一次 `clean_build = true`，确认全量也能过
- [ ] （可选）`Settings → Pages` 已开，`Docs` workflow 变绿
- [ ] （可选）打 tag 能自动发 Release

---

### 参考链接

- GitHub Actions 官方文档：<https://docs.github.com/actions>
- `upload-artifact@v4`：<https://github.com/actions/upload-artifact>
- `actions/cache`：<https://github.com/actions/cache>
- 项目自带 CI：`https://github.com/ruina3s/pokeemerald-expansion/blob/master/.github/workflows/build.yml`
- 上游中文安装说明：<https://github.com/rh-hideout-chinese/pokeemerald-expansion/blob/master/INSTALL_ZH.md>
