# R17 (PBEM00) KernelSU-Next 编译

## 已确认状态

你的 `ten` 分支里 `arch/arm64/configs/sdm670_r17_ksu_defconfig` 已就位（4589 行），
关键取值校验通过：

```
CONFIG_KSU=y
# CONFIG_KPROBES is not set
# CONFIG_KPROBE_EVENTS is not set
# CONFIG_KSU_KPROBE_HOOKS is not set
# CONFIG_OPPO_ROOT_CHECK is not set
# CONFIG_OPPO_EXECVE_BLOCK is not set
# CONFIG_OPPO_EXECVE_REPORT is not set
# CONFIG_MODULE_SIG is not set
# CONFIG_MODULE_SIG_FORCE is not set
CONFIG_OVERLAY_FS=y
CONFIG_KALLSYMS=y
```

唯一缺的是 workflow（`.github` 目录还不存在）。

## 需要传上去的文件

```
你的仓库 (ten 分支)/
├── .github/workflows/build-kernel.yml   ← 已改好，直接检出你的 fork
└── ksu-build/
    ├── inject_hooks.py                  ← hook 注入脚本（实测 6/6 通过）
    ├── btfm_slim.h                      ← 蓝牙头文件兜底副本
    └── btfm_slim_wcn3990.h              ← 同上
```

放 `ksu-build/` 而非 `scripts/`，是因为内核源码自带 `scripts/` 目录，避免混淆。

## ten 分支已修复的三个坑

| 问题 | 根因 | 修法 |
|---|---|---|
| `btfm_slim.h` 找不到 | 文件其实在，但用 `#include <...>` 而 Makefile 无 `-I` | 加 `ccflags-y += -I$(src)` |
| `tfa98xx.c` 编译错 | R17 无 NXP 功放，但 techpack Makefile 硬编码 | sed 删 Makefile 中 tfa98 行 |
| `-Wno-enum-conversion` | GCC 4.9 不识别，被 `-Werror` 放大 | 全局清除该选项 |

这些都属于 ten 分支（realme X fork）从未编译过的代码路径，
被你 R17 的出厂配置触发才暴露。

## 上传

```bash
git clone -b ten https://github.com/hgffdkhn-dot/android_kernel_oppo_PBEM00.git
cd android_kernel_oppo_PBEM00
mkdir -p .github/workflows ksu-build
cp build-kernel.yml .github/workflows/
cp inject_hooks.py  ksu-build/
git add .github ksu-build
git commit -m "ci: add KernelSU-Next build workflow"
git push origin ten
```

## 触发编译

Actions → `Build R17 Kernel` → `Run workflow`

| 模式 | 用途 |
|---|---|
| `stock` | 裸内核。**先跑这个**，验证能否开机 |
| `ksunext` | 集成 KSU。stock 通过后再跑 |

产物在 Artifacts：`r17-stock` / `r17-ksunext`，含 `Image.gz-dtb` + `config.txt`。

## 和你之前那份的区别

原版工作流是「单独 CI 仓库 + checkout NiceM126 源码」。
现在改成**直接 checkout 你自己的 fork**，因为 defconfig 已经在你的分支里了。

## 注意事项

1. **先跑 stock**。ten 分支缺 12 个 OPPO 驱动源码，必须先用裸内核确认能开机。

2. **编译日志搜 `kernel_read`**。ksunext 模式下 cherry-pick 完原型后，
   `fs/exec.c:937` 和 `fs/exec.c:1551` 两处调用要改新参数顺序，不改会报错。

3. **cherry-pick 冲突不会中断**，会打 `[!] 冲突 xxx` 日志，需要你手动处理。

4. 工具链从 AOSP clone，可能要几分钟。

## 刷入

```bash
dd if=/dev/block/bootdevice/by-name/boot of=/sdcard/boot_stock.img
```

用 AnyKernel3 打包 `Image.gz-dtb` 后刷入。
