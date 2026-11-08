# 02 用 GitHub Actions 线上编译（详细流程）

> 结论先行：
> **1）你仓库里已经有 CI（`.github/workflows/build.yml`），但它只"验证能编译"，不产出任何可下载的 ROM。**
> **2）你的 fork 目前 `Actions` 是关闭状态——API 显示 workflow 运行记录为 0 条，所以现在推代码不会触发任何构建。**
> **3）要"线上编译并下载 ROM"，最小改动是新增一个 workflow：编译 `EMERALD` + `upload-artifact` 上传 `pokeemerald.gba`。全文可直接抄的 YAML 在本文第 4 节。**
>
> 依据：`https://api.github.com/repos/ruina3s/pokeemerald-expansion/actions/runs` 返回 `total_count: 0`；`build.yml` 全文无 `upload-artifact`。

## 1. 为什么这个项目适合线上编译

| 因素 | 说明 |
|---|---|
| 仓库是 **public** | GitHub 对公开仓库的 Actions 分钟数**不限额**，用不完也不用担心账单 |
| 不需要原版 ROM | 现代构建（modern build）不依赖 `baserom.gba`；CI 里 `COMPARE=0` 即可。`make compare` 才需要原版 ROM 的 sha1 |
| 依赖干净 | 只需要 apt 装的交叉工具链 + Python3 + libpng，全是标准包 |
| 工具链可缓存 | `tools/` 下 13 个二进制用 `actions/cache` 缓存，key 按工具源码哈希 |
| Windows 本地编译慢 | 你这台是 Windows（Git Bash），本地全量编译要装 WSL2；CI 直接给你一台现成 Ubuntu，这是最省事的路 |

GitHub 官方文档：<https://docs.github.com/actions/using-github-hosted-runners/about-github-hosted-runners>

## 2. 现状盘点：仓库里已有的 3 个 workflow

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

## 3. 第一步：把 Actions 打开（不做这步，后面全是白费）

fork 出来的仓库默认不会自动跑 workflow，必须手动确认一次：

1. 打开 `https://github.com/ruina3s/pokeemerald-expansion/actions`
2. 顶部会有一条黄色横幅：*"Workflows aren't being run on this forked repository"*
3. 点 **"I understand my workflows, go ahead and enable them"**
4. 刷新页面，左侧应出现 `CI`、`Docs`、`Labels` 三个 workflow

如果左侧列表是空的，去 `Settings → Actions → General`，确认 **Actions permissions** 选的是 *"Allow all actions and reusable workflows"*，然后 `Save`。

另外建议（可选）：`Settings → Actions → General → Workflow permissions` 勾选 **Read and write permissions**——后面用 tag 自动发 Release 时需要写权限。

## 4. 第二步：加一个"能下载 ROM"的 workflow（核心）

新建文件：`.github/workflows/build-rom.yml`

```yaml
name: Build ROM

on:
  push:
    branches: [master]
    paths-ignore:            # 只改文档不触发编译，省时间
      - '**.md'
      - 'docs/**'
  workflow_dispatch:         # 支持网页上手动点"Run workflow"
    inputs:
      version:
        description: '要编译的版本 (emerald / firered / leafgreen)'
        required: true
        default: 'emerald'
      release_build:
        description: '是否用 Release 构建 (LTO + NDEBUG)'
        type: boolean
        default: false

concurrency:                 # 同一分支只保留最新一次构建，旧的自动取消
  group: build-${{ github.ref }}
  cancel-in-progress: true

env:
  COMPARE: 0                 # 必须为 0：CI 没有 baserom.gba
  UNUSED_ERROR: 0            # 0 = 未使用变量只警告不报错（CI 更宽容）
  DEPRECATED_ERROR: 0

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Install toolchain
        run: |
          sudo apt-get update
          sudo apt-get install -y --no-install-recommends \
            build-essential binutils-arm-none-eabi gcc-arm-none-eabi \
            libnewlib-arm-none-eabi libpng-dev python3

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

      - name: Build ROM
        run: |
          if [ "${{ inputs.release_build }}" = "true" ]; then
            make -j"$(nproc)" -O release
          elif [ "${{ inputs.version }}" = "firered" ]; then
            make firered -j"$(nproc)" -O
          elif [ "${{ inputs.version }}" = "leafgreen" ]; then
            make leafgreen -j"$(nproc)" -O
          else
            make -j"$(nproc)" -O all
          fi

      - name: Upload ROM
        uses: actions/upload-artifact@v4
        with:
          name: rom-${{ inputs.version || 'emerald' }}
          path: |
            poke*.gba
            poke*.map
            poke*.elf
          if-no-files-found: error
          retention-days: 30
```

### 4.1 逐段解释（知道每行在干什么，出问题才改得动）

| 片段 | 作用 | 什么时候要改 |
|---|---|---|
| `on.push.branches: [master]` | 只在 `master` 上自动编译 | 想每次 commit 都编就加 `paths-ignore` 之外的触发；想 PR 也编就加 `pull_request` |
| `workflow_dispatch.inputs` | 网页手动触发时给 3 个选项 | 想加参数（如 `TESTS` 过滤测试）在这里加 |
| `concurrency` | 连推 3 次只跑最后一次 | 想保留所有构建就删掉 |
| `COMPARE: 0` | 关闭与原版 ROM 的 sha1 比对 | **永远别设 1**，除非把 `baserom.gba` 弄进 runner（版权问题，不要做） |
| `UNUSED_ERROR: 0` | C 里没用到的变量只警告 | 想和官方仓库一样严格就设 1（代价是别人的 PR 很容易红） |
| apt 那一行 | 装 `arm-none-eabi` 交叉工具链 + `libpng-dev`（`gbagfx` 依赖）+ `python3`（mapjson/learnset 脚本依赖） | Ubuntu 镜像换版本时一般不用改 |
| `actions/cache@v4` | 缓存 `tools/` 编好的 14 个二进制 | 改了 `tools/` 源码 → key 变 → 自动重编，无需手动清缓存 |
| `make -j$(nproc) -O` | `-j` 并行；`-O` 让各任务日志按行输出（不然并行日志会糊成一团） | — |
| `make release` | 打开 `LTO`（`config.mk` 里 `USE_LTO_ON_RELEASE=1`）+ 定义 `NDEBUG` + 关闭调试特性 | Release ROM 更大、更快，但**无法用调试菜单** |
| `upload-artifact@v4` | 把 ROM 存成可下载 artifact | 想直接发 Release 见第 5 节 |

### 4.2 产物在哪、叫什么

| 构建方式 | ROM 文件名 | 说明 |
|---|---|---|
| `make` / `make all` | `pokeemerald.gba` | 默认（带调试功能） |
| `make release` | `pokeemerald-release.gba` | LTO + NDEBUG |
| `make firered` | `pokefirered.gba` | — |
| `make leafgreen` | `pokeleafgreen.gba` | — |
| `make debug` | `pokeemerald.gba`（对象在 `build/emerald-debug/`） | 带 `-Og -g` |
| `make check` | `pokeemerald-test.elf` / `-test-headless.elf` | 测试用，不是给玩家玩的 |

`make syms` 才会额外产出 `pokeemerald.sym`（如果你用 mGBA 调试器看符号表，就加上）。

### 4.3 下载步骤

1. `https://github.com/ruina3s/pokeemerald-expansion/actions`
2. 左侧点 **Build ROM** → 点最新一次运行
3. 页面底部 **Artifacts** 区域 → 点 `rom-emerald`
4. 下载得到 zip，解压即 `.gba`（需要**登录 GitHub**；未登录看不到下载按钮）
5. artifact 默认保留 30 天（`retention-days` 控制的），过期就没了

## 5. 进阶：打 tag 自动发 Release（长期可下载）

artifact 会过期，Release 不会。追加一个 job：

```yaml
  release:
    if: startsWith(github.ref, 'refs/tags/')
    needs: build
    runs-on: ubuntu-latest
    permissions:
      contents: write        # 必须：否则无法创建 Release
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: rom-emerald
      - uses: softprops/action-gh-release@v2
        with:
          files: |
            poke*.gba
            poke*.map
          generate_release_notes: true
```

用法：

```bash
git tag v1.17.3-zh.1
git push origin v1.17.3-zh.1
```

推送 tag 后，`Releases` 页面自动出现带 ROM 附件的版本。

> ⚠️ **版权提醒**：这是 ROM 改版工程。公开发布改版 ROM 属于灰色地带，官方仓库（pret / RHH）自己从不发布 `.gba`。建议把仓库保持为 **private**，或者只发布补丁（xdelta/ips patch）。这一点没得商量。

## 6. 常见失败与对策

| 现象 | 原因 | 修法 |
|---|---|---|
| push 后 Actions 页面什么都没发生 | fork 未启用 workflow | 见第 3 节 |
| `Error: Process completed with exit code 2` 在 `make` 阶段 | `-Werror` 把警告当错误（常见：未使用变量、弃用 API） | 把 `UNUSED_ERROR` / `DEPRECATED_ERROR` 设为 0 |
| `unknown character` / `charmap` 相关报错 | 文本里有 `charmap.txt` 未收录的字符（中文特有坑） | 在 `charmap.txt` 补该字，或换字 |
| `make check` 红，但 ROM 能编出来 | 测试套件失败（改机制后常见） | 本地复现：`make check TESTS="TestXxx"` |
| `Docs` workflow 红 | 未开启 GitHub Pages | `Settings → Pages → Source: GitHub Actions`，或直接删掉 `docs.yml` |
| `No space left on device` | 仓库大 + 并行编译产物占空间 | 用 `make -j2`，或只编译 emerald |
| 编译超时（默认 6 小时） | 一般不会；除非开了 `ANALYZE=1` | 别在 CI 用 `ANALYZE=1` |
| `sha1sum: rom.sha1: No such file` 或校验失败 | 有人把 `COMPARE` 设成 1 | 保持 `COMPARE: 0` |

## 7. 性能与成本参考

| 项 | 数值 |
|---|---|
| 首次全量（无缓存） | 约 15–25 分钟（工具编译 + 约 1.1 万张 PNG 转 4bpp + 385 个 C 文件） |
| 有 tool 缓存后 | 约 10–18 分钟（主要耗时在资源转换，无法缓存） |
| 只改一个 `.c` | 约 1–3 分钟（依赖 `.d` 文件生效时） |
| 公开仓库计费 | 免费，不占额度 |
| 私有仓库计费 | 占 2000 分钟/月（Free 计划），按 Ubuntu 2 vCPU 计 |

想更快：把 `build/` 也缓存（**不推荐**——缓存体积大、`.map_version` 与资源生成容易不一致，反而制造奇怪的增量编译错误）；或者上 self-hosted runner。

## 8. 本地对照（用同样的命令验证 CI 行为）

在 WSL2 里：

```bash
sudo apt update
sudo apt install -y build-essential binutils-arm-none-eabi gcc-arm-none-eabi \
  libnewlib-arm-none-eabi libpng-dev python3

cd ~/pokeemerald-expansion
make -j"$(nproc)" -O all      # 等价于 CI 的 build-emerald
make -j"$(nproc)" check      # 等价于 CI 的 test（需要 tools/mgba 的预编译模拟器）
make release                 # 等价于 CI 的 release
```

Windows 原生（MSYS2）也能编，但比 WSL2 慢一个数量级，不建议。

## 9. 完整验证清单

- [ ] Actions 已启用（页面左侧能看到 workflow 列表）
- [ ] `build-rom.yml` 已推到 `master`
- [ ] 手动 Run workflow → 选 `emerald` → 成功
- [ ] Artifacts 里能下载到 `rom-emerald.zip`
- [ ] 解压出的 `pokeemerald.gba` 能在 mGBA 里启动
- [ ] （可选）`Settings → Pages` 已开，`Docs` workflow 变绿

---

### 参考链接

- GitHub Actions 官方文档：<https://docs.github.com/actions>
- `upload-artifact@v4`：<https://github.com/actions/upload-artifact>
- `actions/cache`：<https://github.com/actions/cache>
- 项目自带 CI：`https://github.com/ruina3s/pokeemerald-expansion/blob/master/.github/workflows/build.yml`
- 上游中文安装说明：<https://github.com/rh-hideout-chinese/pokeemerald-expansion/blob/master/INSTALL_ZH.md>
