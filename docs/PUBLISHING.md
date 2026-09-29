# 发布到模组站（Modrinth / CurseForge）指导

> 适用：本仓库（Tinkers' Construct 的 Fabric 移植）以及同组织的 `MinecraftReconstruction/Mantle-Fabric`。
> 本文回答的是「怎么把 GitHub 仓库挂到模组站、怎么让发版自动化」。
> **本文不代替 [ATTRIBUTION.md](../ATTRIBUTION.md)**：项目页、CHANGELOG、版本说明都受其中
> 「非官方 / AI 生成 / 未经原作者审核」表述的约束，发布前请一并读完第 1 节。
>
> 核对时间：2026-09-29。规则引用自 Modrinth Content Rules（2026-08-13 版）与 Minotaur 2.x 文档；
> 模组站界面改版频繁，字段名以站内实际界面为准。

## 0. 先澄清一件事：模组站没有「绑定仓库」这个开关

Modrinth（以及 CurseForge）**不提供**「把 GitHub 仓库连上去、之后自动发版」的功能。
所谓把仓库「挂」到模组站，实际是三件互相独立的事：

| # | 做什么 | 在哪做 | 能不能自动化 |
|---|---|---|---|
| 1 | 建**项目页**，在 Links 里填 GitHub 仓库地址（源码 + issue） | 网站，一次性 | 手工 |
| 2 | 上传**版本**：jar + 版本号 + MC 版本 + loader + 依赖 + changelog | 网站或 API | 可以全自动 |
| 3 | 让 CI 在**打 tag** 时自动执行第 2 步 | GitHub Actions | 全自动 |

第 1 步才是字面意义的「挂仓库」；真正省事的是第 3 步。三种路线：

| 路线 | 需要改什么 | 适合 |
|---|---|---|
| 纯手工上传 | 什么都不用改 | 第一次发布、偶尔发一次 |
| **Minotaur**（Modrinth 官方 Gradle 插件） | `build.gradle` 加一段 + 一条 workflow | 本来就是 Gradle 构建的项目；**推荐** |
| **mc-publish**（社区 GitHub Action） | 只加一条 workflow，零 Gradle 改动 | 想一次同时发 Modrinth + CurseForge + GitHub Release |

---

## 1. 发布前的合规检查（本仓库尤其重要）

### 1.1 许可证：MIT，允许 fork 与再分发

上游 Tinkers' Construct / Mantle 都是 **MIT（SlimeKnights, 2022）**，允许 fork、修改、再分发，
条件是**保留版权声明与许可证全文**。本仓库已经做到：`LICENSE` 原样保留，`ATTRIBUTION.md` 标明来源。

但 MIT 只管**代码**。项目名、图标、以及「这是谁的项目」归模组站的内容规则管，见下。

### 1.2 Modrinth 内容规则里与本仓库直接相关的三条

（2026-08-13 版《Content Rules》，原文 <https://modrinth.com/legal/rules>）

- **§4 版权与转载**：不允许「从别处直接转载」，但 **license-abiding fork 不受此限**；
  上传非自己原创的内容必须**明确署名每一处原始来源**。
  → 我们是 MIT fork，合规前提是把 SlimeKnights 与 AlphaMode 在项目页写明（可直接改写 `ATTRIBUTION.md`）。
- **§6 生成式 AI**：这一条里其实有**两条独立规则**，别混成一条看：
  - §6.1 **披露义务**：当「代码的实质部分由 AI 产出」时，必须勾选站内的
    **Contains AI-generated content** 披露项。本仓库的移植增量（`mcr/*` 相对上游的 diff）
    确实是 AI 在人类指导下生成/改写的（`ATTRIBUTION.md` 自己就是这么写的），
    **所以这一项应当勾上**——它是如实披露，不是自贬。
  - §6.2 **公开禁令**：只针对「内容**主要或完全**由 AI 产出」的项目。
    本仓库的主体（绝大多数逻辑代码）是 SlimeKnights 的原作加上 AlphaMode 的 Fabric 适配层，
    我们的改动是增量，**按代码构成不属于这一条**。
  - 结论：**披露照勾，公开照发**；要能对审核说清「哪一段是谁写的、我们改了什么」——
    这正是 `docs/STATUS.md` 的分支地图与 `git diff <上游基点>..mcr/*` 的用处。
- **§2.2 / §5 描述与元数据**：项目描述**必须有英文版**（除非只面向特定语言）；
  依赖必须填进每个版本的 Dependencies 区；标题里不要塞多余信息。

### 1.3 不能冒充官方

官方项目在 Modrinth 上是 [`tinkers-construct`](https://modrinth.com/mod/tinkers-construct)
（实测 `loaders = ["forge", "neoforge"]`，**没有 Fabric 版**），作者 KnightMiner / SlimeKnights。
所以 Fabric 移植确实是一块空缺，但：

- 项目名不要就叫 `Tinkers' Construct`，slug 不要蹭官方；
  推荐 `Tinkers' Construct: Fabric (Unofficial Port)` + slug `tinkers-construct-fabric` 之类；
- 图标不要直接用官方图标；
- 描述里必须写明：非官方、由 AI 辅助生成、**未经 SlimeKnights / AlphaMode 审核**、
  出问题找我们不要去找原作者（措辞直接沿用 `ATTRIBUTION.md`）；
- 项目页和 README 都要给出上游/官方入口，让想用稳定版的人能走回去。

### 1.4 CurseForge 的政策不一样

CurseForge 对「未经原作者同意的二次分发」历来更严，同一个 jar 不一定能照搬过去。
要发 CurseForge，先单独确认上游对该平台的态度，别默认「Modrinth 能过 = CurseForge 也能过」。

---

## 2. 手工建项目（第一次必须手工做）

1. 注册 <https://modrinth.com> 并验证邮箱（建议开 2FA）；头像菜单 → **Projects** → **Create a project**。
2. 填表（关键字段）：

   | 字段 | 怎么填 |
   |---|---|
   | Name | `Tinkers' Construct: Fabric (Unofficial Port)` —— 见 1.3 |
   | Slug / URL | 例如 `tinkers-construct-fabric` |
   | Summary | 一句话，**不要带 Markdown 格式**，别重复项目名 |
   | Description | 支持 Markdown，**必须有英文版**；把 `ATTRIBUTION.md` 的声明改写进去 |
   | Project type | `Mod` |
   | License | `MIT` |
   | Loaders | `Fabric` |
   | **Links → Source code** | `https://github.com/MinecraftReconstruction/TinkersConstruct` |
   | **Links → Issues** | `.../issues` |
   | Environment | Client **and** server（本模组两端都要） |
   | Content disclosures | 勾上 **Contains AI-generated content**（见 1.2） |

3. 保存后项目默认是 **draft**（站内不可见）。填完提交审核，状态会经过 `processing`，
   **变成 `approved` 之后才对公众可见**。审核期间可以先把版本备好。
4. 上传第一个版本：项目页 → **Versions** → **Create version**：

   - 文件：`build/libs/` 里 **remap 过的那个 jar**（见 4.2，别传 `-dev` / `-sources`）；
   - Version number / Name：见第 6 节的 tag 约定；
   - Release channel：移植版先选 **Alpha** 或 **Beta**，不要一上来就 Release；
   - Game versions：`1.20.1`；Loaders：`Fabric`；
   - Dependencies：`Fabric API`(required)、`Architectury API`(required)、
     **Mantle（我们自己的 Mantle-Fabric 项目）**(required) —— 见第 7 节；
   - Changelog：允许 Markdown，写清「同步到上游哪个版本 + 已知差异 + 指向 docs/」。

5. 审核通过后，把最稳的版本设为 Featured，并给 README 加 Modrinth 链接与徽章。

---

## 3. 拿 API token

自动化靠一个**个人访问令牌**，不是账号密码：

1. <https://modrinth.com/settings/account> → **Personal access tokens** → 新建；
2. 勾权限：**`CREATE_VERSION`**（`modrinth` 发版任务必需）、
   `PROJECT_WRITE`（只有要同步项目简介 `modrinthSyncBody` 时才需要）；
3. 复制（**只显示一次**，丢了只能重建）；
4. GitHub 仓库 → Settings → Secrets and variables → Actions → New repository secret：
   名字 `MODRINTH_TOKEN`，值为上面那串；
5. 本地手动试跑：`MODRINTH_TOKEN=xxx ./gradlew modrinth`。

**不要把 token 写进任何会被提交的文件。** 泄露了立刻回站上撤销重建。

---

## 4. 路线 A：Minotaur（Modrinth 官方插件，推荐）

### 4.1 接上插件

本仓库 `settings.gradle` 已经有 `pluginManagement { repositories { gradlePluginPortal() } }`，
所以只要动 `build.gradle`：

```groovy
plugins {
    id 'fabric-loom' version '1.12-SNAPSHOT'
    id 'maven-publish'
    id 'com.modrinth.minotaur' version '2.+'   // ← 新增
}

modrinth {
    token = System.getenv('MODRINTH_TOKEN')       // 默认就读这个环境变量，写出来更直观
    projectId = 'xxxxxxxx'                        // 项目建好后取页面 URL 里的 slug 或 ID
    versionNumber = System.getenv('MODRINTH_VERSION') ?: version
    versionName = "Tinkers' Construct Fabric ${mod_version} for Minecraft ${minecraft_version}"
    versionType = 'alpha'                         // 移植版先 alpha/beta，别直接 release
    uploadFile = remapJar                         // ⚠️ 用 Loom 必须填 remapJar，不是 jar
    gameVersions = [minecraft_version]            // 不填也会自动检测，显式写更稳
    loaders = ['fabric']
    changelog = System.getenv('MODRINTH_CHANGELOG') ?: 'See the GitHub release notes for this tag.'
    dependencies {
        required.project 'fabric-api'
        required.project 'architectury-api'
        required.project 'porting_lib'            // 注意 slug 是下划线，不是连字符
        // 我们自己的 Mantle Fabric 项目建好后，再加一行：
        // required.project 'mantle-fabric'
    }
    debugMode = System.getenv('MODRINTH_TOKEN') == null   // 本地无 token 时只打印不上传
}
```

常用任务：

| 命令 | 作用 |
|---|---|
| `./gradlew modrinth` | 上传一个新版本 |
| `./gradlew modrinthSyncBody` | 把 `syncBodyFrom` 的内容同步成项目简介（**不可撤销**） |

### 4.2 两个必踩的坑

- **Loom 必须用 `remapJar`**：Loom 产出的 `jar` 是 intermediary（dev）命名的，玩家装了会加载失败。
  症状很好认：文件能下载、进游戏报错 / 模组列表里异常。
- **别把 `-dev` / `-sources` 传上去**：`build/libs/` 里通常有 2–4 个 jar，只有主 artifact 该上传。

### 4.3 tag 触发的 workflow

新建 `.github/workflows/modrinth.yml`（**不要动现有 `build.yml`**，它负责往 maven 发快照）：

```yaml
name: modrinth

on:
  push:
    tags: [ 'v*' ]

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'

      - name: make gradle wrapper executable
        run: chmod +x ./gradlew

      - name: build
        run: ./gradlew build --no-daemon

      - name: publish to modrinth
        env:
          MODRINTH_TOKEN: ${{ secrets.MODRINTH_TOKEN }}
          MODRINTH_VERSION: ${{ github.ref_name }}      # 例如 v3.12.1-fabric.1
        run: ./gradlew modrinth --no-daemon
```

---

## 5. 路线 B：mc-publish（多平台一把梭）

不想改 `build.gradle`、或者要同时发 Modrinth + CurseForge + GitHub Release 时用这条。
它是社区 Action，不是 Modrinth 官方支持的工具（官方只支持 Minotaur），出问题得自己 debug。

```yaml
name: release

on:
  push:
    tags: [ 'v*' ]

permissions:
  contents: write          # GitHub Release 需要

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'

      - run: chmod +x ./gradlew
      - run: ./gradlew build --no-daemon

      - uses: Kira-NT/mc-publish@v3
        with:
          modrinth-id: xxxxxxxx                       # 项目 ID 或 slug
          modrinth-token: ${{ secrets.MODRINTH_TOKEN }}
          github-token: ${{ secrets.GITHUB_TOKEN }}

          files: 'build/libs/!(*-@(dev|sources|javadoc)).jar'
          version: ${{ github.ref_name }}
          version-type: alpha
          game-versions: 1.20.1
          loaders: fabric
          environment: 'client | server'
          changelog-file: changelog.txt               # 本仓库根目录已有这个文件
          dependencies: |
            fabric-api(required)
            architectury-api(required)
          retry-attempts: 2
          fail-mode: warn                             # 先 warn，摸熟了再改 fail
```

同样能加 `curseforge-id` + `curseforge-token` 把 CurseForge 一起发；
`github-token` 那行会让它顺便建 GitHub Release。

---

## 6. 版本号与 tag 约定（配合本仓库现有构建）

现状：`build.gradle` 里 `version = "${minecraft_version}-${artifact_version}"`，
而 `artifact_version = "${mod_version}.${buildNum}"`、CI 里 `buildNum = GITHUB_RUN_NUMBER`、
本地则退化成 `DEV.<sha>`。也就是说**直接让发版工具取 `version`，Modrinth 上的版本号会变成
`1.20.1-3.6.4.512` 这种带 CI 运行号、跟 tag 无关的字符串**，对使用者毫无意义。

建议：

- 用 **tag 作为唯一版本来源**，格式例如 `v3.6.4-fabric.1`（`fabric.N` 表示我们的第 N 次移植构建）；
- workflow 里把 tag 通过环境变量交给插件：`MODRINTH_VERSION: ${{ github.ref_name }}`
  （对应 4.1 里的 `versionNumber = System.getenv('MODRINTH_VERSION') ?: version`）；
  不想让版本号带 `v` 前缀的话，在那个 step 里用 shell 去掉：`MODRINTH_VERSION=${GITHUB_REF_NAME#v}`；
- 同时在 GitHub 建 Release，changelog 里链到本仓库的 `docs/STATUS.md` / 差异登记表。

**不要用 `ARTIFACT_VERSION` 环境变量来传版本号**：当前 `build.gradle` 那两行是
`artifact_version = "${system.getenv().ARTIFACT_VERSION}"`，`system` 是小写拼写错误
（应为 `System`），一设这个变量就会以
`Could not get unknown property 'system' for root project` 直接构建失败
（已实测复现；`Mantle-Fabric` 的 `build.gradle` 有同样的写法）。要修的话是把它改成
`System.getenv('ARTIFACT_VERSION')`，但发布这条链路不必依赖它。

---

## 7. 依赖怎么填（本仓库的实际情况）

Modrinth 会**同时看两层**：

1. **外置依赖**：`fabric.mod.json` 的 `depends` 里那些。
   本仓库 `modImplementation("slimeknights.mantle:Mantle:...")` 没有 `include(...)`，
   说明 **Mantle 是必须由用户另装的前置**（不是打进包里的）。
   → 走 Modrinth 发布时，必须有一个可指向的 Mantle 项目，否则玩家装上去会缺前置。
   `Mantle-Fabric` 是本次发布的**共同前置工程**，建议一起发（或先发它）。
2. **内嵌依赖（jar-in-jar）**：`include(...)` 打进去的，例如一排 `porting_lib_*` 模块、
   `mixinsquared`、`milk-lib`、`dripstone-fluid-lib`、night-config。
   Modrinth 一般能从 jar 内部识别并标成 **embedded**，发布后核对一遍即可。

填写时注意：

- slug 要准：Fabric API = `fabric-api`、Architectury API = `architectury-api`、
  Cardinal Components = `cardinal-components-api`、Porting Lib = **`porting_lib`**（下划线）。
- Porting Lib 是 LGPL：**不要**把它标成 `embedded` 后又对外声明"本模组完全 MIT"，
  项目页的许可说明要说清内嵌组件各自的许可（`ATTRIBUTION.md` 里已有清单）。
- 依赖区是**按版本**存的，不是按项目：以后新增/删除依赖要记得在新版本的 Dependencies 里改。

---

## 8. 发布顺序：要不要先把 Mantle 弄上去？

**短答：要发的如果是 3.12.1 那条线的移植版，是——先发 Mantle-Fabric。**
如果只是把现在默认分支 `1.20.1` 上的 3.6.4 构建发上去，技术上不用（它把 Mantle 内嵌了），
但那个版本跟 Alpha 已经在 Modrinth 上的 Hephaestus 是同一个东西，价值不大。

下面这些不是推测，是仓库现状 + 站上实测：

| 事实 | 出处 |
|---|---|
| 默认分支 `1.20.1` 把 Mantle **内嵌**进 jar：`modImplementation(include("slimeknights.mantle:Mantle:..."))`，`mantle_version=1.9.291` | `git show 1.20.1:build.gradle` |
| 工作分支 `mcr/upstream-3.12.1` 已改为**外置**依赖（去掉了 `include`），`mantle_version=1.11.DEV.ad2e7db0` | 该分支的 `build.gradle` / `gradle.properties` |
| Modrinth 上的项目 `mantle` 只有 `forge`/`neoforge`（最新 1.11.117），**没有 Fabric 版** | Modrinth API 实测 |
| 公开可下载的 Fabric 版 Mantle 只到 `1.20.1-1.9.296`（devos maven），而 3.12.1 要求 `[1.11.113,)` | `docs/STATUS.md` |
| Alpha 的 Hephaestus 在 Modrinth 上是**内嵌** Mantle：jar 内含 `META-INF/jars/Mantle-1.20.1-1.9.291.jar`，所以它的项目页不列 Mantle 依赖 | 下载发布包实测 |

也就是说：mcr 分支产出的 jar，现在**任何 Modrinth 用户都凑不齐前置**——
`mantle >= 1.11.DEV.*` 在公开渠道无处可下。所以这个布局下顺序只能是
**先发 Mantle-Fabric，再在 Tinkers 的版本里把 Mantle 标成 required 指向它**。

三条路线，选一条并写进 `docs/STATUS.md`：

| 路线 | 用户要装几个文件 | 代价 |
|---|---|---|
| **A. 外置**（mcr 分支现状）：先发 Mantle-Fabric | 2 个（Tinkers + Mantle） | 两个项目都要维护发版；好处是库能独立更新、依赖关系透明 |
| **B. 内嵌**（默认分支 / Alpha 现状）：只发 Tinkers 一个 jar | 1 个 | 不用建 Mantle 项目；但 Mantle 一更新就得重发 Tinkers，而且别的模组也内嵌 Mantle 时会 mod id 冲突 |
| C. 两条都发（内嵌版 + 外置版） | 1 或 2 个 | 覆盖面最广，维护成本最高 |

另外两件发之前得想清楚的事：

- **Alpha 的 `hephaestus` 页面还在线上**：<https://modrinth.com/mod/hephaestus>，28.8 万下载，
  最近一次 1.20.1 版本是 2025-11-15 的 `1.20.1-3.6.4.305`（beta）。
  你们要发的是它的**延续**（目标 3.12.1），而且 mod id 同为 `tconstruct`（外加 `mantle`）——
  **同一 MC 版本下两个项目不能共存**。建议发布前跟 Alpha 打个招呼，
  项目页写明「延续自 Hephaestus，AlphaMode 未参与、未审核、未背书」。
- **Mantle-Fabric 单独发时，它自己也要过一遍第 1 节**：独立项目页、独立源码链接、
  同样要写非官方声明与 AI 披露，别只在 Tinkers 那边写。

---

## 9. 排错清单

| 症状 | 原因 / 处理 |
|---|---|
| `Invalid Authentication Credentials` 或 401 | token 没勾 `CREATE_VERSION`，或 `MODRINTH_TOKEN` 没设进 secret/环境变量 |
| `Version number already exists` | Modrinth 同一项目下版本号唯一；改号或删旧版本再传 |
| 上传成功但进游戏加载失败 | 传了 dev 命名的 `jar`，应传 `remapJar` 的产物 |
| 版本号里带了 CI 运行号 | 见第 6 节，用 tag 覆盖 `versionNumber` |
| 项目页 404 / 搜不到 | 还在 draft 或审核中；确认已提交审核并 approved |
| Modrinth 列出的依赖跟预期不符 | 先看 `fabric.mod.json` 的 `depends`，再按第 7 节在版本里改 |
| `./gradlew modrinth` 在本地就上传了 | 想干跑就设 `debugMode = true`，它只打印载荷不上传 |
| `modrinthSyncBody` 把简介覆盖坏了 | 该任务**不可撤销**，先用 `debugMode` 看一遍；简介内容从 `syncBodyFrom` 来 |
| 玩家反馈「缺少前置 `mantle`」 | 见第 8 节：外置布局下必须先让 Mantle-Fabric 有公开下载源，或改回内嵌 |
| 页面内容被举报/下架 | 优先回看 1.2 的 AI 披露与 1.3 的非官方声明是否齐全 |

---

## 10. 本仓库的落地顺序建议

1. **先定政策**：1.2 的 AI 条款和 `ATTRIBUTION.md` 的口径过一遍，确认"能不能公开、以什么措辞公开"。
2. **先发 Mantle-Fabric**（Tinkers 的前置），再发本仓库；两者在站上互指。理由与三条路线见第 8 节。
3. **手工发一个 alpha 验证闭环**：干净实例 + 只装必要前置，能进世界、能开一次合成界面。
4. **再上自动化**：`modrinth` 块 + `modrinth.yml` + `MODRINTH_TOKEN`，用 tag 触发。
5. 想要 CurseForge 再考虑换/加 mc-publish，别一上来就三平台同时开。
6. 每次发布都带上：上游署名 + 非官方声明 + AI 披露 + 已知差异指针（`docs/BEHAVIOUR-DIFFERENCES.md`）。

### 上线前 checklist

- [ ] 项目名 / slug / 图标不含官方元素，描述有**英文版**且写明非官方、AI 生成、未经原作者审核
- [ ] Content disclosures 勾了 **Contains AI-generated content**
- [ ] Links 里 Source code / Issues 指向本仓库
- [ ] 版本：正确 jar、正确 MC 版本与 loader、依赖（Mantle / Fabric API / Architectury）填齐
- [ ] 版本通道是 alpha/beta，changelog 写清同步到哪个上游版本
- [ ] token 只存在于 GitHub Secrets，本地试跑用环境变量，没进 git 历史
- [ ] 现有 `build.yml`（maven 的 snapshot 流程）没被改动
