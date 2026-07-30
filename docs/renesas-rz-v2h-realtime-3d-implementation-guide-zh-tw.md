# Renesas RZ/V2H 即時 3D 應用實作手冊

> 本文件是依據 Deep Vision Consulting〈A Realtime 3D Application on the Renesas RZ/V2H〉主題所撰寫的**原創實作手冊**，不是原文的逐字翻譯或替代品。原文內容、圖片與措辭請以[文章頁面](https://deepvisionconsulting.com/a-realtime-3d-application-on-the-renesas-rz-v2h/)為準。

## 1. 目標與完成標準

本手冊說明如何在 RZ/V2H 評估板上建立一個可重現、可量測的即時 3D 程式。完成後應能：

1. 由 Linux 啟動 EGL/OpenGL ES 圖形程式；
2. 持續繪製可旋轉的 3D 物件；
3. 使用 DRM/KMS 全螢幕顯示，或在 Wayland/Weston 視窗中顯示；
4. 記錄 FPS、幀時間、CPU/GPU 使用率與溫度；
5. 在離開程式或初始化失敗時，完整釋放 GPU 與顯示資源。

建議先採「最小三角形 → 深度測試立方體 → 紋理模型 → 效能調校」的順序。不要一開始便同時加入攝影機、AI 推論與複雜模型，否則難以定位瓶頸。

## 2. 適用環境與架構

### 2.1 硬體

- Renesas RZ/V2H 評估板及相容電源；
- microSD 或其他 BSP 支援的開機媒體；
- HDMI/顯示器與線材；
- USB-UART 序列線；
- 開發主機（建議 Ubuntu LTS，至少 100 GB 可用空間）；
- 選配：USB 攝影機、乙太網路與散熱風扇。

### 2.2 軟體層次

```text
應用程式
  ├─ 場景更新、相機、模型、材質、FPS 統計
  ├─ OpenGL ES 3.x（繪圖）
  ├─ EGL（context 與 surface）
  └─ Wayland/Weston 或 GBM + DRM/KMS（顯示）
Linux BSP
  ├─ GPU 使用者空間函式庫
  ├─ DRM/GPU 核心驅動
  └─ RZ/V2H 硬體
```

RZ/V2H 的 BSP、GPU 套件與授權檔必須使用彼此相容的版本。版本混用是 `eglInitialize()`、`eglCreateContext()` 或動態連結失敗的常見原因。

## 3. 取得 BSP 並建立映像

以下以 Yocto 工作流程示意；**release 名稱、layer 名稱與 machine 值應以所下載的 Renesas RZ/V2H BSP release note 為準**。

### 3.1 固定版本

建立一份 `versions.txt`，至少記下：

```text
BSP release=
Linux kernel=
Yocto release=
GPU package=
GPU package checksum=
machine=
image=
toolchain=
```

不要追蹤任意分支的最新提交。能重建同一映像，比偶然成功一次更重要。

### 3.2 設定建置主機

依 BSP 文件安裝 Yocto 相依套件，解壓 BSP 與需另外取得的 GPU 套件，再初始化環境：

```bash
source poky/oe-init-build-env build-rzv2h
```

檢查 layer 與 machine：

```bash
bitbake-layers show-layers
bitbake-layers show-recipes | grep -E 'mesa|wayland|weston|egl|gles'
bitbake -e | grep '^MACHINE='
```

在 `conf/local.conf` 加入開發階段需要的功能。實際套件名稱會隨 BSP 改變，先用 `bitbake-layers show-recipes` 確認：

```conf
DISTRO_FEATURES:append = " opengl wayland"
IMAGE_INSTALL:append = " weston wayland-utils kmscube"
EXTRA_IMAGE_FEATURES:append = " ssh-server-openssh tools-debug"
```

若採用純 DRM/KMS，不必強制加入 Weston，但仍需 EGL、OpenGL ES、GBM 與 DRM 的 runtime/development 套件。

### 3.3 建置與燒錄

```bash
bitbake <image-name>
```

依 release note 將 bootloader、核心、DTB 與 root filesystem 寫入開機媒體。燒錄前後都應核對 checksum；裝置節點（例如 `/dev/sdX`）務必以 `lsblk` 再確認，避免覆寫主機磁碟。

## 4. 板端驗證

先不要執行自己的程式。由序列主控台登入後，確認核心、顯示節點與動態函式庫：

```bash
uname -a
cat /etc/os-release
ls -l /dev/dri/
dmesg | grep -Ei 'drm|gpu|display|hdmi|firmware|error|fail'
ldconfig -p | grep -Ei 'EGL|GLES|wayland|gbm|drm'
```

若映像提供工具，再執行：

```bash
eglinfo
weston-info
kmscube
```

通過標準範例後才開始除錯自己的程式。若 `kmscube` 也失敗，問題通常在 BSP、DTB、權限、連接器模式或 GPU 套件，而非應用程式。

## 5. 建立最小專案

### 5.1 目錄

```text
rzv2h-3d-demo/
├── CMakeLists.txt
├── assets/
│   ├── model.glb
│   └── texture.png
├── shaders/
│   ├── basic.vert
│   └── basic.frag
└── src/
    ├── main.cpp
    ├── platform_egl.cpp
    ├── platform_egl.hpp
    ├── renderer.cpp
    └── renderer.hpp
```

將「原生視窗/顯示」、「EGL 生命週期」及「OpenGL ES renderer」分開，可在 Wayland 與 DRM 後端間切換，而不必改動場景邏輯。

### 5.2 CMake 骨架

```cmake
cmake_minimum_required(VERSION 3.16)
project(rzv2h_3d_demo LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

find_package(PkgConfig REQUIRED)
pkg_check_modules(EGL REQUIRED egl)
pkg_check_modules(GLES REQUIRED glesv2)

add_executable(rzv2h-3d-demo
  src/main.cpp
  src/platform_egl.cpp
  src/renderer.cpp)

target_include_directories(rzv2h-3d-demo PRIVATE
  ${EGL_INCLUDE_DIRS} ${GLES_INCLUDE_DIRS})
target_link_libraries(rzv2h-3d-demo PRIVATE
  ${EGL_LIBRARIES} ${GLES_LIBRARIES})
target_compile_options(rzv2h-3d-demo PRIVATE
  -Wall -Wextra -Wpedantic)
```

若 sysroot 不提供 `egl.pc` 或 `glesv2.pc`，改用 toolchain file 指定 vendor 函式庫；不要直接連結開發主機的 `/usr/lib`。

## 6. EGL 初始化順序

平台程式碼應嚴格遵循下列順序，並在每一步檢查錯誤：

1. 建立 Wayland display/window，或開啟 DRM 裝置並建立 GBM device/surface；
2. 以平台對應方式取得 `EGLDisplay`；
3. 呼叫 `eglInitialize()`，記錄 EGL major/minor；
4. 檢查 `eglQueryString()` 的 vendor、version、client APIs 與 extensions；
5. `eglBindAPI(EGL_OPENGL_ES_API)`；
6. `eglChooseConfig()`，要求 window surface、RGBA 色彩、depth buffer 與 ES3 renderable type；
7. 建立 `EGLSurface`；
8. 建立 OpenGL ES 3 context；
9. `eglMakeCurrent()`；
10. 設定 swap interval，進入繪圖迴圈。

概念性 config 屬性如下：

```cpp
const EGLint config_attribs[] = {
    EGL_SURFACE_TYPE, EGL_WINDOW_BIT,
    EGL_RENDERABLE_TYPE, EGL_OPENGL_ES3_BIT,
    EGL_RED_SIZE, 8,
    EGL_GREEN_SIZE, 8,
    EGL_BLUE_SIZE, 8,
    EGL_ALPHA_SIZE, 8,
    EGL_DEPTH_SIZE, 24,
    EGL_NONE
};

const EGLint context_attribs[] = {
    EGL_CONTEXT_CLIENT_VERSION, 3,
    EGL_NONE
};
```

不要假定第一個 `EGLConfig` 一定可用；應列印候選 config 的屬性，並確認它與原生視窗格式相容。

## 7. OpenGL ES 繪圖管線

### 7.1 著色器

頂點著色器：

```glsl
#version 300 es
layout(location = 0) in vec3 a_position;
layout(location = 1) in vec3 a_normal;
layout(location = 2) in vec2 a_uv;

uniform mat4 u_mvp;
uniform mat4 u_model;

out vec3 v_normal;
out vec2 v_uv;

void main() {
    gl_Position = u_mvp * vec4(a_position, 1.0);
    v_normal = mat3(u_model) * a_normal;
    v_uv = a_uv;
}
```

片段著色器：

```glsl
#version 300 es
precision mediump float;

in vec3 v_normal;
in vec2 v_uv;
uniform sampler2D u_texture;
out vec4 frag_color;

void main() {
    vec3 n = normalize(v_normal);
    vec3 light = normalize(vec3(0.4, 0.8, 0.5));
    float diffuse = max(dot(n, light), 0.15);
    frag_color = vec4(texture(u_texture, v_uv).rgb * diffuse, 1.0);
}
```

編譯與連結後必須讀取 `glGetShaderInfoLog()` 與 `glGetProgramInfoLog()`；即使回傳成功，也建議在開發映像保留日誌。

### 7.2 狀態與每幀流程

初始化一次：

```cpp
glEnable(GL_DEPTH_TEST);
glEnable(GL_CULL_FACE);
glCullFace(GL_BACK);
glFrontFace(GL_CCW);
```

每幀只做必要工作：

```text
讀取單調時鐘與輸入
→ 依 delta time 更新旋轉角度
→ 計算 model/view/projection 矩陣
→ 清除 color/depth buffer
→ 綁定 program、VAO、texture
→ 更新 uniform
→ glDrawElements()
→ eglSwapBuffers()
→ 更新統計資料
```

動畫必須依 elapsed time，而不是「每幀增加固定角度」。如此在 30 FPS 與 60 FPS 下，物件速度才會一致。

## 8. 模型與資源處理

- 優先使用 glTF 2.0/GLB；它比自訂文字格式更容易保留 mesh、材質與貼圖關係。
- 離線三角化模型，合併可共用材質的 draw call。
- 位置與法向量可使用 32-bit float；UV 通常可壓縮。
- 盡量使用 indexed drawing，並依 GPU 支援選擇 16-bit 或 32-bit index。
- 啟動階段一次載入並上傳靜態資源；不要在繪圖迴圈讀檔、解 PNG 或反覆建立 buffer。
- 先以 512×512 或 1024×1024 貼圖驗證，再逐步增加解析度。
- 若模型來自桌面 OpenGL 工具，需核對座標系、矩陣排列、面繞序、色彩空間與 shader precision。

## 9. 交叉編譯與部署

安裝與 BSP 完全相符的 SDK，然後建立獨立 shell：

```bash
source /opt/<vendor-sdk>/environment-setup-<target-triplet>
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --parallel
file build/rzv2h-3d-demo
```

`file` 的輸出必須是目標架構，而非 x86-64。部署時保留資源相對路徑：

```bash
rsync -av build/rzv2h-3d-demo assets shaders root@<board-ip>:/opt/rzv2h-3d-demo/
ssh root@<board-ip> 'cd /opt/rzv2h-3d-demo && ./rzv2h-3d-demo'
```

Wayland 後端可能需要：

```bash
export XDG_RUNTIME_DIR=/run/user/0
export WAYLAND_DISPLAY=wayland-0
```

DRM/KMS 後端則應停止會占用 DRM master 的 compositor，並由有權限存取 `/dev/dri/card*` 的使用者執行。兩種後端不要同時爭用同一顯示裝置。

## 10. 即時性與效能量測

### 10.1 正確量測幀時間

使用 `CLOCK_MONOTONIC` 或 `std::chrono::steady_clock`。每秒輸出一次彙總值，而非每幀 `printf`：

- 平均 FPS；
- 平均幀時間；
- P50、P95、P99 幀時間；
- 最差幀時間；
- missed-frame 次數（例如 60 Hz 下超過 16.67 ms）。

只報平均 FPS 會隱藏卡頓。測試至少包含 60 秒暖機與 5 分鐘穩態量測。

### 10.2 系統側工具

```bash
top -H -p "$(pidof rzv2h-3d-demo)"
pidstat -p "$(pidof rzv2h-3d-demo)" 1
cat /sys/class/thermal/thermal_zone*/temp
cat /sys/kernel/debug/dri/0/state
```

debugfs 路徑與 GPU 計數器依 BSP 而異；若節點不存在，先確認核心設定及掛載狀態，不要在正式映像永久開放不必要的 debug 介面。

### 10.3 調校順序

1. 關閉每幀日誌與 `glFinish()`；
2. 將靜態資料放入 VBO/IBO，減少 CPU→GPU 複製；
3. 合併材質與 draw call，減少狀態切換；
4. 降低 overdraw、透明混合與片段著色器成本；
5. 量測不同解析度，以判斷是否受 fill-rate 限制；
6. 加入 mipmap 與適當 texture filtering；
7. 最後才調整 CPU affinity、排程優先權或 DVFS。

提高即時排程優先權可能餓死系統服務；只有在量測證實 CPU 排程造成 deadline miss，且已設計 watchdog 與降級策略時才採用。

## 11. 攝影機或 AI 疊圖的擴充方式

若下一階段要把攝影機或 DRP-AI 推論結果疊在 3D 場景上，建議拆成三條管線：

```text
擷取執行緒：camera → buffer queue
推論執行緒：最新 frame → preprocess → accelerator → result queue
繪圖執行緒：texture upload/import → 3D scene → overlay → present
```

實務原則：

- queue 設定上限，延遲優先時丟棄舊幀，不讓資料無限堆積；
- 能使用 DMA-BUF/zero-copy 時避免額外 memcpy，但先做正確性版本再最佳化；
- 為擷取、前處理、推論、後處理、繪圖、present 分別打 timestamp；
- 將 AI 座標由輸入影像尺寸正確換算到 viewport，並處理 letterbox/pillarbox；
- renderer 只讀取一份不可變的最新結果，避免長時間持鎖。

## 12. 常見故障排除

| 症狀 | 優先檢查 | 建議處置 |
|---|---|---|
| 找不到 `libEGL.so` | rootfs 與 SDK 套件 | 安裝相符 runtime；以 `readelf -d` 檢查依賴 |
| `eglInitialize()` 失敗 | native display、驅動、權限 | 先跑 BSP 標準範例，列印 `eglGetError()` |
| `EGL_BAD_CONFIG` | surface/config 格式 | 列舉 config，核對 color/depth/renderable type |
| `EGL_BAD_MATCH` | context、config、window 不相容 | 核對 ES 版本與 native visual |
| DRM permission denied | 使用者群組或 compositor | 使用正確 seat/群組；停止占用 DRM master 的服務 |
| 黑畫面但無錯誤 | viewport、矩陣、面剔除 | 設定 `glViewport`；暫停 culling；畫固定三角形 |
| 模型全黑 | 法向量、光線、precision | 顯示純色，再逐項恢復 normal/texture/light |
| 畫面上下顛倒 | UV 或影像原點不同 | 離線翻轉或在 shader 一致轉換 V 座標 |
| FPS 被固定 | VSync/swap interval | 這可能是正確行為；同時觀察幀時間與卡頓 |
| 長時間後降速 | 溫度或 DVFS | 記錄 thermal/clock，改善散熱後重測 |
| 偶發卡頓 | 每幀配置、I/O、shader 編譯 | 預先配置、快取 shader、移除繪圖執行緒 I/O |

EGL 錯誤碼應轉為可讀字串；OpenGL ES 開發版則在重要階段呼叫 `glGetError()`。正式效能測試可降低檢查頻率，避免量測被除錯開銷污染。

## 13. 清理順序

程式收到 SIGINT、視窗關閉或發生可恢復錯誤時：

1. 停止繪圖與工作執行緒；
2. 等待 GPU 不再使用應用程式資源；
3. 刪除 texture、buffer、VAO、program 與 shader；
4. `eglMakeCurrent(display, EGL_NO_SURFACE, EGL_NO_SURFACE, EGL_NO_CONTEXT)`；
5. destroy context 與 surface；
6. `eglTerminate()`；
7. 銷毀 Wayland 或 GBM/DRM 原生資源；
8. 關閉檔案描述元並輸出最後統計。

所有初始化函式應只取得一類資源，並能讓呼叫端按反向順序清理。如此即使初始化到一半失敗，也不會殘留 DRM framebuffer 或失效指標。

## 14. 驗收清單

### 功能

- [ ] 冷開機後不需人工修正即可啟動；
- [ ] 畫面尺寸與顯示器模式正確；
- [ ] 深度、剔除、貼圖與動畫方向正確；
- [ ] resize/關閉/SIGINT 都不會當機；
- [ ] 連續執行 30 分鐘沒有資源持續成長。

### 效能

- [ ] 記錄解析度、更新率、BSP、GPU 套件與程式 commit；
- [ ] 具備 FPS、P95/P99 與最差幀時間；
- [ ] 同時保存 CPU、溫度與時脈資料；
- [ ] Release build、相同場景、相同 VSync 條件下比較；
- [ ] 結果包含原始 CSV，而不只有截圖。

### 可重現性

- [ ] 保存 SDK/BSP checksum 與 `local.conf` 變更；
- [ ] 專案可由乾淨 build directory 重建；
- [ ] 部署腳本不依賴開發者家目錄；
- [ ] README 列出執行參數、後端、資源路徑與已知限制。

## 15. 建議的實作里程碑

| 里程碑 | 產出 | 通過條件 |
|---|---|---|
| M0 平台 | 可開機映像與 UART 日誌 | DRM/GPU 無致命錯誤 |
| M1 顯示 | BSP 的 EGL/KMS 範例 | 穩定顯示 10 分鐘 |
| M2 最小繪圖 | 單一彩色三角形 | 正確 swap、可正常退出 |
| M3 3D | 旋轉立方體、深度測試 | 無閃爍、速度不隨 FPS 改變 |
| M4 資產 | GLB 模型與紋理 | 資源只在啟動時載入 |
| M5 量測 | CSV 與效能報告 | 含 P95/P99、溫度與測試條件 |
| M6 整合 | 攝影機/AI 疊圖（選配） | 有界 queue、延遲可分段量測 |

依此順序保留每個可工作的 commit。若後續整合失敗，可以快速退回上一個已驗證的基線。

## 參考入口

- [主題文章：A Realtime 3D Application on the Renesas RZ/V2H](https://deepvisionconsulting.com/a-realtime-3d-application-on-the-renesas-rz-v2h/)
- [Renesas RZ/V2H 產品頁](https://www.renesas.com/en/products/rz-v2h)
- [Khronos EGL Registry](https://registry.khronos.org/EGL/)
- [Khronos OpenGL ES Registry](https://registry.khronos.org/OpenGL/index_es.php)
- [Wayland Documentation](https://wayland.freedesktop.org/docs/html/)

下載 BSP、GPU 套件與硬體手冊時，應以產品頁導向的最新 release note 為準；本文的示例名稱不可取代特定版本的官方步驟。
