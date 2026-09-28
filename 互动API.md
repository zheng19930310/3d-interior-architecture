# 互动 API 调用说明（window.Room3D）

页面里的所有互动（开灯、开电视、开水、翻马桶盖、抽书、进房间、切换日夜……）都已经从事件处理里抽出来，
统一挂到 `window.Room3D` 上。你可以在浏览器控制台、宿主页面、iframe 父级，或任意外部按钮 / 语音助手 /
后端指令里直接调用，不需要再模拟点击。

- 文档版本：`Room3D.version === '1.1.0'`（1.1.0 新增第九轮的抽出 / 展开 / 换位方法与 `book` 事件；1.0.0 的方法全部保留）
- 对象文件：`index.html`（源码）/ `客厅3D效果图.html`（单文件成品，内网离线可直接打开）
- 本文档最后一节给了 7 个**多方法联动**的集成示例，可以直接复制改参使用。

---

## 0. 怎么拿到 Room3D

脚本是 `type="module"`，异步加载完才注册 API。轮询或用 ready 事件二选一：

```js
// 方式 A：等事件（推荐）
document.addEventListener('room3d:ready', (e) => {
  const Room3D = e.detail;          // 等同 window.Room3D
  Room3D.setNight(true);
});

// 方式 B：轮询兜底
function ready(cb){
  if (window.Room3D) return cb(window.Room3D);
  const t = setInterval(() => { if (window.Room3D) { clearInterval(t); cb(window.Room3D); } }, 50);
}
ready(Room3D => Room3D.focus('sofa'));
```

**iframe 集成**（把成品页嵌进你的系统）：

```html
<iframe id="room" src="客厅3D效果图.html" style="width:100%;height:600px;border:0"></iframe>
<script>
  const api = () => document.getElementById('room').contentWindow.Room3D;
  document.getElementById('btn-night').onclick = () => api().setNight(true);
</script>
```

> 坐标单位：米。相机使用右手坐标系，`+x` 朝东、`+z` 朝南、`+y` 向上。地面 `y = 0`，抬高地台面 `y ≈ 0.46`。

---

## 1. 方法总览（45 个方法 + 5 个底层对象）

| 分组 | 方法 | 返回 |
|---|---|---|
| 场景信息 | `listItems()` `getItem(ref)` `getMesh(ref)` `listZones()` | 见 §3 |
| 部件开关 | `listParts(item?)` `getPart(id)` `setPart(id, value?)` `isPartOn(id)` | 见 §4 |
| 书架·书目 | `books()` `book(ref)` `openBook(ref, open?)` `closeBook(ref)` `toggleBook(ref)` `closeAllBooks()` `openRandomBook(n?)` | 见 §5 |
| 书架·四段互动 | `bookState(ref)` `pullBook(ref)` `unfoldBook(ref)` `foldBook(ref)` `shelveBook(ref)` `moveBook(ref, row, slot)` | 见 §5 |
| 书架·层位查询 | `shelfSlots(row)` `shelfCount(row)` `shelfFree(row)` `activeBook()` `frameBook(ref, dur?)` | 见 §5 |
| 相机 / 导航 | `enterRoom(zoneId)` `goOverview()` `getZone()` `focus(ref)` `flyTo(pos, target, dur?)` `screenToRoom(x, y)` | 见 §6 |
| 选中态 | `select(ref)` `deselect()` `getSelected()` | 见 §7 |
| 灯光 / 氛围 | `setNight(on)` `toggleNight()` `isNight()` `setAutoRotate(on?)` `setExposure(v)` | 见 §8 |
| 屋顶 / 分区 | `setRoof(on)` `setZoneOverlay(on)` | 见 §8 |
| 事件 | `on(ev, fn)` `off(ev, fn)` `emit(ev, data)` | 见 §9 |
| 底层 | `THREE` `scene` `camera` `renderer` `controls` | 见 §10 |

参数里的 `ref` 一律支持三种写法（详见各节）：**物品 id 字符串** / **THREE.Object3D** / **`getItem()` 返回的对象**。
所有方法对未知 id 都返回 `null` / `false`，**不抛异常**，方便外部无条件调用。

---

## 2. 物品 id 清单（17 个，`listItems()` 的固定结果）

| id | 中文名 | 所在分区 | 可互动部件 |
|---|---|---|---|
| `deck` | 抬高地台 · 木平台 | deck | — |
| `blinds` | 木百叶帘 | deck | — |
| `shelf` | 木质开放书架 | deck | `book-0` … `book-130`（131 本） |
| `branches` | 插花枝条 | deck | — |
| `floor-lamp` | 布艺落地灯 | deck | `lamp` |
| `rug` | 羊毛短绒地毯 | living | — |
| `sofa` | 模块化布艺沙发 | living | — |
| `chair-stripe` | 条纹异形单椅 | living | — |
| `pouf` | 圆形条纹坐墩 | living | — |
| `table` | 圆形玻璃茶几 | living | — |
| `chair-cane` | 藤编木质休闲椅 | living | — |
| `side-table` | 实木边几 | living | — |
| `tv` | 大尺寸壁挂电视 | tv | `tv` |
| `fireplace` | 微水泥地台 / 壁炉 | tv | — |
| `spotlights` | 嵌入式射灯 | living/deck/tv | — |
| `bath` | 浴室卫生间 | bath | `shower` `tap` `toilet-lid` `flush` |
| `closet` | 步入式衣帽间 | closet | — |

分区 id（`listZones()`）：`living` 会客区、`deck` 地台阅读区、`tv` 影音墙、`bath` 浴室卫生间、`closet` 衣帽间。
其中 `bath` / `closet` 是 `doll: true`，进入后自动用娃娃屋剖切机位（掀掉天花从斜上方看进去）。

---

## 3. 场景信息

```js
Room3D.listItems()                    // ['deck','blinds','shelf', ...] 17 个 id
Room3D.getItem('sofa')                // { id:'sofa', name:'模块化布艺沙发', tag:'家具 / FURNITURE',
                                      //   desc:'米白亚麻模块沙发…', zones:['living'] }
Room3D.getMesh('sofa')                // THREE.Group（可直接算包围盒 / 加挂点）
Room3D.listZones()                    // [{ id:'living', name:'会客区', doll:false }, ...]
```

`getItem()` 的 `desc` 就是信息卡上那段文案，做外部 UI（侧栏、语音播报）时可以直接复用，不用另写一份。

---

## 4. 部件开关：`setPart(id, value?)` —— 最常用的一个

```js
Room3D.setPart(id, value?)   // value 省略 = 取反；显式传 true/false = 绝对设置
Room3D.isPartOn(id)          // true / false；未知 id → null
Room3D.getPart(id)           // { id, item, kind:'switch'|'pulse', value }
Room3D.listParts()           // 全部 137 个部件 id（2 灯/电视 + 4 卫浴 + 131 本书）
Room3D.listParts('bath')     // ['shower','tap','toilet-lid','flush']
Room3D.listParts('shelf')    // ['book-0','book-1',...,'book-130']
```

**返回值是该部件的中文状态文案**（同时也是信息卡上显示的那句提示），失败返回 `null`：

| id | kind | `setPart(id, true)` | `setPart(id, false)` | 备注 |
|---|---|---|---|---|
| `lamp` | switch | `'开灯'` | `'关灯'` | 落地灯，带灯罩自发光 + 点光源淡入淡出 |
| `tv` | switch | `'开机'` | `'关机'` | 屏幕贴图 + 辉光 + 电视柜反射 |
| `shower` | switch | `'花洒放水'` | `'花洒关水'` | 开水时生成水滴粒子，阀体指示柄转 66°，关水自动清理 |
| `tap` | switch | `'龙头放水'` | `'龙头关水'` | 台盆龙头，把手上抬 40° + 水柱 + 盆内水面浮现 |
| `toilet-lid` | switch | `'掀起马桶盖'` | `'盖上马桶盖'` | 0.4s 绕后沿铰链翻起 |
| `flush` | pulse | `'冲水'` | `'冲水'` | **单次触发**：面板按下 0.12s 后回弹，水流约 1.4s 自动停止 |
| `book-N` | switch | `'抽出《书名》'` | `'归位《书名》'` | 只有两态（架上 / 手上）；展开、换位请用 §5 的书本方法 |

关键点：

1. `flush` 是 `pulse` 类型，调几次都是"触发一次冲水"，传 `false` 也照样触发，不要拿它当状态位。
2. 幂等：`setPart('lamp', false)` 连着调两次，第二次仍是 `'关灯'` 且不会重放动画（内部比对当前值）。
3. **API 与鼠标点击共用同一套状态机**。外部调用后，卡片上的开关状态、夜里灯亮不亮都是同步的；
   反过来用户手动点了灯，你的外部 UI 也能通过 `on('part')` 收到（§9）。
4. 动画是异步的（补间），`setPart` 立即返回文案；如果要"等动画结束"再截图 / 取状态，
   用 `await new Promise(r => setTimeout(r, 500))`，或监听 `part` 事件。

```js
Room3D.setPart('lamp');                 // 未指定 → 取反，返回 '开灯'/'关灯'
Room3D.setPart('shower', true);         // 强制开
Room3D.setPart('toilet-lid', true);
Room3D.setPart('flush');                // 触发一次冲水
if (Room3D.isPartOn('tv') === false) Room3D.setPart('tv', true);
Room3D.setPart('no-such-part');         // null，不报错
```

---

## 5. 书架：抽出 → 展开 → 合并 / 放回 / 换位

书架 5 层、共 **131 本**可点击的书。每本书是一个三态状态机：

```
                 pullBook / 点一次书              unfoldBook / 再点一次
   shelved ─────────────────────→ pulled ────────────────────→ open
   （在架上）   ←───────────────────┘   ←───────────────────────┘
              shelveBook / 「放回书架」        foldBook / 「合并」按钮
                      ↑
            moveBook(ref, row, slot) 插回新的层与位（自动 shelved）
```

| state | 含义 | 视觉表现 |
|---|---|---|
| `shelved` | 在架上 | 立在原层原位（或平放叠在别的书上） |
| `pulled` | 抽到视觉正面 | 从格架沿 +x 抽出 0.62m，越过侧框，封面转向相机 |
| `open` | 展开细看 | 由一个共用的"展开本"代演：左右两半叶绕书脊张开，**展开面法线严格指向相机**，镜头在哪书就朝哪翻开 |

**鼠标点击的循环是 `shelved → pulled → open → pulled → open …`**：点一下抽到正面，再点一下摊开，
再点一下合上（仍拿在手上）。"回到架上"由「放回书架」按钮或 `shelveBook()` 负责，不占点击次数。
方法之间不要求按顺序调用——`unfoldBook()` 对还在架上的书也有效，会一次性完成抽出 + 展开。

### 5.1 状态与四段动作

```js
Room3D.bookState(ref)                 // 'shelved' | 'pulled' | 'open'；未知 → null
Room3D.pullBook(ref)                  // 架上 → 抽到视觉正面
Room3D.unfoldBook(ref)                // → 展开（已展开则幂等返回 null）
Room3D.foldBook(ref)                  // 展开 → 合并（仍拿在手上）；非展开态返回 null
Room3D.shelveBook(ref)                // → 放回书架（回到它当前该在的位置）
Room3D.activeBook()                   // 此刻"在手上/在展示"的书 id，没有 → null
Room3D.frameBook(ref, dur = 0.9)      // 把相机推到书面正前方，返回 true / false
```

返回值仍是**中文状态文案**（`'抽出《阴翳礼赞》'` / `'展开《阴翳礼赞》'` / `'归位《阴翳礼赞》'`），
与信息卡、下方操作条上显示的是同一句，可以直接播报或写日志。

**全局只允许一本在展示**。`unfoldBook(B)` 时若 A 还摊开着，A 会自动收回到 `pulled`（书本体重新可见），
不会两本书抢同一个"展开本"对象。

### 5.2 换位（选层 + 选该层第几位）

```js
Room3D.moveBook(ref, row, slot)
// → { ok:true, row:4, slot:2, units:38, msg:'《柔光》→ 第4层 第2位' }
// → { ok:false, msg:'第3层放不下，先减少该层的书' }
```

`row` / `slot` 都从 **1** 起。`slot` 超出该层现有位置数时自动插到末尾（内部 `clamp`）。
换位不是简单地挪一本，而是**整层重排**：

- 目标层第 `slot` 位及之后的所有单元依次**后移一位**，让出插入口；
- 来源层删掉这本书留下的空位，后面的书**前移补上**；
- 两边的书各自重新计算占位长度与间隙，动画同步进行（0.55s 缓入缓出）；
- 平放的书换位后仍是平放，立放的书仍是立放；
- **放不下就整次回滚**：两层数量不变，书还在原层原位，只返回 `ok:false`，不会留下悬空的动画。

换位成功后这本书直接进入 `shelved`（等于"插回去"），`activeBook()` 清空，页面下方的操作条随之收起。

### 5.3 层 / 位置查询（做"选层选位"UI 就靠这三个）

```js
Room3D.shelfSlots(2)      // [{ slot:1, kind:'v',    titles:['阴翳礼赞'] },
                          //  { slot:5, kind:'flat', titles:['空间的光','静物与白'] },   ← 一摞两本
                          //  { slot:25, kind:'flat',titles:['静物与白','住宅本质'] }, ...]
Room3D.shelfSlots(3)      // [{ slot:1, kind:'flat', titles:[…4 本] },
                          //  { slot:2, kind:'prop', titles:[] }]      ← 陶罐也占一个位置
Room3D.shelfCount(2)      // 26 —— 该层"位置"（占位单元）数，也是 moveBook 可用的最大 slot
Room3D.shelfFree(2)       // 3  —— 该层还能再塞几本（按最薄的书 + 间隙估算）
```

`kind` 三种：`'v'` 立放（1 本 1 个位置）、`'flat'` 平放叠摞（可含多本）、`'prop'` 装饰物（绿萝 / 陶碗，`titles` 为空数组）。
页面上的层位面板正是 `slot` 从 1 遍历到 `shelfCount(row)+1`（多出来的那一位 = 插到末尾）。

出厂摆位（未操作过时，`seed = 7` 确定性生成）：

| 层 | 位置数 | 书目数 | 还能放 | 构成 |
|---|---|---|---|---|
| 第 1 层 | 36 | 36 | 3 | 全立放 |
| 第 2 层 | 26 | 28 | 3 | 24 立放 + 2 摞平放 |
| 第 3 层 | 2 | 4 | 32 | 1 摞平放 + 1 件陶罐装饰 |
| 第 4 层 | 37 | 37 | 1 | 全立放（最挤） |
| 第 5 层 | 25 | 26 | 0 | 22 立放 + 2 摞平放 + 1 件陶碗装饰 |

### 5.4 书目表与兼容用的开关式方法

```js
Room3D.books()                          // [{ id:'book-0', index:0, title:'空间的光',
                                        //    open:false, state:'shelved', row:1, slot:1, units:36 }, ...] 131 条
Room3D.book(ref)                        // 单本书的同上结构；找不到 → null
Room3D.openBook(ref, open = true)       // open=true 抽出；false 归位；传 'open'/'unfold' 直接展开
Room3D.closeBook(ref)                   // 归位
Room3D.toggleBook(ref)                  // 取反
Room3D.closeAllBooks()                  // 全部归位，返回归位的本数
Room3D.openRandomBook(n = 1)            // 随机抽出 n 本（只在合着的书里随机），返回实际抽出数
```

`ref` 四种寻址方式，都认（`books()` 之外的书本方法均可）：

```js
Room3D.pullBook(0);                     // ① 数组索引（0-based）
Room3D.pullBook('book-12');             // ② 部件 id
Room3D.pullBook('阴翳礼赞');             // ③ 书名（20 个书名循环使用，命中第一个同名书）
Room3D.pullBook(Room3D.activeBook());   // ④ 当前在展示的那本
```

`open` 字段是第八轮留下的布尔位，等价于 `state !== 'shelved'`，老代码不用改。

书名共 20 种：空间的光 / 静物与白 / 住宅本质 / 材料的诚实 / 光影集 / 柔光 / 极简主义 / 造物 / 室外的房间 /
尺寸与比例 / 手作的温度 / 灰度 / 窗的知识 / 坐具论 / 陶与土 / 日常的诗意 / 阴翳礼赞 / 构筑文化 / 家的构造 / 留白。
要按标题批量操作，先 `books()` 拿全表再过滤：

```js
Room3D.books().filter(b => b.title === '光影集').forEach(b => Room3D.pullBook(b.id));
```

### 5.5 页面上那排按钮

书一旦抽出，画面下方会浮出一条操作栏：`《书名》 第N层 · 第M位  [展开书本] [放回书架] [更换书本位置]`。
点「更换书本位置」再展开一层选择面板：`插到第几层`（5 个按钮）→ `该层第几个位置`（`shelfCount+1` 个按钮）→
「插回这里」。它们内部调的就是上面这组方法，所以你用 API 驱动时，这排按钮和提示文案会**同步跟着变**；
反过来用户手动点按钮，你也能从 `book` 事件里收到同样的状态（§9）。

---

## 6. 相机与空间导航

```js
Room3D.enterRoom(zoneId)                // 'living' | 'deck' | 'tv' | 'bath' | 'closet'；未知 id → false
Room3D.goOverview()                     // 回到全屋俯瞰
Room3D.getZone()                        // 当前分区 id，俯瞰时为 null
Room3D.focus(ref)                       // 选中该物品并把相机推到合适的近景
Room3D.flyTo(pos, target, dur = 1.2)    // pos/target 可为 [x,y,z] 或 THREE.Vector3，dur 单位秒
Room3D.screenToRoom(x, y)               // 把屏幕坐标（CSS 像素，相对视口）转成命中的物品信息
```

```js
Room3D.enterRoom('bath');                              // 进浴室（自动娃娃屋剖切机位）
Room3D.focus('bath');                                  // 选中浴室并推近
Room3D.flyTo([2.6, 1.9, 1.2], [0, 0.9, -2.6], 0.8);   // 自定义机位，0.8 秒飞过去
Room3D.goOverview();
Room3D.screenToRoom(320, 480);                         // { id:'sofa', name:'模块化布艺沙发', tag:'家具 / FURNITURE' } | null
```

`flyTo` 是**缓入缓出的补间**，调用后由渲染循环每帧推进；飞行期间相机位置由动画接管，
用户此时的拖拽 / 缩放不会改变落点，动画结束（默认 `dur = 1.2` 秒，传 `0` 或负数回落到内置约 0.87 秒）后
即交还 OrbitControls。`focus(ref)` 内部用物品包围盒算距离与高度，比手写 `flyTo` 更稳，
做"跳转到某个家具"优先用它。

---

## 7. 选中态（信息卡）

```js
Room3D.select(ref)      // 弹出物品信息卡 + 描边高亮，返回 true / false
Room3D.getSelected()    // 当前选中的物品 id，未选中 → null
Room3D.deselect()       // 收起卡片
```

`select()` 对同一个物品重复调用会**切换**（再次调用等于取消），和点击行为一致。
`focus()` = `select()` + 相机推近，大多数"带解说的跳转"直接用 `focus()`。

---

## 8. 氛围：日夜、屋顶、自动旋转、曝光

```js
Room3D.setNight(on)             // true = 夜灯模式，返回最终 night；按钮文案自动同步
Room3D.toggleNight()            // 取反，返回当前 night 布尔值
Room3D.isNight()                // 是否夜间
Room3D.setAutoRotate(on?)       // 省略参数 = 取反，返回最终布尔值
Room3D.setExposure(v)           // 渲染曝光，返回生效值（默认 1.0，演示建议 0.9 ~ 1.3）
Room3D.setRoof(on)              // true = 盖上天花，返回最终可见性。进 doll 分区时内部会置 false 做剖切
Room3D.setZoneOverlay(on)       // 地面分区色块 + 名称标签（概览态才好看），返回最终可见性
```

注意：`setNight(true)` 只改环境光、天光板、天花灯带与射灯，**不会自动开落地灯或电视**——
它尊重用户手动留下的状态。想在夜里亮灯，得显式 `setPart('lamp', true)`（§11 示例 1 就是这么写的）。

---

## 9. 事件总线：让外部 UI 跟着场景走

```js
const off = Room3D.on(event, handler);   // 返回解绑函数
Room3D.off(event, handler);              // 也可以显式解绑
Room3D.emit(event, payload);             // 手动发（一般不需要，除非你加了自己的控件）
```

| event | payload | 触发时机 |
|---|---|---|
| `part` | `{ id, item, kind, value, label }` | 任一部件被切换（**点击和 API 调用都会发**） |
| `book` | `{ id, title, state, row, slot }` | 书本状态变化（抽出 / 展开 / 合并 / 归位 / 换位）；`row`、`slot` 从 1 起 |
| `select` | `{ id, name, tag, desc }` | 卡片弹出 |
| `deselect` | `{ id }` | 卡片收起（`id` 为被关闭的，可能 `null`） |
| `room` | `{ zone, name }` | 进入房间或回到俯瞰（俯瞰时 `zone = null, name = '全屋俯瞰'`） |
| `night` | `{ night }` | 日夜切换 |
| `*` | 上述所有，额外带 `event` 字段 | 万能监听，做日志 / 埋点很方便 |

`book` 与 `part` 是配对发出的：走 `setPart` / `openBook` / `closeBook` 这条开关式入口时，先收到 `book`
（新状态）再收到 `part`（`value = state !== 'shelved'`）。只关心书本的话听 `book` 就够，`row`/`slot` 会带上
换位后的新家，可以直接刷进你的外部 UI：

```js
Room3D.on('book', ({ id, title, state, row, slot }) => {
  document.getElementById('now-reading').textContent =
    `《${title}》 ${state === 'open' ? '展开中' : state === 'pulled' ? '已抽出' : '已归位'} · 第${row}层第${slot}位`;
});
```

```js
Room3D.on('part', ({ id, value, label }) => {
  document.querySelector(`[data-part="${id}"]`)?.classList.toggle('on', value);
});
Room3D.on('room', ({ zone, name }) => console.log('当前：', name));
const stop = Room3D.on('night', ({ night }) => sendToAnalytics(night ? 'night' : 'day'));
stop();   // 解绑
```

---

## 10. 底层对象（需要自己扩展时）

`Room3D.THREE / scene / camera / renderer / controls` 全部原样导出，
可以在外面挂辅助线、加热点、换材质、读 `controls.target`：

```js
const { THREE, scene, camera, controls } = Room3D;
const box = new THREE.Box3().setFromObject(Room3D.getMesh('sofa'));
console.log('沙发尺寸', box.getSize(new THREE.Vector3()));

// 挂一个自有热点
const pin = new THREE.Mesh(new THREE.SphereGeometry(0.03), new THREE.MeshBasicMaterial({ color: 0xff5533 }));
pin.position.set(0.2, 1.1, -0.4);
scene.add(pin);
```

---

## 11. 联动调用示例（6 个可直接落地）

### 示例 1 ｜ 一键"夜晚观影"

```js
function movieMode(){
  Room3D.setNight(true);                 // 先切日夜，环境光降下来
  Room3D.setPart('tv', true);            // 开电视
  Room3D.setPart('lamp', true);          // 落地灯做唯一氛围光
  Room3D.enterRoom('tv');                // 飞到影音墙
  Room3D.focus('tv');                    // 再推到屏幕近景
  Room3D.setAutoRotate(false);           // 别自己转
}
function leaveMovieMode(){
  Room3D.setPart('tv', false);
  Room3D.setPart('lamp', false);
  Room3D.setNight(false);
  Room3D.goOverview();
}
```

顺序有讲究：**先 `enterRoom` 再 `focus`**。`enterRoom` 会重置分区/屋顶可见性，
`focus` 只调相机；反过来会被 `enterRoom` 覆盖掉机位。

### 示例 2 ｜ 导览剧本（带解说词，逐步推进）

```js
const TOUR = [
  { zone:'deck',   text:'抬高地台与木百叶，先接受光。', act:()=>Room3D.focus('blinds') },
  { zone:'deck',   text:'落地灯是夜间的主体光。',        act:()=>Room3D.setPart('lamp', true) },
  { zone:'living', text:'沙发的朝向决定了会客区的重心。',act:()=>Room3D.focus('chair-stripe') },
  { zone:'tv',     text:'电视墙用微水泥延展，弱化体量。',act:()=>Room3D.focus('fireplace') },
  { zone:'bath',   text:'湿区独立，磨砂玻璃透光不透形。',act:()=>{ Room3D.setPart('toilet-lid', true); Room3D.setPart('flush'); } },
];

async function runTour(){
  for(const step of TOUR){
    Room3D.enterRoom(step.zone);
    await wait(900);                     // 等飞行动画
    step.act && step.act();
    say(step.text);
    await wait(2600);
  }
  Room3D.goOverview();
}
const wait = ms => new Promise(r => setTimeout(r, ms));
```

`say()` 可以接 `speechSynthesis`：

```js
function say(text){
  const u = new SpeechSynthesisUtterance(text); u.lang = 'zh-CN';
  speechSynthesis.cancel(); speechSynthesis.speak(u);
}
```

### 示例 3 ｜ 书架讲解：随机抽三本、逐本展开再全部收回

```js
async function bookshelfDemo(){
  Room3D.enterRoom('deck');
  Room3D.focus('shelf');
  await wait(1000);

  Room3D.openRandomBook(3);              // 随机抽 3 本到视觉正面
  const opened = Room3D.books().filter(b => b.state === 'pulled');
  opened.forEach((b, i) => console.log(`${i+1}. 《${b.title}》  第${b.row}层第${b.slot}位  id=${b.id}`));

  for(const b of opened){                // 逐本展开（后一本会自动收掉前一本）
    Room3D.unfoldBook(b.id);
    await wait(1800);
    Room3D.foldBook(b.id);
    await wait(500);
  }
  console.log('归位', Room3D.closeAllBooks(), '本');   // 全部收回
}
```

想按主题抽（比如同一书名全部抽出）：

```js
Room3D.books().filter(b => b.title === '阴翳礼赞')
              .forEach((b, i) => setTimeout(() => Room3D.pullBook(b.id), i * 180));
```

### 示例 4 ｜ 重新摆放：把一层的书挪到另一层，两边一起让位

`moveBook` 是页面上"更换书本位置"按钮的底层方法，配合 `shelfSlots` / `shelfFree` 就能整层重排：

```js
async function reflow(fromRow = 5, toRow = 3, n = 5){
  const all = Room3D.books();
  const titles = Room3D.shelfSlots(fromRow).flatMap(u => u.titles);   // 该层现有的书名（按位置顺序）
  const order = titles.slice(-n)                                      // 取末尾 n 本
    .map(t => all.find(b => b.title === t && b.row === fromRow))       // 同名书要按"层"锁定
    .filter(Boolean);

  for(let i = 0; i < order.length; i++){
    const res = Room3D.moveBook(order[i].id, toRow, i + 1);           // 插到目标层第 1、2、3… 位
    if(!res.ok){ console.warn(res.msg); break; }                       // 放不下就停，两层都不动
    await wait(150);                                                   // 等这一步的让位动画
  }
  console.log(`第${fromRow}层 ${Room3D.shelfCount(fromRow)} 个位置，第${toRow}层 ${Room3D.shelfCount(toRow)} 个位置`);
}
reflow(5, 3, 5);
```

每一步都是两层同时重排：目标层第 `i+1` 位及之后的书整体后移让出入口，来源层由后面的书前移补位，
所以视觉上就是"重新摆放"，而不是只挪走一本。想在同层内换序（比如把某本常读的固定到第 1 位），
把 `toRow` 传成 `fromRow`、再自定义 `order` 的排序即可。

外部只要听 `book` 事件就能画出实时进度：

```js
Room3D.on('book', ({ title, state, row, slot }) =>
  console.log(`[书架] 《${title}》 ${state} → 第${row}层第${slot}位`));
```

### 示例 5 ｜ 外部按钮面板：状态双向同步

HTML：

```html
<div id="panel">
  <button data-part="lamp">落地灯</button>
  <button data-part="tv">电视</button>
  <button data-part="shower">花洒</button>
  <button data-part="tap">龙头</button>
  <button data-part="toilet-lid">马桶盖</button>
  <button data-pulse="flush">冲水</button>
</div>
```

JS：

```js
document.getElementById('panel').addEventListener('click', e => {
  const btn = e.target.closest('button'); if (!btn) return;
  const id = btn.dataset.part;
  if (id) Room3D.setPart(id);                              // 取反
  else if (btn.dataset.pulse) Room3D.setPart(btn.dataset.pulse);
});

// 场景里点击 / API 调用都会回灌按钮状态
Room3D.on('part', ({ id, kind, value }) => {
  const btn = document.querySelector(`[data-part="${id}"]`);
  if (btn) {
    btn.classList.toggle('on', value);
    btn.setAttribute('aria-pressed', value ? 'true' : 'false');
  }
});
```

### 示例 6 ｜ 卫浴演示流程（进屋 → 放水 → 冲水 → 收尾）

```js
async function bathDemo(){
  Room3D.enterRoom('bath');              // doll 分区自动掀顶（roofGroup 隐藏）
  await wait(1100);

  Room3D.setPart('toilet-lid', true);
  Room3D.setPart('tap', true);           // 台盆放水
  await wait(2400);
  Room3D.setPart('tap', false);

  Room3D.setPart('shower', true);        // 花洒开
  await wait(2400);
  Room3D.setPart('shower', false);

  Room3D.setPart('flush');               // 脉冲：自动结束
  await wait(1600);
  Room3D.setPart('toilet-lid', false);
  Room3D.goOverview();
}
```

### 示例 7 ｜ 语音 / URL 参数驱动

把 URL 当轻量协议用，方便外部系统一条链接就跳到某个场景：

```js
// 客厅3D效果图.html?zone=bath&night=1&lamp=1&tv=1
// 或 客厅3D效果图.html?zone=deck&book=阴翳礼赞&open=1&move=3:2
const q = new URLSearchParams(location.search);
document.addEventListener('room3d:ready', ({ detail: R }) => {
  if (q.get('night') === '1') R.setNight(true);
  if (q.get('zone')) R.enterRoom(q.get('zone'));
  ['lamp','tv','shower','tap','toilet-lid'].forEach(id => {
    if (q.get(id) === '1') R.setPart(id, true);
    if (q.get(id) === '0') R.setPart(id, false);
  });
  const book = q.get('book');                       // ?book=阴翳礼赞 或 ?book=0
  if (book !== null){
    const ref = isNaN(+book) ? book : +book;
    R.pullBook(ref);
    if (q.get('open') === '1') R.unfoldBook(ref);   // ?open=1 直接摊开
    const mv = q.get('move');                       // ?move=层:位
    if (mv) R.moveBook(ref, ...mv.split(':').map(Number));
    R.frameBook(ref);                               // 镜头推到书面正前方
  }
  if (q.get('focus')) R.focus(q.get('focus'));
});
```

语音助手侧只要把意图映射成方法名即可，`setPart` / `enterRoom` / `focus` 三个方法能覆盖九成指令；
书本相关的意图（"把那本《柔光》挪到第三层第二个"）对应 `pullBook` + `moveBook(ref, 3, 2)` 两步。

---

## 12. 已知边界与注意事项

1. **补间是异步的**：`setPart` / `pullBook` 返回的是文案，不是最终几何状态。要做截图或状态断言，
   留 300–500ms，或监听 `part` / `book` 事件后再取。
2. **`enterRoom` 之后不要马上 `flyTo` / `frameBook`**：`enterRoom` 内部自己发起了飞行动画，两者会互相覆盖。
   需要自定义机位时，先 `enterRoom`，`await` 约 900ms，再 `flyTo` / `frameBook`。
3. **概览态下点击只在当前可见物品上生效**：`mode === 'room'` 时射线只与该分区内的物品求交，
   `screenToRoom()` 同样遵守这个过滤规则。书本在 `deck` 区，抽书前先 `enterRoom('deck')`。
4. **磨砂玻璃与水滴不可点击**：它们带 `userData.noPick`，所以花洒阀门不会被挡住；
   你自己往 `scene` 里加东西时，如果只是装饰，记得也打上 `noPick`，否则会抢射线。
5. **未知 id 一律静默**：`setPart('x')` 返回 `null`、`enterRoom('x')` 返回 `false`、
   `moveBook('x', 1, 1)` 返回 `null`。想要提示音/日志，自己判返回值，别指望异常。
6. **展开态下书本体网格是隐藏的**：`state === 'open'` 时页面上看到的是一本共用的"展开本"在代演，
   `book(ref)` 的 `open` / `state` / `row` / `slot` 仍指向真书。要拿真书的包围盒，
   先 `foldBook()` 收回，或读 `books()` 里的 `row`/`slot`，别对 `getMesh('shelf')` 里那本书做射线。
7. **同时只有一本在展示**：`unfoldBook(B)` 会把还在摊着的 A 自动收成 `pulled`；
   做"多本并排对比"需要先把前面的 `foldBook` / `shelveBook` 掉。
8. **换位放不下会整次回滚**：`moveBook` 返回 `{ ok:false, msg }`，两层数量与书本位置都不变，
   不会出现半截动画；别把它当"部分成功"处理。
9. **`slot` 是"位置"不是"第几本书"**：平放的一摞算 1 个位置（可能含 2–4 本），装饰物也占 1 个位置。
   要精确到书，用 `shelfSlots(row)` 看每个位置的 `titles`。
10. 单文件版 `客厅3D效果图.html` 已把 Three.js r160 一起打进去了，离线双击即可运行；
    API 行为与 `index.html` 完全一致。
