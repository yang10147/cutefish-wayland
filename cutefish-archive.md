# CutefishOS Wayland 移植 — 归档文档
> 已完成组件记录。新对话开始 libcutefish 移植时上传此文件。

---

## 用户环境

```
OS:        CachyOS x86_64
DE:        KDE Plasma 6（Wayland 会话）
Qt 版本:   6.10.2 / KDE Frameworks 6.24.0 / 内核 6.19.10-1s
用户名:    yong / 主目录: /home/yong
代理端口:  7890（克隆前需设置）
会话类型:  wayland
```

## 重要背景知识

- **FishUI**：Qt5 编译，Qt6 不可加载。所有 QML 必须完全去掉 FishUI，用原生 Qt6 替代。
- **KWindowSystem**：Wayland 下窗口枚举/激活 API 不可用。PlasmaWindowManagement 协议仅对 `plasmashell` 开放，第三方程序被拒绝（KDE 安全限制）。
- **交付方式**：Claude 打包 `.tar.gz`，解压后 cp 替换，再编译测试。tar 命令在下载目录执行。
- **QML 编译进二进制**：修改 QML 后必须删除 `qrc_resources.cpp` 强制重编：
  ```bash
  rm ~/cutefish-settings/build/cutefish-settings_autogen/UVLADIE3JM/qrc_resources.cpp
  make -j$(nproc)
  sudo cp ~/cutefish-settings/build/cutefish-settings /usr/bin/cutefish-settings
  ```
- **Theme 单例**：`qml/Theme.qml` 作为 FishUI 替代，根类型 `Item`，`qml/qmldir` 注册为 singleton，子目录用 `import "../"` 访问。
- **QT_QUICK_CONTROLS_STYLE=Basic**：必须在 main.cpp 最早处 `setenv` 设置，在 QApplication 创建前。
- **ListModel 在自定义组件内报错**：改用 JS 数组，delegate 里用 `modelData`。
- **音频插件**：`org.kde.plasma.private.volume`，`HasVolume`→`hasVolume`，`PulseAudio.NormalVolume`→`65536`。
- **settings-daemon 需手动启动**：`cutefish-settings-daemon &`（session 集成待后续处理）。

---

## 已完成组件

### cutefish-dock ✅
LayerShellQt 贴底部，exclusiveZone 生效，图标/右键菜单正常。窗口激活/高亮空实现（PlasmaWindowManagement 协议限制）。

关键改动：Qt5→Qt6/KF6，去 X11，加 LayerShellQt；QML 完全重写去 FishUI；手写 DBus adaptor；注册 icontheme image provider。

### cutefish-statusbar ✅
LayerShellQt 贴顶部，系统托盘/时钟/关机菜单/控制中心正常。**音量滑块**改用 `pactl` 命令行实现（原 DBus 接口不存在）。亮度滑块依赖 settings-daemon 运行。窗口标题空实现（同 dock 限制）。

关键改动：同 dock 移植策略；`volume.cpp` 完全重写用 QProcess 调 pactl，2 秒轮询同步外部音量变化。

### cutefish-core 各子组件 ✅

| 组件 | 关键改动 |
|------|---------|
| session | 直接编译，无需修改 |
| shutdown-ui | Qt5→Qt6，去 FishUI/GraphicalEffects，纯深色渐变背景 |
| settings-daemon | 去所有 X11/XCB；mouse/touchpad 改用 kwinrc+KWin DBus；theme 去 XResources |
| screen-brightness | Qt5→Qt6，源码无需修改 |
| notificationd | Qt5→Qt6/KF6，去 KWindowSystem X11 调用，去 FishUI，注册 icontheme provider |
| gmenuproxy | Qt5→Qt6/KF6，去 XCB，窗口属性 no-op |
| powerman | Qt5→Qt6/KF6，去 XCB/DPMS，DPMS 调用 no-op |
| polkit-agent | Qt5→Qt6，PolkitQt5→PolkitQt6，去 FishUI，拖动改 startSystemMove() |
| chotkeys | XCB xcb_grab_key 完全重写为 KF6GlobalAccel C++ API |
| clipboard | Qt5→Qt6，QApplication→QGuiApplication |

### cutefish-screenshot ✅
通过 XDG Desktop Portal 截图（Wayland 兼容）。剪贴板功能已修复。

### cutefish-launcher ✅
### cutefish-screenlocker ✅

### cutefish-filemanager ✅
文件列表/右键菜单/重命名/双击打开均正常。

### cutefish-settings ✅（网络三件套/User 除外）

**编译方法：**
```bash
cd ~/cutefish-settings/build
cmake .. -DCMAKE_INSTALL_PREFIX=/usr
rm cutefish-settings_autogen/UVLADIE3JM/qrc_resources.cpp
make -j$(nproc) && sudo cp build/cutefish-settings /usr/bin/cutefish-settings
```

**已完成页面：** About / Notification / Power / Proxy / Touchpad / Fonts / DateTime / DefaultApp / Battery / Dock / Language / Display（亮度+缩放）/ Appearance / Wallpaper / Sound / Cursor（图标占位）

**跳过页面：**
- WLAN / Wired / Bluetooth → 依赖 `Cutefish.NetworkManagement` Qt5 插件（libcutefish 移植目标）
- User → 依赖 `Cutefish.Accounts` Qt5 插件（libcutefish 移植目标）

**关键 C++ 改动：**
- Qt5→Qt6/KF6，去 X11Extras
- `background.cpp`：std::sort lambda 修复
- `cursor/cursortheme.cpp`：QX11Info → Qt6 native interface
- `fonts/kxftconfig.cpp`：QX11Info::appDpiY() → primaryScreen()->logicalDotsPerInchY()

---

## 已知遗留问题

| 问题 | 原因 | 优先级 |
|------|------|--------|
| Dock/Statusbar 窗口激活/高亮缺失 | PlasmaWindowManagement 协议仅对 plasmashell 开放 | 低（待自有 compositor）|
| Statusbar 背景色写死灰色 | 需壁纸服务配合 | 低 |
| Display 分辨率/旋转不可用 | 依赖 Qt5 Cutefish.Screen 插件 | 中 |
| Cursor 页面图标空白 | cursor.svg 未正确编译进 qrc | 低（视觉） |
| settings-daemon 未自动启动 | 缺 autostart/session 集成 | 中（完整桌面必须） |
| powerman/shutdown-ui 按钮无效 | 依赖 com.cutefish.Session | 待 session 集成 |
| 光标被放大 | cupdatecursor 写入了光标配置 | 待查 |

## 跳过组件

- `cutefish-core/xembed-sni-proxy` — 纯 X11
- `cutefish-core/cupdatecursor` — 纯 X11
- `cutefish-core/sddm-helper` — 暂不处理
- `cutefish-calculator / cutefish-icons / cutefish-wallpapers / cutefish-qt-plugins`

---

## 下一阶段：libcutefish 移植

### 目标
libcutefish 是一组 Qt5 QML 插件，移植为 Qt6 后可解锁 cutefish-settings 的：
- **WLAN / Wired / Bluetooth** 页面（依赖 `Cutefish.NetworkManagement`）
- **User** 页面（依赖 `Cutefish.Accounts`）

### 插件清单

| 插件 | 功能 | 依赖 |
|------|------|------|
| `Cutefish.NetworkManagement` | 网络管理（WiFi/有线/蓝牙） | NetworkManagerQt, ModemManagerQt |
| `Cutefish.Accounts` | 用户账户管理 | AccountsService DBus |
| `Cutefish.Screen` | 屏幕管理 | KScreen（已用 kscreen-doctor 绕过，可跳过）|

### 仓库
```bash
# 克隆前设置代理
export https_proxy=http://127.0.0.1:7890
git clone https://github.com/cutefishos/libcutefish.git ~/libcutefish
```

### 移植策略
- Qt5→Qt6，KF5→KF6
- NetworkManagerQt / ModemManagerQt 确认是否有 Qt6 版本（KF6 已包含）
- `Cutefish.Screen` 优先级低，可跳过

---

*最后更新：2026-04-13*
*当前进度：一期全部完成。下一步：libcutefish Qt6 移植，解锁网络三件套和 User 页面。*
