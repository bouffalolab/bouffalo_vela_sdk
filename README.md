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
repo init -u git@github.com:bouffalolab/bouffalo_vela_sdk.git -b release/trunk-5.5
repo sync -j8
# 编译（board:config 名以 vendor/bouffalolab/boards 实际为准）
./build.sh vendor/bouffalolab/boards/bl616:nsh -j8
```

### 4.2 复现某个发版（冻结快照）
```bash
repo init -u git@github.com:bouffalolab/bouffalo_vela_sdk.git \
          -b bl-vela-sdk-trunk-5.5.1 \
          -m tags/bl-vela-sdk-trunk-5.5.1.xml
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

## 7. manifest 最小闭包收敛流程

`manifests/bouffalo-vela.xml` 当前是「起步最小集 + 强相关项」，**不是**精确闭包
（vendor/bouffalolab 板级配置进树后才能定死）。收敛步骤：

```
1. 放一个 BL616/BL618 的 defconfig
2. repo sync 当前最小集 → 编译
3. 报「缺某路径 / 找不到某库」→ 把对应 project 从上游 openvela.xml 抄回本清单
4. 回到 2，迭代到能干净编出固件
   → 此时清单里剩的，就是真实最小闭包
```

**已剔除**（确认 BL 不需要）：其他所有 vendor（espressif/xiaomi/infineon+illd/bes/sifli/…）、
benchmarks、未用的 graphics/interpreters、非 RISC-V 工具链。

---

## 8. CI

`.github/workflows/ci-build.yml`：仅 BL616/BL618 编译冒烟。
public 仓 + github 标准 runner = 免费。重型全量编译/测试仍在内部 CI。

---

## 目录

```
manifests/
  default.xml                       repo init 默认入口
  bouffalo-vela.xml                 开发清单（跟 trunk-5.5）
  tags/
    bl-vela-sdk-trunk-5.5.1.xml     冻结快照样例（发版时脚本生成）
scripts/
  release.sh                        发版自动化（内部→github）
```
