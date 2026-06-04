# BL Vela SDK

芯片原厂（Bouffalo Lab）维护的、基于 openvela 的长期演进 SDK。

> **一句话定位**：本仓库 = **集成清单 + CI + 发版入口**，不放驱动源码。
> 它用 repo manifest 把「openvela 基座 + BL 驱动适配层 + 复用驱动仓」三者钉版冻结，
> 让任何人都能 100% 复现某个 SDK 版本。

---

## 1. 三层治理模型

| 层 | 内容 | Owner | 载体 |
|---|---|---|---|
| OS 基座 | openvela | 小米 Vela | github/open-vela（**trunk-5.5 tag 为同步点**） |
| **BL Vela SDK** | 基座 + BL 芯片移植/驱动/板级 + 复用驱动 | **Bouffalo Lab（本仓）** | 本仓 + vendor/bl 系列仓 |
| 产品 | SDK + 业务代码 | 下游产品团队 | 产品内部仓（拉分支作为产品 commit 原点） |

**职责边界（已与各方约定）**
- **基线来源**：以 github/open-vela 的 `trunk-5.5` tag 为准。小米 Vela 保证该 tag 与其内部基线等价。
- **集成清单归属**：最终产品 manifest 由**下游产品团队拥有**，引用本 SDK 的 vendor tag。本仓的 manifest 仅用于 BL 自家 CI / 发版 / 验证基准。
- **Review 门禁**：BL **自管主线**（vendor_bouffalolab 等仓）；下游产品团队仅在采纳某个 SDK tag 进产品时做准入 review。
- **源与分发分离**：源码 + review 在 BL 内部 gerrit；github 是**对外分发镜像**，下游消费方永不接触内部仓。

---

## 2. 仓库拓扑

```
github/open-vela/*                         OS 基座（小米 Vela，trunk-5.5 tag）
github/bouffalolab/bouffalo_vela_sdk       ← 本仓：manifest + CI + 发版入口
github/bouffalolab/vendor_bouffalolab      BL 芯片移植/驱动/board 适配层（源码镜像）
github/bouffalolab/bl_lhal                 寄存器级 HAL（源码，复用自 bouffalo_sdk）
github/bouffalolab/bl_wireless             无线协议栈（★预编译库 .a，方案 A）
github/bouffalolab/bl_phyrf                PHY/RF 校准（★预编译库 .a，方案 A）
```

> **方案 A**：wireless / phyrf 不公开源码——在 BL 内部用 openvela 同款工具链编成 `.a`，
> 只把库 + 公开头文件推到 github。详见 §5。

> **⚠️ 现状（与本节"目标态"有差距）**：当前 `manifests/bl-vela-sdk.xml` 是 **openvela
> trunk 全量基座的镜像**（≈170 个 project），BL 适配层 `vendor/bouffalolab` 已接入
> （当前为 `bl616cl/bl616cldg` 脚手架，驱动源码待填充），lhal/wireless/phyrf 尚未接入。
> 本节描述的是接入后的目标拓扑；收敛路径见 §7。
> 两个 remote 用的是**相对路径**（`../open-vela/`、`../bouffalolab/`），
> 即 open-vela 与 bouffalolab 必须与本清单仓位于同一 Git host 的同级命名空间下。

---

## 3. 版本号语义

格式：`bl-vela-sdk-<openvela基线>.<SDK迭代号>`，例：`bl-vela-sdk-trunk-5.5.1`

- `<openvela基线>`：本版本绑定的 openvela tag（如 `trunk-5.5`）。
- `<SDK迭代号>`：在该基线上的驱动功能/bugfix 累积号（`.0 .1 .2 …`）。
- 分支：`release/trunk-5.5` 持续出 `5.5.0 / 5.5.1 / …`。
- openvela 升到 5.6 → 拉 `release/trunk-5.6` 出 `5.6.0`，**同时 5.5.x 继续维护一段时间**（双线并行）。

**预编译库的版本绑定（硬约束）**：`bl_wireless` / `bl_phyrf` 的 ABI 必须与 openvela 基线工具链一致。
基线换工具链（如 5.5→5.6）→ 这两个库**必须重编重发**，tag 随基线走（`wireless-trunk-5.5.1`）。

---

## 4. 使用方式

### 4.1 开发 / 跟最新（开发清单）
```bash
repo init -u git@github.com:bouffalolab/bouffalo_vela_sdk.git \
          -b release/trunk-5.5 \
          -m manifests/bl-vela-sdk.xml
repo sync -j8
# 编译（board:config 名以 vendor/bouffalolab/boards 实际为准）
./build.sh vendor/bouffalolab/boards/bl616cl/bl616cldg:nsh -j8
```
> 开发清单的 `default revision` 是 `trunk`（跟分支），所以"跟最新"即跟 openvela trunk。
> **注意现状**：`vendor/bouffalolab` 已接入，但当前仅为 `bl616cl/bl616cldg` 脚手架
> （ARM 占位，真实 chip 移植/驱动源码待填充），尚不能直接编出固件；lhal/wireless/phyrf 仍未接入。

### 4.2 复现某个发版（冻结快照）
```bash
repo init -u git@github.com:bouffalolab/bouffalo_vela_sdk.git \
          -b bl-vela-sdk-trunk-5.5.1 \
          -m manifests/tags/bl-vela-sdk-trunk-5.5.1.xml
repo sync -j8
# 此时所有 project 钉死在发版时的具体 commit，bit-for-bit 可复现
```

### 4.3 下游产品消费（产品侧）
产品侧在自己的 product manifest 中引用本 SDK 的 vendor tag，叠加业务码后发版。
本 SDK 的冻结快照可作为产品 manifest 的 `<include>` 基底。

---

## 5. 发版流程

由 `scripts/release.sh` 自动化，串起：

```
内部 gerrit 源（review 通过）
  │
  ├─ vendor_bouffalolab / lhal ── 镜像 push ──▶ github 源码仓 + 打 tag
  │
  ├─ wireless / phyrf ── 内部编 .a（openvela 同款工具链）──▶ github 库仓 + 打 tag
  │
  ├─ 同步出完整树 → repo manifest -r → 冻结快照 tags/bl-vela-sdk-X.Y.Z.xml ──▶ 本仓
  │
  └─ gh release create（附冻结清单 + Release Notes）
```

执行：`scripts/release.sh trunk-5.5.1`（脚本内 TODO 项需按内部实际地址/工具链填充）。

---

## 6. ⚠️ 主线纪律（治理关键）

**任何为某个产品做的驱动改动，必须同步进 `vendor_bouffalolab` 主线。**

否则会出现「产品分支领先、SDK 主线腐烂」——这正是本 SDK 要消除的风险。
建议在 CI 卡：产品分支的驱动 commit 若无对应主线 cherry-pick，标红告警。

---

## 7. manifest 收敛流程（全量基座 → 最小闭包）

`manifests/bl-vela-sdk.xml` 当前是 **openvela trunk 全量基座的镜像**（≈170 个 project，
`default revision = trunk`），BL 适配层中 `vendor/bouffalolab` 已接入（脚手架，`revision`
继承默认 `trunk`），lhal/wireless/phyrf 仍待挂接。先保证「能像上游 openvela 一样 sync + 编」，再逐步收敛。

```
1. 接入 BL 适配层：✅ 已放开 vendor/bouffalolab（bl616cl/bl616cldg 脚手架）；待挂上 lhal/wireless/phyrf 仓
2. 放一个 BL616/BL618 的 defconfig
3. repo sync 全量 → 编译，记录实际被引用到的 project
4. 反向裁剪：删掉编译用不到的子系统
5. 回到 3，迭代到能干净编出固件 → 剩下的就是真实最小闭包
```

**当前仍保留、待裁剪确认**：vendor/xiaomi 系列、全部 benchmarks、
未用的 graphics/interpreters、四平台（linux/darwin/windows）工具链
（gcc/build-tools/cmake，由 `groups="notdefault,platform-*"` 控制按平台拉取）。
> 早期构想里这份清单是"起步最小集再往上加"，现已反转为"全量基座再往下裁"——
> 大方向（钉版冻结、可复现、BL 自管主线）不变，只是收敛起点换了。

---

## 8. CI

`.github/workflows/ci-build.yml`：仅 BL616/BL618 编译冒烟。
public 仓 + github 标准 runner = 免费。重型全量编译/测试仍在内部 CI。

---

## 目录

```
manifests/
  bl-vela-sdk.xml                   开发清单（openvela trunk 全量基座 + vendor/bouffalolab 脚手架）
  tags/
    bl-vela-sdk-trunk-5.5.1.xml     冻结快照样例（发版时脚本生成，钉 refs/tags/trunk-5.5）
scripts/
  release.sh                        发版自动化（内部→github）
```
> repo init 入口由 `.repo/manifest.xml` 经 `<include name="manifests/bl-vela-sdk.xml"/>`
> 选定，已无独立的 `default.xml`。
