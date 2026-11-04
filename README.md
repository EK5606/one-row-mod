# one-row-mod

一个独立的 ZMK module，提供 **8 个 home-row-mod 键**：左手 4 个（Shift / Alt / Gui / Ctrl），右手 4 个（Ctrl / Gui / Alt / Shift）。

它实现的是一套 kanata 风格的 HRM 语义（同手 HRM 的 one-shot 改成了“锁定到 hold 松开”），不是 ZMK 默认的 `hold-trigger-key-positions`：

- 同手普通字母 → 立即 tap
- 异手键 / 空格 / 回车 / Tab / 方向键 / 数字 / Esc 等 → 立即 hold
- 单独按住 1 秒 → hold
- 同一手已经有一个 HRM 处于 hold 时，点按另一个 HRM → 那个 HRM 的 **mod 被按下并锁定**；后续键都带上它，直到该手所有 HRM hold 松开才释放
- 反向序列：
  - 左 Gui → 左 Ctrl = Backspace
  - 左 Alt → 左 Gui = Tab
  - 右 Gui → 右 Ctrl = Enter
  - 右 Alt → 右 Gui = Delete
- 连击窗口 200ms（同一 HRM 200ms 内连按仍是字面字母）

**不需要修改 ZMK 源码。** 这个仓库是一个外部 module：新增了一个 behavior 驱动、DT binding 和 8 个 behavior 实例。驱动本身是从 ZMK 的 `behavior_hold_tap.c` 复制并改写的（MIT，保留了版权声明），所以它不是“零代码配置”，但完全不碰上游文件。

---

## 目录

```
one-row-mod/
├── zephyr/module.yml
├── CMakeLists.txt
├── Kconfig
├── dts/
│   ├── bindings/behaviors/
│   │   ├── zmk,behavior-hrm-hold-tap.yaml
│   │   └── zmk,hrm-layout.yaml
│   └── one-row-mod/hrm.dtsi          # 8 个 behavior 实例
├── src/behavior_hrm_hold_tap.c
├── examples/
│   ├── lunar-paw-layout.dtsi     # lunar-zmk / paw 的精确 position
│   ├── lunar-paw-keymap-snippet.dtsi
│   ├── lunar-paw.patch
│   └── pskeeb5-layout.dtsi
└── LICENSE
```

---

## 安装

### 方式 A：作为 west project（推荐，GitHub Actions 可用）

把本仓库推到 GitHub，例如 `https://github.com/EK5606/one-row-mod`，然后在 `config/west.yml` 里加：

```yaml
manifest:
  remotes:
    - name: ek5606
      url-base: https://github.com/EK5606
  projects:
    - name: one-row-mod
      remote: ek5606
      revision: main
    # ... 其它项目
  self:
    path: config
```

lunar-zmk 的 `config/west.yml` 里已经有 `ek5606` remote，直接加 project 即可。

### 方式 B：本地 build（还没推到 GitHub 时）

```bash
export ZMK_EXTRA_MODULES=/home/arcpyris/Workspace/one-row-mod
west build ...
```

或者在 `build.yaml` 的对应条目里加：

```yaml
include:
  - board: nice_nano//zmk
    shield: paw_left nice_view
    snippet: nrf52840-nosd studio-rpc-usb-uart
    cmake-args: -DZMK_EXTRA_MODULES=/home/arcpyris/Workspace/one-row-mod
```

> 本地绝对路径只适合本机；GitHub Actions 构建时需要方式 A。

---

## 使用

### 1. include

在 `config/paw.keymap` 里，**放在 `<behaviors.dtsi>` 后面**：

```c
#include <behaviors.dtsi>
#include <one-row-mod/hrm.dtsi>
```

### 2. 定义 hrm_layout

在 keymap 里加一个 `hrm_layout` 节点。position 是 `bindings` 数组的扁平下标（从 0 开始），和 combo 的 `key-positions` 是同一套数字。

lunar-zmk / paw 的 44 键布局已经算好，直接复制 `examples/lunar-paw-layout.dtsi`：

```dts
/ {
    hrm_layout: hrm_layout {
        compatible = "zmk,hrm-layout";

        left-shift-position = <13>;   /* A */
        left-alt-position = <14>;     /* S */
        left-gui-position = <15>;     /* D */
        left-ctrl-position = <16>;    /* F */

        right-ctrl-position = <19>;   /* J */
        right-gui-position = <20>;    /* K */
        right-alt-position = <21>;    /* L */
        right-shift-position = <22>;  /* ; */

        left-tap-positions = <1 2 3 4 5 17 25 26 27 28 29>;
        right-tap-positions = <6 7 8 9 10 18 30 31 32 33 34>;
    };
};
```

`left-tap-positions` / `right-tap-positions` 只列**同手非 HRM 字母**，不要列 HRM 键、空格、拇指键等；其它键默认按 hold 处理。

lunar-zmk 的完整 keymap 改动已经生成成补丁：

```bash
cd /home/arcpyris/Workspace/lunar-zmk
git apply --check /home/arcpyris/Workspace/one-row-mod/examples/lunar-paw.patch
git apply         /home/arcpyris/Workspace/one-row-mod/examples/lunar-paw.patch
```

补丁只改 `config/paw.keymap`：加 include、加 `hrm_layout`、把 8 个 `&hml/&hmr` 换成 `&hrm_*`。旧的 `hml`/`hmr` 定义会保留（不再被使用），确认新行为正常后可以删掉。

另外，lunar 现有的 `equal_gt` 宏用的是 `&kp EQUAL`；按住 Shift 时会变成 `+`，所以 `=>` 实际会输出 `+>`。要完全复刻 kanata，把它改成 `&kp KP_EQUAL`（Keypad Equal 不受 Shift 影响）。

### 3. 替换 keymap 里的 8 个键

绑定顺序和 `&mt` / `&lt` 一样：**先 hold，后 tap**。

```dts
&hrm_lsft LEFT_SHIFT A
&hrm_lalt LEFT_ALT S
&hrm_lgui LEFT_GUI D
&hrm_lctl LCTRL F

&hrm_rctl RCTRL J
&hrm_rgui RIGHT_GUI K
&hrm_ralt RIGHT_ALT L
&hrm_rsft RIGHT_SHIFT SEMICOLON
```

lunar-zmk 里原来那 8 个 `&hml` / `&hmr` 替换成上面的即可。旧的 `hml` / `hmr` behavior 定义可以删掉（不再使用）。

---

## 位置编号怎么数

以 lunar-zmk 的 default layer 为例：

```
row 1:  0 TAB   1 Q   2 W   3 E   4 R   5 T   6 Y   7 U   8 I   9 O  10 P  11 DEL
row 2: 12 ESC  13 A  14 S  15 D  16 F  17 G  18 H  19 J  20 K  21 L  22 ;  23 ENT
row 3: 24 LSFT 25 Z  26 X  27 C  28 V  29 B  30 N  31 M  32 ,  33 .  34 /  35 WIN
thumb: 36 ENT  37 SPC 38 mtl 39 ENT 40 SPC 41 mtl 42 BSPC 43 none
```

拆分键盘也是同一个全局下标（左半 + 右半按 `bindings` 顺序排），不是行列坐标。

---

## 行为细节

| 场景 | 结果 |
|---|---|
| 按 `A` 后快速按 `Q` | `a` + `q`（同手字母立即 tap） |
| 按住 `A` 后按 `Y` | Shift + Y（异手立即 hold） |
| 按住 `A` 1 秒 | Shift hold |
| 按住 `A`，点 `S`，再按 `K`、`L` | Shift+Alt+K、Shift+Alt+L；Alt 保持到 `A` 松开 |
| 按住 `A`，点 `S`，松开 `A` | Shift 和锁定的 Alt 一起释放 |
| 左 `D` → 左 `F` | Backspace（Gui 被抑制，不会弹 Super） |
| 左 `S` → 左 `D` | Tab |
| 右 `K` → 右 `J` | Enter |
| 右 `L` → 右 `K` | Delete |
| 同一 HRM 200ms 内连按两次 | 两个字面字母 |
| 空格长按 | 走你自己的 `&lt` / 符号层（本 module 不改 space） |

物理修饰键（真正的 LShift/LCtrl 等）不参与 HRM 状态，所以 `Ctrl+A`、`Shift+S` 不会被影响。

---

## 注意事项

- **不要在同一块键盘上同时跑 ZMK HRM 和 kanata HRM**，否则会双重映射。用了这个 module 就把该键盘从 kanata 的 `devices` 里移除。
- 这个 module 的驱动是 ZMK `main`（2026-11）`behavior_hold_tap.c` 的改写版。ZMK 的 behavior API 变化时需要同步更新。
- 锁定的 mod 由驱动直接通过 `&kp` 按下/释放，不经过 ZMK 的 sticky key。锁定按手计：同一手有多个 HRM hold 时，要等最后一个 hold 松开才释放。
- 如果同一个 mod 也被物理键或其它 behavior 按住，释放锁定 mod 时可能清掉共享的 HID 位；这是边界情况，正常单源使用时没有影响。
- ZMK 同一时刻只允许一个 undecided hold-tap；本驱动依赖内置 hold-tap 的事件顺序，不要启用 `hold-while-undecided`。
- `tapping-term-ms = 1000`、`quick-tap-ms = 200` 定义在 `dts/one-row-mod/hrm.dtsi` 的 8 个实例里，要改就改那里，或者在你的 keymap overlay 里覆盖某个实例。
- 位置数组和 keymap 顺序强耦合；改 keymap 布局后要同步改 `hrm_layout`。

---

## License

MIT。驱动部分来自 ZMK（Copyright (c) 2020 The ZMK Contributors），见 `LICENSE`。
