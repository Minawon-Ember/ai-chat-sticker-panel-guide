# 给 AI 聊天 App 做表情包：三种 App 各怎么做

> 因为鉴于有的朋友的前端App是原生的，有的朋友家的是网页套 Capacitor 壳子，还有很多人的 AI 住在 Telegram 机器人里。
> 这份把三种都写了：**先看第一部分（通用），再跳到你家那种。**

**先认一下你家是哪种：**

| 你家 App | 怎么认 | 看哪节 |
|---|---|---|
| 原生 iOS | 用 Xcode 写 Swift / SwiftUI，改界面要重新编译装机 | 第一部分 + 第二部分 |
| 网页壳子 | HTML / CSS / JS，套了 Capacitor（或类似的壳），改了代码刷新就生效，不用重新装 | 第一部分 + 第三部分 |
| 纯网页 / 添加到主屏幕 | 同上，只是没套壳 | 同网页壳子 |
| Telegram 机器人 | AI 在 TG 里跟你聊 | 第一部分 + 第四部分 |

**没有 Mac 也没关系**：第二部分要 Mac，第三、四部分全程不用。

---

## 第一部分：通用（三种都要做）

### 1. AI 怎么收发表情包：靠名字

模型看图要花 token，而且一张表情包的意思，往往不在画面里，在**你给它起的名字**里。

所以每张贴纸入库时必须起个名字（比如"委屈巴巴""盯——"），发消息时带上这个名字：

```json
{ "image": "/img/up-abc123.gif", "stickerName": "委屈巴巴" }
```

- **你发给 AI**：转给模型的文字里不放图，只放 `[表情: 委屈巴巴]`。模型秒懂，还不费 token。
- **AI 发给你**：给 AI 一个发表情的工具（命令行、function calling、或者约定它在回复里写 `[表情: 名字]` 由你的后端替换成图），按名字去库里找那张图发出去。名字写不全也行，先精确匹配、再包含匹配；再给一个"列出所有名字"的口子，AI 不知道有哪些时自己查。

**起名就是给 AI 的说明书。** 名字里可以带括号写画面，比如"一只猫猫等消息（配文：我是小三吗聊几句不聊了）"，AI 用得会更准。

### 2. 库：一个 JSON 就够

```json
{
  "categories": ["猫猫", "颜文字"],
  "stickers": [
    { "id": "…", "image": "/img/up-abc.gif", "name": "委屈巴巴", "category": "猫猫" },
    { "id": "…", "text": "(｡•́︿•̀｡)", "category": "颜文字" }
  ]
}
```

- `category` 就是"贴纸包"的名字，分组靠它
- 整个库放服务器上一个文件，**整存整取、整库覆盖**。表情就几百条，做增量同步不划算，整份覆盖反而不会出错
- 客户端本地缓存一份，打开面板不用等网络
- "最近用过"只存本地，丢了不心疼
- 加贴纸的流程：选图 → **先起名、挑包** → 再入库。不起名不让进

### 3. 动图

动图的格式有好几种（GIF / APNG / 动态 WebP / 实况那种 HEICS），建议**入库时一律转成 GIF**，后面所有地方只看扩展名 `.gif` 就知道是动图，不用先下载。

- 小 GIF（< 2.5MB、边长 ≤ 512）原样存，其余重新编码到 ≤ 400px、≤ 90 帧。贴纸显示最大也就 150pt 左右，400px 够了
- 帧太多就隔帧取，跳过那几帧的时长并进来，总时长不变
- 每帧时长 ≤ 10ms 的当 100ms（很多 GIF 写 0 表示"尽快"，浏览器也是这么处理的）

网页和 TG 显示 GIF 都是自带的，什么都不用做。原生 iOS 要自己播，见第二部分。

### 4. 面板长什么样（我们家的做法，照着 Telegram 贴纸页）

- 顶上一排圆：最近用过 / 每个包的封面 / 颜文字 / 加号。选中的圆有个环，切换时环滑过去（SwiftUI 用 `matchedGeometryEffect`，网页用一个绝对定位的环做 transform 动画）
- 下面 4 列网格，**裸图，不加格子底色**
- **轻点** = 发送
- **按住** = 放大预览，背后压暗 + 模糊
  - 拖到别的格子上松手 = 发那一张
  - 原地松手 = 钉住，出现 发送 / 改名 / 删除
  - 拖出面板松手 = 收起
- 长按顶上包的圆 = 改包名 / 删包
- 面板底色透明，直接坐在键盘区域的灰底上，看起来像系统键盘的一页
- 长按全部自己写手势，没用系统菜单

**发出去那一下：贴纸"飞"进聊天。** 拿格子在屏幕上的位置做起点，生成一个替身图飞到聊天底部，落地变成气泡。

**聊天里的贴纸气泡**：裸图、固定 150×150、贴着角落，不加气泡底色。固定尺寸是为了行高不跳：等图片加载完才知道尺寸的话，聊天列表会跳。

---

## 第二部分：原生 iOS（SwiftUI）

### 难点：切换时输入栏不能动

**失败的第一版**：藏一个看不见的输入框，它的键盘就是表情面板；点表情按钮时把焦点交给它。问题是两个输入框交接焦点那一下，系统会**先收键盘再弹**，输入栏沉下去再弹上来，还会透出候选栏残影。

**第二版：同一个输入框，原地换 `inputView`。**

UIKit 输入框的 `inputView` 设了以后，弹出来的就不是键盘而是你给的 view。关键是：

> **焦点一直在同一个输入框上，只换 `inputView`，再调 `reloadInputViews()`。**

系统在键盘窗口里原地换内容，不收不弹，输入栏纹丝不动，就是 Telegram 的效果。

SwiftUI 的 `TextField` 没暴露 `inputView`，要绕一下：在输入框的 `.background` 里放一个零尺寸 UIView 当"探针"，它挂进视图树后，顺着它往上找几层，再往下找到真正的 `UITextView` / `UITextField`：

```swift
final class Probe: UIView {
    weak var coordinator: Coordinator?
    override func didMoveToWindow() {
        super.didMoveToWindow()
        if window != nil { coordinator?.locateTextInput(from: self) }
    }
}

// 从探针往上走，每层的子树里找最宽的那个输入控件
func locateTextInput(from probe: UIView) {
    var node = probe.superview
    for _ in 0..<10 {
        guard let current = node else { break }
        if let found = findTextInput(in: current) { attach(found); return }
        node = current.superview
    }
}
```

切换：

```swift
func apply(active: Bool) {
    textView.inputView = active ? panelBox : nil
    if textView.isFirstResponder {
        textView.reloadInputViews()      // 正在打字：原地换
    } else {
        textView.becomeFirstResponder()  // 没聚焦：先换好再聚焦，弹出来就是面板
    }
}
```

面板用 SwiftUI 写，`UIHostingController` 包一层塞进去。

### 三个坑

**1. 面板高度被压成 0 / 被压矮**

`inputView` 的高度系统只认 `intrinsicContentSize`，SwiftUI 内容会被压矮。外面套一个容器，自己报告"我就是键盘那么高"：

```swift
final class PanelBox: UIView {
    var panelHeight: CGFloat = 302 {
        didSet { invalidateIntrinsicContentSize() }
    }
    override var intrinsicContentSize: CGSize {
        CGSize(width: UIView.noIntrinsicMetric, height: panelHeight)
    }
}
```

容器里**全程用 frame + autoresizingMask，别用 Auto Layout 约束**。约束会和键盘窗口强加的 frame 打架，打输了高度被解成 0，面板整个消失。

**2. 换回中文键盘，联想栏没了**

系统只换回键盘本体，联想栏要等输入框"再被碰一下"才长出来。在 `reloadInputViews()` 的**下一拍**替用户碰一下：

```swift
DispatchQueue.main.async {
    input.inputDelegate?.selectionWillChange(input)
    input.inputDelegate?.selectionDidChange(input)
    input.inputDelegate?.textWillChange(input)
    input.inputDelegate?.textDidChange(input)
}
```

必须是下一拍，同一拍真机没反应。保险起见盯着键盘高度，还比切换前矮就再 reload 一次，最多三次。

**3. 切换那一下聊天记录抖**

换回键盘时系统同一帧连报三次高度：336 → 308（没联想栏）→ 336。列表每次都跟着调就会抖。切换后 0.5 秒内只认最后一次高度。

**面板里不能放输入框**：一聚焦真键盘就把面板顶掉，所以"搜索贴纸"没法做在面板里。改名这类要打字的，回聊天页弹，弹完再把面板叫回来。

### 播动图

SwiftUI 的 `Image` 不会动。把帧解出来做成 `UIImage.animatedImage`，交给 `UIImageView` 播（走 Core Animation，不占主线程）：

```swift
struct AnimatedStickerView: UIViewRepresentable {
    let image: UIImage
    func makeUIView(context: Context) -> UIImageView {
        let v = UIImageView(image: image)
        v.contentMode = .scaleAspectFit
        // 尺寸完全听 SwiftUI 的框，别让图片固有尺寸撑开行高
        v.setContentHuggingPriority(.defaultLow, for: .horizontal)
        v.setContentHuggingPriority(.defaultLow, for: .vertical)
        v.setContentCompressionResistancePriority(.defaultLow, for: .horizontal)
        v.setContentCompressionResistancePriority(.defaultLow, for: .vertical)
        return v
    }
    func updateUIView(_ v: UIImageView, context: Context) {
        if v.image !== image { v.image = image }  // 同一张别重设，会从头播
    }
}
```

- `animatedImage` 只认等长帧，GIF 每帧时长不一样：取所有帧时长的最大公约数当一格，长帧重复放几次（同一个 UIImage 对象，不多占内存）
- 先显示静态首帧，动图解完再换上去，行高不跳
- 相册选图的 `PhotosPicker` 要设 `preferredItemEncoding: .current`，不然系统可能帮你转成静图

---

## 第三部分：网页壳子（Capacitor）/ 纯网页

**全程只改网页代码，不用 Mac，不用重新打包。**

### 网页做不到原生那种"原地换键盘"

网页没法把系统键盘换成你的面板。能做的是：**收起键盘，在同样的位置垫一块一样高的面板**。做好了输入栏几乎不动，只是切换时会闪一下，够用。

### 第一步：量键盘多高

不用装插件（装插件要重新打包），用浏览器自带的 `visualViewport`：键盘弹起来，可见区域会变矮，差值就是键盘高度。

```js
// 页面刚打开、键盘没弹的时候，记下完整高度
let fullHeight = window.innerHeight;
let keyboardHeight = 300; // 没量到之前先用个默认值

window.visualViewport.addEventListener('resize', () => {
  const h = fullHeight - window.visualViewport.height;
  if (h > 120) keyboardHeight = h;   // 小于 120 的多半是地址栏伸缩，不算
});

// 横竖屏切换后重新记
window.addEventListener('orientationchange', () => {
  setTimeout(() => { fullHeight = window.innerHeight; }, 300);
});
```

### 第二步：切换

**顺序很重要**：先把面板垫上，再收键盘。反过来的话，键盘收下去那一瞬间底下是空的，输入栏会掉下去再弹上来。

```js
const input = document.querySelector('#input');
const panel = document.querySelector('#sticker-panel');

// 键盘 → 面板
emojiButton.addEventListener('click', () => {
  panel.style.height = keyboardHeight + 'px';
  panel.hidden = false;      // 先垫上
  input.blur();              // 再收键盘
});

// 面板 → 键盘：点回输入框
input.addEventListener('focus', () => {
  // 别马上收面板：等键盘真的弹上来了再收，不然中间会空一下
  const done = () => {
    if (fullHeight - window.visualViewport.height > 120) {
      panel.hidden = true;
      window.visualViewport.removeEventListener('resize', done);
    }
  };
  window.visualViewport.addEventListener('resize', done);
  setTimeout(() => { panel.hidden = true; }, 600); // 兜底
});
```

**注意**：点表情按钮时，按钮本身别抢走焦点，不然键盘会先收一次。给按钮加：

```js
emojiButton.addEventListener('mousedown', e => e.preventDefault());
emojiButton.addEventListener('touchstart', e => e.preventDefault(), { passive: false });
```

（加了 `touchstart` 的 preventDefault，click 可能不触发，那就把切换逻辑挪到 `touchend` 里。）

### 第三步：面板里

- 网格用 CSS Grid，`grid-template-columns: repeat(4, 1fr)`
- 动图直接 `<img src="xxx.gif">`，浏览器自己会播
- 贴纸气泡**固定尺寸**（比如 150×150，`object-fit: contain`），别等图片加载完才知道多大，不然聊天列表会跳
- 长按预览：`touchstart` 开个 400ms 计时器，`touchmove` / `touchend` 取消；iOS 上记得给图片加 `-webkit-touch-callout: none; user-select: none;`，不然长按会弹系统的"存储图像"菜单

### 加贴纸

```html
<input type="file" accept="image/*" id="pick">
```

选完弹个小框起名、挑包，再上传到服务器、写进 JSON。上传 GIF 时别用 canvas 压缩（canvas 一画就只剩第一帧），直接传原文件，大的交给服务器端转。

---

## 第四部分：Telegram 机器人

TG 自己就有贴纸，面板、动图、预览全是现成的，**你只需要做第一部分的"名字"那一段。**

### 你发贴纸给 AI

机器人收到的贴纸消息长这样（只列有用的字段）：

```json
{
  "sticker": {
    "file_id": "CAACAgUAAxk…",
    "file_unique_id": "AgADxx…",
    "emoji": "😭",
    "set_name": "SomeCatPack"
  }
}
```

TG 自带的 `emoji` 往往不准（一只猫配个 😭 而已），所以自己维护一张表，用 `file_unique_id` 当钥匙：

```json
{
  "AgADxx…": { "name": "委屈猫猫（等你看出来）", "file_id": "CAACAgUAAxk…" }
}
```

- 收到贴纸：查表，有名字就给模型 `[表情: 委屈猫猫（等你看出来）]`；没名字退回 `[表情: 😭]`，再让你（或机器人自己问你）补个名字
- `file_unique_id` 是固定的，`file_id` 可能会变，所以钥匙用前者，发的时候用后者

### AI 发贴纸给你

按名字在表里找到 `file_id`，调 Bot API：

```
POST https://api.telegram.org/bot<TOKEN>/sendSticker
{ "chat_id": 你的 chat_id, "sticker": "CAACAgUAAxk…" }
```

表里的贴纸从哪来：你每发一张新贴纸，机器人就自动记下它的 `file_id`，起了名就入库。用得越多，AI 能用的越多。

---

## 给想做的人的顺序建议

1. **先做第一部分的"名字"**，不管哪种 App，AI 立刻就能用，效果最明显
2. 面板先用最笨的办法（弹个框）也行
3. 想要丝滑，再按你家那种去抠切换
4. 动图最后做

有问题可以来问～
