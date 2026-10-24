# 时间流逝无需资金 — Cultist Simulator 模组

让「时间流逝」不再消耗「资金」。

- 游戏版本：2026.1.g.2（Steam，AppID 718670）
- 游戏语言：简体中文（`config.ini` 中 `Culture=zh-hans`）
- 模组 ID（mods.txt 中的条目 = 文件夹名）：`no_funds_time_passes`
- 模组版本：1.1.0

## 它改了什么

### 1. 主体：时间流逝不再收费

原版的时间循环是这条 recipe 链，全部挂在 `actionid: "time"` 的 verb 上：

```
drawseason（抽一个时节事件牌）
   └─ linked → needs（「时间流逝」，warmup: 60 秒）
                 ├─ slots: Expenses —— greedy，required {funds: 1}，自动抓走一张资金
                 └─ linked → discardseason
                               └─ effects: { season: -1, funds: -999 }  ← 销毁资金
                                      └─ linked → drawseason（循环）
```

`needs` 由 `_aspects.json` 里 `ingredient` 性相的 `induces` 触发；而「资金」元素带有
`ingredient` 性相，所以只要桌面上有资金，计时器就会被拉起（原版注释原文：
"This is so anything that generates Funds will always trigger the needs timer if it's not already running."）。

本模组覆盖了两个 recipe（`content/recipes/no_funds_time.json`），id 与核心内容相同，
因此只替换下列属性：

1. **`needs`**
   - `slots` 置空 → 不再有「开销」卡槽，不会抓走资金。
   - `alt` 列表中移除了 `starvation`（挨饿）。原版该分支的 `requirements` 是
     `{funds: -1}`，即「桌面上没有资金时」用「挨饿」顶替「时间流逝」，会扣 1 点健康并
     加 1 点「饥饿」。不删掉它的话，装了模组之后玩家常常身无分文，反而每分钟扣血。
     **这一改动会让「饥饿」失去来源，连带教士 DLC 的胜利条件不可完成——已由下面第 2 项
     补回。**
   - `warmup` 仍是 `60`：**一分钟的时长保留**，只是不再收费。
   - 其余 alt 分支（生病、疑心、野心、绝望、幻象、玫瑰、愉悦的一天等）与 `linked`
     全部原样保留，季节事件照常触发。
2. **`discardseason`**
   - `effects` 从 `{ "season": -1, "funds": -999 }` 改为 `{ "season": -1 }`，
     即只丢弃旧时节，不再销毁资金。

被移除的「开销」卡槽原文：`我得付商店老板的帐。不交钱就会挨饿。`

### 2. 补丁：把「饥饿」的来源改成闭环

## 教士 DLC 与「饥饿」

**问题**：教士（Priest）的胜利条件是集齐七道伤疤锁，判定在
`minorknockvictory`（`requirements: { openedlockscar: 6, lockscar: 1 }`）。七道锁各需要一种
「训诫」卡，一一对应：

| 伤疤锁 | 需要的训诫 |
|---|---|
| edge 刃 | `injury` 伤口 |
| forge 铸 | `compensationb` 青铜币 |
| **grail 杯** | **`hunger` 饥饿** |
| heart 心 | `vitality` 活力 |
| lantern 灯 | `erudition` 博识 |
| moth 蛾 | `glimmering` 微光 |
| winter 冬 | `affliction` 病痛 |

而全游戏产出 `hunger` 的地方**只有一处**：`time.json` 里 `starvation` 的
`effects: { health: -1, hunger: 1 }`。删掉挨饿分支，杯之伤疤锁就永久做不出来。

**为什么不能简单把 `starvation` 放回去**：

1. 本模组下资金只会越攒越多，「恰好没钱」反而很难达成，教士可能照样触发不了；
2. 更严重的是 `hunger` 元素带 `xtriggers: { "restorehealth": "health" }` —— 拿饥饿去入梦
   （配一张资金当滋养）它就变成一张「健康」卡。免费饥饿 = 免费健康，全职业都能刷。

**做法**：改写原版「衰老 + 原料」的入梦结果，把「凭空产出」换成「闭环回收」。

原版这条链在 `core/recipes/dream_rest.json`：

```
dream.decrepitude            requirements {decrepitude:1, ingredient:1}   warmup 30
  linked → dream.decrepitude.ingredientheartf   真血路径：aspects {tohealth:1} → 变回「健康」【不动】
  linked → dream.decrepitude.failure            effects {ingredient:-1, erudition:1}
```

原料放进的是「衰老」元素**自带的 `consume` 卡槽**（`abilities.json` 里
`decrepitude.slots`，`actionid: dream`，`forbidden: poison`），所以 `ingredient` 才能满足
`requirements`——这一点也说明了为什么不能靠「桌面匹配」去做这个补丁。

模组覆盖了两个 recipe，都在 `content/recipes/no_funds_decrepitude.json`。第一个是**父配方**
（入口，真血与普通原料共用），第二个是**普通原料分支**：

```json
{
    "id": "dream.decrepitude",
    "startdescription": "红杯从不干涸。我将吞下这异质之物，忍饥聆听。"
},
{
    "id": "dream.decrepitude.failure",
    "startdescription": "我口舌生津，视线浮动；红杯开口了。[衰老与原料一并消耗，获得“饥饿”。]",
    "effects": { "ingredient": -1, "decrepitude": -1, "hunger": 1 }
}
```

即：**衰老与原料一并消耗，还原为一张「饥饿」**（原版是原料换「博闻」、衰老不变）。
「圣雅纳略的真血」那条路径**完全没有改动**——`linked` 分支、`effects`、`aspects`
一个字都没动。

**关于「初始描述」**：`dream.decrepitude` 是两条分支共用的入口，玩家在 30 秒 warmup 期间
看到的就是它的 `startdescription`（原版是「我的灵魂像沼泽鬼火一样……」）。所以想改「一开始
显示的那句」，必须改父配方而不是 `.failure`。正因为两条路径共用，这里用的是路径中性的
说法（「这异质之物」），真血路径读到它也不违和；真血路径自己的两段文案（「闻到瓶中血微弱
的锈蚀气味……」／「圣雅努略已打开了岁月之门……」）保持原样。

**为什么这样能堵住刷健康**：饥饿与衰老构成闭环——

```
饥饿 --(60 秒内不进食)--> 衰老 --(衰老 + 原料)--> 饥饿
```

每次想把饥饿兑现成健康（`hunger` 的 `xtriggers.restorehealth: "health"`，配一张资金当
滋养），都必须先烧掉一张「衰老」；而衰老本身就是过去失去的健康，数量有限、无法凭空产出。
最终代价是：一张衰老 + 一张原料 + 一张资金 → 一张健康。也可以理解为「用一份异质之物
赎回一道旧伤」。

**附带影响（刻意的取舍）**：这条改动对所有职业生效——原版「衰老 + 普通原料 → 博闻」
变成「衰老 + 普通原料 → 饥饿」。好处是**不依赖教士 DLC、不碰任何核心 verb、不需要额外
卡槽**，也对得上红杯的设定。

**已知文案小瑕疵（决定不改）**：`decrepitude` 元素自带的说明写着
「Decrepitude remains until you die」，这条改动之后不再准确。**刻意保留原样**：`decrepitude`
是通用元素，改它的说明会波及所有职业的提示文本，收益不抵风险。作者已确认知悉。

**踩过的坑（留给以后改这个模组的人）**：CS 的 `requirements` **只匹配「放进卡槽的卡」，
不匹配整张桌面**。曾经试过用 `requirements: {priestjob: 1}` 来识别教士局，但入梦卡槽只
接受 `ability / desire / illhealth / way / funds`，而 `priestjob` 的性相只有 `job`，
根本放不进去——配方永远不匹配，游戏里毫无反应，日志表现为
`prediction … updated to 'dreamhealth'`。同理，桌上**已经存在**的卡要用 `extantreqs`
检查，不能用 `requirements`。

## 为什么 loc 里也写了一份完整覆盖

游戏的内容加载顺序是：

1. 核心 content
2. 已启用模组的 content
3. 核心 loc（当前语言）
4. 已启用模组的 loc

核心的 `content/loc_zh-hans/recipes/time.json` 并不是只有文本，而是**整份镜射**，
里面同样带着 `slots`、`alt`、`effects`。如果 loc 阶段会套用结构属性，那么第 3 步会把
第 2 步删掉的「开销」卡槽重新装回去，也会把 `dream.decrepitude.failure` 的 `effects`
改回原版。因此模组的 loc 文件里也写了同样的文本 + 结构覆盖，无论 loc 只套用可本地化
属性（此时多余字段被忽略）还是套用全部属性，最终结果都正确。

`loc/` 的确切层级在官方文档里没有写明，所以同一份文件放了三个候选位置：

- `loc/loc_zh-hans/recipes/<name>.json`
- `loc/recipes/<name>.json`
- `loc_zh-hans/recipes/<name>.json`

命中的那一个生效，未命中的会被忽略（同一份覆盖重复套用是幂等的）。

## 文件

```
no_funds_time_passes/
├─ synopsis.json                             模组元信息（名称 / 作者 / 版本 / 描述）
├─ cover.png                                 创意工坊封面，512×512（见下节）
├─ workshop_description.txt                  创意工坊描述（中英双版，复制粘贴用）
├─ LICENSE.md                                授权说明（Sixth History 社区许可）
├─ sixth-history-logo.png                    许可要求的标志
├─ serapeum_catalogue_number.txt             ⚠ 游戏生成的「创意工坊条目 id」，仅存在于已安装副本，勿删
├─ content/recipes/no_funds_time.json        覆盖 needs、discardseason
├─ content/recipes/no_funds_decrepitude.json 覆盖 dream.decrepitude.failure
├─ loc/loc_zh-hans/recipes/time.json         中文文本（含同样的结构覆盖，见上文）
├─ loc/loc_zh-hans/recipes/no_funds_decrepitude.json
├─ loc/recipes/…                             loc 路径候选 B（同上两份）
└─ loc_zh-hans/recipes/…                     loc 路径候选 C（同上两份）
```

## 安装

模组文件夹放到存档目录下的 `mods/`：

```
%USERPROFILE%\AppData\LocalLow\Weather Factory\Cultist Simulator\mods\no_funds_time_passes\
```

然后在同一层目录的 `mods.txt` 里写一行文件夹名：

```
no_funds_time_passes
```

`mods.txt` 是启用列表（加载顺序按行序）。也可以启动游戏后在
Options → Mods（选项 → 模组）里勾选启用。切换启用状态后需要重启游戏。

## 上传 / 更新创意工坊

**关键文件：`serapeum_catalogue_number.txt`。** 里面只有一行数字（10 字节），是
**游戏首次上传时自动生成**的，内容就是 Steam 创意工坊条目的 id。游戏靠它决定该
「更新已有条目」还是「新建条目」：

- 文件在 → mod 面板给出**更新**，内容推到原来那条；
- 文件不在 → 只给**上载**，于是每次都会**新建一个条目**（订阅者、评分、评论全留在旧条目上）。

**千万不要删除或整体重建 mod 文件夹**——删掉这个文件等于丢掉条目绑定。更新内容时只覆盖
改动过的文件。如果已经丢了，可以从 Steam 自己的日志里找回原 id：

```
<Steam>\logs\workshop_log.previous.txt
[2026-10-12 20:29:35] [AppID 718670] Create new workshop item of type Community for AppID 718670 : 3818478340 (OK)
```

把那个数字写回 `serapeum_catalogue_number.txt`（**纯数字、无换行、无 BOM**）即可恢复更新能力。

本模组的条目情况（2026-10-14 记录）：

| 条目 id | 状态 |
|---|---|
| **3818478340** | 2026-10-12 20:29 创建，正式条目，`serapeum_catalogue_number.txt` 现在指向它 |
| 3819628128 | 2026-10-14 19:55 因绑定丢失而误建的重复条目，需在 Steam 上单独删除 |

## 卸载

- 在 Mods 界面取消勾选，或
- 从 `mods.txt` 删掉那一行，或
- 直接删除 `mods/no_funds_time_passes` 文件夹。

都不影响已有存档，也不修改任何游戏本体文件。

## 其他语言

如果游戏语言不是简体中文，核心的 `loc_de` / `loc_es` / `loc_fr` / `loc_jp` / `loc_ru`
同样是完整镜射。若那些文件会覆盖结构属性，其他语言下需要为对应语言各加一份
`loc/loc_<culture>/recipes/*.json`（内容与 zh-hans 那份相同，只是文本换成该语言）。

## 创意工坊封面与上传

游戏本体（`SecretHistories.Main.dll`）里的判定原文是：

> The mod needs a 100x100 PNG file named 'cover.png'

也就是上传创意工坊要求 mod 根目录下有一个 PNG 叫 `cover.png`；缺了它 `Player.log` 会报
`has no preview image; can't upload it to Workshop`。

**尺寸不要用 100×100。** Steam 会按预览图的**原生像素**显示，100×100 的图会被塞进
约 150–230px 的格子中央、四周留黑边（实测过）。而游戏本体对尺寸并不校验：把
`cover.png` 换成 512×512 后，日志里那句 `has no preview image` 就消失了。所以本模组
内置的是 **512×512**（另有 1024×1024 备选，供 HiDPI 使用）。

封面（已定稿）：

- 左上：游戏内「时间流逝」的**橙色**小时漏图标；右下：资金金币图标，上面压一个红色禁行标志
- 两块沿画布对角线相接、中间无间隔，重叠 20%（相对时间图标边长）
- 资金图标在上层，带投影；🚫 为矢量绘制，环圈收细以便金币透出
- 背景：程序化生成的深木纹（扭曲正弦木纹＋细孔噪声）

美术素材取自 Cultist Simulator 英文 wiki：`File:Time icon.png`（黄色小时漏变体）、
`File:Funds.png`（金币＋木纹，256×256）、`File:Timepasses.png`（游戏内框的截图，
用于取框体配色、并量出 verb 图标 : 资金卡 ≈ 1 : 0.39 的比例）。
生成脚本为工作区里的 `workshop_cover.py`，尺寸、配色、遮挡程度都是脚本参数。

## 授权 / Licence

本模组是《密教模拟器》的**非官方**内容，依 Weather Factory Ltd 的
[Sixth History 社区许可](https://weatherfactory.biz/sixth-history-community-licence/) 发布；
与 Weather Factory Ltd、Secret Histories 及任何相关官方内容**无隶属关系**，未获其背书。

![Sixth History](sixth-history-logo.png)

完整的合规说明、著作权归属与美术素材出处见 [LICENSE.md](LICENSE.md)。
