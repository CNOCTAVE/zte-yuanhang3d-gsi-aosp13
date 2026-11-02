# ZTE Z9101（远航 3D）AOSP 13 GSI 刷机包

生成时间：2026-11-02 · 设备序列号 320147137223 · UNISOC UMS9620-AB（T760）

---

## 1. 包内容

| 文件 | 大小（字节） | 说明 |
|---|---|---|
| `super.bin` | 9,487,515,648 | sha256 `93fab37f6a3af341a338f39c1ce8cb1b40e05548e17a4b0a6f1a1591a50353be`；原厂 super 的副本，`system_a` extent 已替换为自编 GSI，其余 extent 逐字节保持原厂 |
| `partition_1793452837.xml` | 3,227 | 展锐下载用的分区尺寸表（原样复制，未改） |
| `pgpt.bin` | 32,768 | 原始 GPT（未改） |
| `custom_exec_no_verify_65012f48.bin` | 96 | 展锐下载模式配套文件（未改） |

上层目录另有 `../system_gsi.img`（1,907,032,064 字节，sha256 `5fc8d134dc6e71ab247eb4370b7300f2bf50dc6a91cc418307c33142c4c1335a`），即单独的可刷 system 镜像。

**原始备份 `D:\肥猫小助手\TQ` 全程只读，一个字节都没有改动。**

---

## 2. 镜像是怎么来的（可复现）

```bash
# 环境：WSL Ubuntu-22.04，/aosp（AOSP 13，android13-release，BUILD_ID=TP1A.221105.002）
# 宿主包：libncurses5 libtinfo5 python-is-python3 zip unzip xz-utils lzop liblz4-tool zstd device-tree-compiler
repo init -u https://android.googlesource.com/platform/manifest -b android13-release --depth=1 --partial-clone
repo sync -j4
source build/envsetup.sh
lunch aosp_arm64-userdebug
m systemimage -j6          # 目标名没有下划线；generic_arm64 是合并布局，system_ext 已在 system.img 内
# 产物 out/target/product/generic_arm64/system.img

# 拼进 super：system_a 起始 = 2048 扇区 x 512 = 1 MiB（由本机 lpdump 实测）
cp <原厂>/super.bin super.bin
dd if=system.img of=super.bin bs=1M seek=1 conv=notrunc
```

`aosp_arm64`（而非 `aosp_arm64_64only`）是因为本机 `ro.product.cpu.abilist32` 非空，vendor 支持 32 位。

## 3. 已做的完整性验证

- 写入区 1,907,032,064 字节与 `system.img` 逐字节 `cmp` **完全一致**
- LP 元数据区（0–1 MiB）与原厂**逐字节相同**，`dd` 从 1 MiB 起写，未触碰超级块与元数据槽
- `system_ext_a` / `vendor_a` / `product_a` / `odm_a` 起点仍是原厂 EROFS 魔数 `e0f5e1e2`；4.7 GB、5.5 GB、6.5 GB、文件尾四个窗口抽查均与原厂一致
- GSI 1.78 GB < `system_a` extent 4.04 GB，**未越界**

## 4. 兼容性审计（不需真机，已用原厂 vendor 元数据实测）

用备份里真实的 vendor+odm VINTF manifest（40 个文件、91 条 HAL 声明、86 个实例键）对照自编 GSI 的 framework matrix：

- `compatibility_matrix.7` 中 7 条非 optional 要求，**满足 6 条**
- 唯一缺口：**AIDL `android.hardware.power`**——matrix 要 major 2（minor 0–3），厂商声明 major 3。影响：功耗 hint 客户端绑不上、退化为 no-op，**不是启动阻断**
- `audio@7.1` 落在 matrix 允许的 `7.0-1` 区间 ✓；AIDL `IHealth/default` 存在 ✓
- 反向（vendor 要求 framework）仅 `android.hidl.manager@1.0` 静态缺失，由镜像内 `hwservicemanager` 运行时自注册且我们提供 1.2（向后兼容）✓
- 原厂元数据自身有问题：`/odm/etc/vintf/manifest/manifest_kernel.xml` 写 `<kernel target-level="6"/>`，而 vendor manifest 是 `target-level="7"`，`assemble_vintf` 拒绝合并并报 "level 7 must not set kernel level 6"

版本上限结论：**只能 AOSP 13**。vendor `target-level=7`、HIDL HAL 18 个 / AIDL 3 个，Android 14+ 移除了 HIDL 加载，`graphics.composer@2.4`、`camera.provider@2.4`、`audio@7.1` 等点不亮。

---

## 5. ⚠ 当前这台机器刷不进去——原因与依据

解锁码是唯一门槛，且**中兴不提供**。已逐条排除其它路径：

| 路径 | 实测结果 |
|---|---|
| `fastboot flashing unlock` / `oem unlock` | 前者 `unknown cmd`；后者需厂商按 IMEI/SN 生成的码，无码即 `Unlock bootloader fail.` |
| 读取 unlock challenge | 中兴残废版 fastboot 里 `getvar *` 全空，只有 `partition-size:*` 可用，拿不到 challenge |
| DSU（Dynamic System Updates） | `/system/bin/dsud`、`dsu_note` **在这套固件里不存在**；`ro.build.type=user`、`ro.debuggable=0`、adb 只有 uid=2000 shell |
| 直接刷修改过的 super | 原厂 vbmeta 里有 `system` 的 hashtree 描述符，替换 system 后首次挂载即 verified boot 失败；锁定态下载模式也会拒绝未签名镜像 |

绕过最后一项需要改写中兴签名的 AVB 验证链，**这件事不做**。

因此本包的实际用途：
1. 日后取得解锁码（厂商渠道或按 IMEI 出码的第三方服务）后直接可用；
2. 给**已解锁的同型号机器**使用；
3. 作为已验证的构建产物存档。

## 6. 刷写步骤（仅在解锁后执行）

```bash
# 0) 先确认已解锁：拿到码后
fastboot oem unlock 0x<解锁码>          # 会清空 userdata，原厂备份里没有 userdata，无法还原个人数据

# 1) 关掉 verity / 验证（GSI 用 AOSP testkey 签名，锁着验证必失败）
fastboot --disable-verity --disable-verification flash vbmeta <原厂vbmeta副本>

# 2) 线刷整块 super（展锐走 ResearchDownload / 肥猫小助手"写入分区"）
#    目录里必须有名为 super.bin 的文件（工具按分区名匹配，本包已是该名字）
#    只勾选 super 一个分区，不要动 prodnv / nr_fixnv1 / nr_fixnv2 / calinv（射频 NV，写了找不回）

# 3) 首启失败时：长按电源 15s 强制重启进入下载模式，用 D:\肥猫小助手\TQ 原件回滚
```

回滚（救砖）：用 `D:\肥猫小助手\TQ\3D_20261031_Oem_Lock_20261031212031\` 里的原厂 `super.bin` 原样刷回即可， userdata 不在 super 内，所以回滚不会额外清数据。

## 7. 已知限制

- `super.bin` 里 system 是 **ext4**，原厂是 EROFS。内核本就支持 ext4，但分区文件类型与原厂不一致。
- 分区几何未变，所以**不需要** `lpmake`/`mkfs.erofs`。
- slot B 的 extent 全空（VAB 虚拟 A/B），本包只改了 slot A。
- GSI 是 `userdebug` + testkey，`adb root` 可用，但银行类 App 的 SafetyNet/Play 校验会失败。
