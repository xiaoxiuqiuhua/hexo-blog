---
title: 当我想截个图，却折腾了三套快捷键
date: 2026-06-08 21:00:00
tags:
  - 工具
  - Claude Code
  - 截图
  - 自制
categories:
  - 折腾
cover: /images/posts/j-screenshot-tool/banner.webp
description: 用 Claude Code 手搓了一个截图工具，只为按 F2 就能截。
---

> 某天我只是想截一张图，愣是打开了三个软件、记住了三套快捷键……

## 一、起因：截图这件小事，怎么就这么难

我最近越来越爱截图了 —— 写博客要截、问 AI 要截、给同事吐槽 bug 也要截。可每次掏出截图工具的过程，都像在开"找钥匙"的盲盒：

- **豆包截图**：截倒是挺顺，但截完的图片只能发给豆包，再想发给 GPT、Claude、微信得先存到本地再上传。
- **Edge 截图**：必须先打开 Edge，哪怕我根本没在用浏览器。
- **微信截图**：偶尔手抖，截完直接给你"已发送给文件传输助手"，还顺手弹个窗。
- **Snipaste**：什么都好，就是有点"重"——贴图、标注、取色器、历史记录……我截图就只想截图。

这些工具的**快捷键又不一样**：豆包是 `Ctrl+Shift+A`、Edge 是 `Ctrl+Shift+S`、微信是 `Alt+A`、Snipaste 是 `F1`。记混了就在桌面上盲按三秒钟，烦。

**共同症结就一句话：快捷键不统一、入口不统一、结果还要再走一次"另存为"。**

于是有个念头越来越强：**自己搓一个呗，啥也别加，只截图，按 F2，按完就走。**

## 二、过程：和 Claude Code 的几轮对话

需求本身简单到一句话：

> 可以截电脑屏幕上的所有显示的内容，F2为快捷键，开启后直接进托盘。

就这三句。

我把这句话发给了 Claude Code。它没有急着动手，而是先问了我两件事：

1. 用什么语言？（我说 Python，最近的项目都是 Python）
2. 多屏要不要支持？高 DPI 要不要支持？（我说要，多屏用户必须的）

然后它给出了一个看起来过分整齐的方案：

- 入口：PySide6 做托盘 + 选区遮罩
- 热键：pynput 走 Windows `RegisterHotKey`，全局 F2
- 截图：Pillow 的 `ImageGrab`，`all_screens=True` 一把抓多屏
- 打包：PyInstaller `--onedir`，出 `Snap86.exe`

四个模块，每个二十几行，加起来不到 300 行 Python。

接下来的几轮对话才是有意思的。我没怎么写代码，全程在做这几件事：

- 跑一遍，**提"哪里不对"**：遮罩在第二块屏上没出来、`ImageGrab` 截下来颜色偏了、托盘图标莫名其妙消失……
- Claude Code 给原因 + 改法：第二块屏是 `HiDPI` 缩放要单独算、截图得转 `QImage.Format_RGB888` 不然画图粘出来发绿、托盘消失要设 `AppUserModelID` 让 Windows 把它当成正经应用。
- 我再说"还是不对"，它再改。

这种感觉怎么说呢 —— **不是"我一行行写代码"，是"我提需求 + 它实现 + 我说哪里不对"**。和我以前写代码的方式完全不一样，但效率出奇地高。

## 三、成果：J 截图工具

项目代号是 `snap86`（snap + 86 版的 86，因为我自己是 86 版五笔用户，名字顺手就用上了），产品名「J 截图工具」—— 名字里带 J 没别的意思，就是想让 `Win+J` 系列里有一个能让我自定义的快捷动作（虽然最后还是用了 F2）。

来一张特性表，看看它能做啥：

| 操作 | 结果 |
|---|---|
| 按 **F2** | 唤起全屏半透明遮罩 |
| 拖动鼠标 | 实时显示选区（挖空 + 蓝边 + 尺寸文字）|
| 松开左键 | 抓图 + 复制到剪贴板 + 托盘气泡「已复制 W×H」 |
| **Esc** / **右键** | 取消 |
| 选区 < 2 像素 | 视为取消 |
| 右键托盘 → 退出 | 真正退出进程 |

截完之后去画图、Word、微信、QQ、VSCode、Photoshop 任何地方，**Ctrl + V 直接粘出来**。不落盘、不弹主窗、不联网、不编辑。

**核心代码其实就两块**。第一块是全局热键（简化版，30 行内）：

```python
# pynput.keyboard.GlobalHotKeys 底层走 Windows RegisterHotKey,
# 跟 Qt 自己处理的 F2 不抢焦点;listener 在子线程里跑,
# callback 用 Qt Signal 把事件抛回主线程(emit 跨线程安全)
class GlobalHotkey(QObject):
    activated = Signal()

    def __init__(self, hotkey: str = "<f2>", parent=None):
        super().__init__(parent)
        self._listener = keyboard.GlobalHotKeys({hotkey: self._on_press})

    def start(self):
        self._listener.daemon = True  # 不卡主进程退出
        self._listener.start()

    def _on_press(self):
        # 在 pynput 子线程里被调,emit 是线程安全的
        self.activated.emit()
```

第二块是截图 + 写剪贴板（同样简化版）：

```python
def grab(sel) -> QImage | None:
    # all_screens=True 在 Win10/11 上把多屏拼成一张大图,
    # bbox 用虚拟桌面坐标即可(OverlayWindow.mapToGlobal() 给的就是这套)
    img = ImageGrab.grab(
        bbox=(sel.x, sel.y, sel.x + sel.w, sel.y + sel.h),
        all_screens=True,
    )
    rgb = img.convert("RGB")
    w, h = rgb.size
    # 复制一份独立 buffer(QImage 不会持有 PIL 的内存)
    data = rgb.tobytes("raw", "RGB")
    return QImage(data, w, h, w * 3, QImage.Format.Format_RGB888).copy()

def copy_to_clipboard(qimg: QImage) -> bool:
    cb = QApplication.clipboard()
    cb.setImage(qimg)  # Qt 同时写 PNG 和 DIBv5,主流程序 Ctrl+V 都能粘
    # 读回 size 简单校验一下(Qt 6 setImage 不会返回 bool)
    got = cb.image()
    return not got.isNull() and got.size() == qimg.size()
```

热键监听、遮罩选区、抓图、剪贴板，四个模块拼起来就完事了。打包用 `python build.py`（PyInstaller `--onedir`），产物在 `dist/Snap86/Snap86.exe`，双击启动后藏托盘，**日常用根本感觉不到它的存在**。

## 四、总结：这种小工具，自己搓才过瘾

做 J 截图工具大概花了我两个晚上加起来不到 4 小时。代码量小到可以一张截图发完，**真正的成本几乎全在"我提问题"上**。

最让我高兴的不是工具本身，而是"我自己搓的"这件事本身：

- 快捷键按我的习惯来 —— **F2 一按，截完就消失**。
- 选区 < 2 像素自动取消，因为我经常不小心多拖了一像素。
- 托盘气泡用「已复制 W×H」告诉我结果，没声音、没弹窗。
- 不存历史记录，因为截到一半又想截的图很罕见。

**这才是"自己的工具"该有的样子。** 如果用现成的，永远要妥协它的某种"贴心"。

更要紧的是门槛变了。以前我心里很清楚，要做这种工具，得先踩过这些坑：

- `pynput` 的子线程和 Qt 主线程不能直接调 UI
- `ImageGrab` 在多屏上 bbox 怎么算
- `QClipboard` 写完读不回来怎么校验
- Windows 托盘要 `AppUserModelID` 才不会被合并

**现在我只要会描述需求。** 这不是我代码水平提高了，是"提需求"和"评估结果"被解耦了 —— Claude Code 负责中间那条很长的路。

事实上这也不是我第一次这么干了。之前还搓过一个 [86 版五笔学习助手](blog.billnas.cn/tags/wubi86/)，连本博客本身都是用 Claude Code 配出来的。**小而具体的事，正在变得"自己做"更划算。**

下次再遇到一个不顺手的工具，**我大概不会再忍了**。

---

源码放在 `e:\Claude Code\snap86\`，暂未公开仓库。如果你也在 Windows 上被一堆截图工具的快捷键折腾过，可以照着这篇的思路自己搓一个 —— 真的不难。
