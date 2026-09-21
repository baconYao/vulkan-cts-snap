# Vulkan CTS Snap：core22/24/26 多 base 建置與除錯紀錄（繁體中文）

本文記錄將 `vulkan-cts-snap` 擴充為支援 `core22`／`core24`／`core26`
三個 base（各自對應 Snap Store 上已註冊的 `baconyao-vulkan-cts-22`／
`-24`／`-26` 三個獨立套件名稱），涵蓋 `amd64` 與 `arm64` 兩種架構，並在
`machines.json` 列出的 5 台實機上驗證的完整過程與踩到的坑。

## 目標與範圍

- 3 個 base（core22 / core24 / core26）× 2 個架構（amd64 / arm64）
  = 共 6 個 snap 產物。
- 每個 base 對應一個獨立、已在 Snap Store 註冊的套件名稱，各自發布到
  自己的 `edge` channel（非同一套件的不同 track）。
- 在使用者提供的 5 台機器上實測：G1200 2、G1200 3、G700 1、CIX P1、
  NXP Leuven（全部都是 arm64 機器）。

## 建置方式：為什麼用 Launchpad remote-build

這台開發機沒有 `sudo`、沒有 LXD、也沒有 Multipass，因此無法本機建置，
所有 6 個組合都透過 `snapcraft remote-build`（背後用 Launchpad 的
build farm）完成。過程中發現幾個實務上的怪癖：

1. **下載常常卡住**：即使 Launchpad 那邊已經 `Succeeded`，
   `snapcraft` 內建的下載器經常卡在 0 bytes 不動。解法：從
   log 裡擷取 Launchpad 的直接下載網址，`kill` 掉卡住的
   `snapcraft` 行程（必須用數字 PID，不能用 `pkill`），改用
   `wget -c <url>` 直接下載，穩定成功。
2. **core22 (jammy) 的 arm64 build 特別不穩**：同樣的
   `platforms:` 寫法在 core24／core26 都能一次過，但 core22
   前後失敗了 5 次，包含 Launchpad 基礎設施的暫時性錯誤
   （"Instance is not running"）和 build-plan 驗證錯誤
   （"build-on architectures ... does not match host architecture"）。
   最後改用舊式的 `architectures:` 寫法（而非新式 `platforms:`）才
   順利通過 build-plan 驗證，並額外挖出下一個真正的 bug。
3. **絕對不要讓同一個 base 目錄的 amd64 與 arm64 build script
   同時跑**：兩個 script 都會在同一個資料夹裡做
   `rm -rf .git && git init` 的暫存操作，若同時執行會互相
   破壞對方的 `.git` 與 `snapcraft.yaml`（曾經真的中招，
   導致 3 個 base 的 yaml 內容全部混在一起）。正確作法是「不同
   base 目錄可以平行跑，但同一個 base 目錄裡 amd64／arm64 兩個
   build 必須先後序列執行」。

## Bug #1：core22 arm64 交叉編譯時的 apt 相依性衝突

用舊式 `architectures:` 格式通過 build-plan 驗證後，實際編譯階段在
安裝 arm64 交叉套件時失敗：

```
libgl1-mesa-dri:arm64 依賴 libelf1:arm64 (>= 0.142)，但 apt 拒絕安裝
```

原因是 host 上已經裝了 amd64 版本的 `libelf1`（給 native 編譯用），
而 apt 的 `Multi-Arch: same` 限制要求同一套件在不同架構下版本必須
完全一致，apt 因此「按兵不動」而非自動升級。

**修法**：在 `core22/snap/snapcraft.yaml` 的 `vulkan-cts` part 的
`override-build` 裡，於安裝其他交叉套件「之前」先手動
`apt-get install -y --no-install-recommends libelf1:arm64`，強迫
apt 先把它處理好。

## Bug #2：patchelf 把 arm64 執行檔的 ELF interpreter 寫成 amd64 的路徑（最關鍵的 bug）

修完 apt 的問題後，core22 的 arm64 build 終於成功產出 snap，但裝到
G1200 2 上執行 `baconyao-vulkan-cts-24.test` 時卻出現：

```
deqp-vk: cannot execute: required file not found
```

### 根因

`snapcraft.yaml` 裡 `vulkan-cts` part 原本有
`build-attributes: [enable-patchelf]`。這個屬性會讓 snapcraft 在
打包階段，把每一個 ELF 執行檔的 `PT_INTERP`（動態連結器路徑）改寫成
「編譯所在主機」（build host，也就是 amd64）的 core base 連結器路徑，
例如：

```
/snap/core24/current/lib64/ld-linux-x86-64.so.2
```

但這個 snap 實際上是要跑在 **arm64** 的機器上！amd64 的連結器路徑
在 arm64 環境裡根本不存在，execve 自然回報「required file not
found」。用 Python 手動解析 ELF program header（因為 Ubuntu Core
測試機上沒有 `file`／`readelf` 可用）證實：不管是自己編譯的
`deqp-vk`，還是從 deb 套件 staging 進來的 `vulkaninfo`，全部都中招，
且 3 個 base（core22/24/26）都一樣。

### 修法

直接把 `build-attributes: [enable-patchelf]` 從 3 個 base 的
`vulkan-cts` part 移除。`enable-patchelf` 本來是為了 classic
confinement（沒有獨立掛載命名空間）而設計的可攜性補丁；strict
confinement 的 snap 本來就在自己的掛載命名空間內執行，base snap
自己的 `/lib` 目錄就已經提供了正確、原生架構的連結器，完全不需要
patchelf 介入。

## Bug #3：透過 content-interface 使用 vendor Vulkan 驅動時，本機 Loader 搶先被載入

修完 patchelf 之後，重新編譯 arm64 版本並安裝到 G1200 2（用
`gpu-2404` content-interface 連接 MediaTek 提供的
`mediatek-genio-g1200-gpu-drivers-core24` GPU provider snap），執行
`dEQP-VK.info.build` 這次終於能執行了，但立刻出現：

```
FATAL ERROR: vk.createInstance(...): VK_ERROR_INCOMPATIBLE_DRIVER
```

### 根因

用 `LD_DEBUG=libs` 追蹤動態連結過程後發現：`deqp-vk` 本身不會直接
`dlopen("libvulkan.so.1")` 時，實際載入的是我們自己 snap 裡透過
`stage-packages: [libvulkan1]` 帶進來的標準 Khronos Vulkan Loader
（在 `$SNAP/usr/lib`），而不是 content-interface 提供的 vendor
版本（在 `$SNAP/gpu-2404/lib`）。原因是 `bin/test` 腳本設定
`LD_LIBRARY_PATH` 時，`$SNAP/usr/lib` 排在 `$SNAP/gpu-2404/lib`
「前面」，所以動態連結器優先找到我們自己帶的那一份。

### 修法

把 `$SNAP/gpu-2404/lib`（content-interface 提供的函式庫目錄）
「插到 `LD_LIBRARY_PATH` 最前面」，確保 vendor 提供的
`libvulkan.so.1` 優先被載入。

## Bug #4：Vendor 提供 `libmali.so` 但沒有附上 ICD manifest json

修完 Bug #3 之後重新測試，仍然是同一個
`VK_ERROR_INCOMPATIBLE_DRIVER` 錯誤，但這次連
`gpu-2404/lib/libvulkan.so.1`（vendor 版本）都已經正確被載入了。

用 `VK_LOADER_DEBUG=all` 追蹤 Khronos Loader 的行為，發現它有去
`/usr/share/vulkan/icd.d/` 等標準路徑搜尋 ICD manifest（描述實際
GPU 驅動 `.so` 檔案位置的 json 檔），但完全找不到任何檔案，最後印出：

```
ERROR | DRIVER: vkCreateInstance: Found no drivers!
```

### 根因

`mediatek-genio-g1200-gpu-drivers-core24` 這個 provider snap 透過
content-interface 分享出來的內容裡，只有 `libmali.so`（Arm Mali GPU
驅動本體）和 `libvulkan.so.1`（標準 Khronos Loader），完全沒有附帶
`usr/share/vulkan/icd.d/*.json` 這種 ICD manifest 檔案。用 Python
直接搜尋 `libmali.so` 的二進位內容，證實它確實有實作
`vk_icdGetInstanceProcAddr`、`vk_icdNegotiateLoaderICDInterfaceVersion`
等標準 ICD 介面函式──也就是說驅動本身是「合格」的 ICD，只是缺一份
告訴 Loader「去哪裡找它」的 manifest 檔案而已。

### 修法

在 `bin/test` 腳本裡，當偵測到 content-interface 目錄下沒有現成的
ICD manifest、但找得到 `libmali.so` 時，**在執行期動態產生一份
ICD manifest json**（寫到 `$XDG_RUNTIME_DIR` 或
`$SNAP_USER_COMMON`），內容指向該 `libmali.so` 的實際路徑，再把
`VK_ICD_FILENAMES` 指向這份產生出來的 json。三個 base 的
`bin/test` 都套用了同樣的邏輯（分別對應各自的 content-interface
名稱：core22 為 `graphics`、core24 為 `gpu-2404`、core26 為
`gpu-2604`）。

## Bug #5：core26 的 configure hook 用 `#!/usr/bin/env python3` 但 base 裡沒有 python3

在 CIX P1（core26）上 `snap install` 時失敗：

```
env: 'python3': No such file or directory
```

`core26/snap/hooks/configure` 沿用了 core22/core24 的寫法（用一支
極簡的 Python script 印一行訊息），但 core26 (resolute) 的 base
image 顯然沒有內建 `python3` 可用。由於這個 hook 本來就只是印一行
訊息、完全沒有邏輯，直接改成 `#!/bin/sh` 就完全避開這個相依性問題。

## `--no-confinement` 模式的重要提醒：不能透過 `snap run` 呼叫

在測試 G700 1（Debian GPU，用 `--no-confinement` 模式）時，一開始
用標準的：

```
baconyao-vulkan-cts-24.test --no-confinement dEQP-VK.info.build
```

結果失敗：

```
Error: --no-confinement requires the host Vulkan loader at
/usr/lib/aarch64-linux-gnu/libvulkan.so.1
```

即使該路徑在 host 上確實存在（`ls` 可以看到）。原因是：這樣呼叫
仍然是透過 `snap run`（也就是 `<snap>.test` 這個 command）啟動，
一樣會進入 strict confinement 的獨立掛載命名空間，在這個命名空間裡
`/usr/lib` 是 base snap 提供的內容，而不是 host 的 `/usr/lib`，所以
host 的 Vulkan 函式庫自然「看不到」。

**正確用法**：必須繞過 `snap run`，直接呼叫 snap 安裝路徑下的原始
腳本，徹底跳出 confinement：

```bash
sudo /snap/baconyao-vulkan-cts-24/current/test --no-confinement dEQP-VK.info.build
```

這樣才會真的用 host 的檔案系統、host 的 Vulkan Loader 與驅動。

## 最終測試結果（5 台機器全部通過，含真實 GPU 運算）

| 機器 | Base | 連接方式 | dEQP-VK.info.build | dEQP-VK.compute.pipeline.basic.* |
|---|---|---|---|---|
| G1200 2 | core24 | content-interface (`gpu-2404`) | Pass | 54/78 通過（1 Fail、23 Not supported，屬正常的硬體差異） |
| G1200 3 | core24 | content-interface (`gpu-2404`) | Pass | 同 G1200 2 |
| G700 1 | core24 | `--no-confinement`（host deb 驅動） | Pass | 54/78 通過 |
| CIX P1 | core26 | `--no-confinement`（host deb 驅動） | Pass | 78/80 通過 |
| NXP Leuven | core24 | `--no-confinement`（host deb 驅動） | Pass | 72/78 通過 |

「Compute succeeded」等訊息代表測試案例是真的在 GPU 上跑完一整輪
compute shader 運算並讀回結果驗證，不只是驅動載入成功而已，可以
確定 3 個 bug 全部真正修好、GPU 真的有在動。

## 已知限制：core22 沒有對應的 arm64 測試機

`machines.json` 列出的 5 台機器全部是 Ubuntu 24.04 或 26.04，沒有
任何一台是 22.04 (jammy)。因此 **core22 的 arm64 版本從未在真實硬體
上測試過**，只確認過它能成功編譯、產出有效的 squashfs 檔案、且
ELF interpreter 已修正為正確的 arm64 路徑。若之後拿到 22.04 的
arm64 機器，建議優先重新驗證這個組合。core22 的 amd64 版本則沒有
這個限制（可以在一般 x86_64 桌機/伺服器上安裝測試）。

## Snap Store 發布現況

`.github/workflows/snap-ci.yml` 已經設定好完整的 CI/CD 流程：
`push` 到 default branch 時會自動建置全部 6 個組合，並透過
`snapcore/action-publish@v1` 發布到對應套件（`baconyao-vulkan-cts-22`
`/-24/-26`）各自的 `edge` channel，前提是 GitHub repo 的 Secrets
裡要設定 `SNAPCRAFT_STORE_CREDENTIALS`。

**這台開發機上沒有已登入的 Snap Store 帳號**（`snapcraft whoami`
回報 invalid-credentials，也沒有任何本機儲存的 store 憑證），因此
本次工作階段**沒有實際執行 `snapcraft upload/release` 推上
edge channel**。若要手動推送，可在有帳號登入權限的環境下執行：

```bash
snapcraft upload --release=edge /tmp/vulkan-builds/core22/baconyao-vulkan-cts-22_1.4.5_amd64.snap
snapcraft upload --release=edge /tmp/vulkan-builds/core22/baconyao-vulkan-cts-22_1.4.5_arm64.snap
snapcraft upload --release=edge /tmp/vulkan-builds/core24/baconyao-vulkan-cts-24_1.4.5_amd64.snap
snapcraft upload --release=edge /tmp/vulkan-builds/core24/baconyao-vulkan-cts-24_1.4.5_arm64.snap
snapcraft upload --release=edge /tmp/vulkan-builds/core26/baconyao-vulkan-cts-26_1.4.6_amd64.snap
snapcraft upload --release=edge /tmp/vulkan-builds/core26/baconyao-vulkan-cts-26_1.4.6_arm64.snap
```

或者在 GitHub repo 設定好 `SNAPCRAFT_STORE_CREDENTIALS` 這個
secret 後，把 `baconyao-experiment` 分支合併進 default branch，
讓既有的 CI workflow 自動完成建置與發布。
