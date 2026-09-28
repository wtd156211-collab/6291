# 矩形装箱布局（从 0 实现）

起始环境只有本说明、`samples/**` 与 `.gitignore`，没有实现代码，实现要新写。Python 3.13、只用标准库；页面用原生 HTML/Canvas，无构建、无依赖；自测用 unittest、只读 `samples/**`。

## 一、范围

要做：在仓库根写一个矩形装箱引擎 `pack.py`（库与命令行入口共用一份实现），读入一批矩形与容器口径，算出每件摆在哪儿、要不要转方向；再按引擎输出生成一个自包含页面，放 `web/` 下，把布局按比例画出来，标出编号与朝向，并给出利用率与耗时。

不做：不规则形状与三维装箱；任意角度旋转、镜像与缩放摆放（只允许 0° 与 90°）；多容器、多页拼接与纸张选择；刀缝与工艺补偿；交互编辑与页面内重排；网络、CDN、第三方依赖与构建步骤；输入容错（`samples/**` 保证合法）。

## 二、口径与公式

### 2.1 坐标、旋转与间隙

- 一律整数坐标；原点在容器左上角，x 向右、y 向下；矩形占 `[x, x + w') × [y, y + h')`，`(x, y)` 是左上角。
- 旋转只允许 90°（宽高互换），用 `rotated` 标注；`rotatable=false` 的件不许旋转。
- 矩形之间、矩形与容器边界之间允许贴合（间隙 0），不要求刀缝；空档的上限见 2.4。

### 2.2 两种容器

- `fixed`：容器宽高由输入给出，全部矩形都要放进去。
- `auto`：不给容器尺寸，引擎自选正整数宽高；验收要求所选容器的面积不超过输入的 `max_area`。

### 2.3 利用率

    utilization = 矩形总面积 ÷ 容器面积

`fixed` 下它只由「是否全部装入」决定；`auto` 下的面积上限与利用率门槛等价（面积 ≤ max_area ⟺ 利用率 ≥ 总面积 ÷ max_area）。输出保留 6 位小数。

### 2.4 不变量（全部是硬要求）

1. **全覆盖**：每件恰好放一次；输出与输入逐条对应、顺序一致。
2. **不越界**：`0 <= x`、`x + w' <= W`、`0 <= y`、`y + h' <= H`；`w'`、`h'` 按朝向取。
3. **不重叠**：任意两件的内部不相交，允许边或角贴合。
4. **间隙上限**：`gap_max` 是输入给的非负整数（各组的取值写在 `samples/expected/*.txt` 里）。对每个整数 x ∈ [0, W) 取一条竖直扫描线，把与它相交的件按 y 升序排好，相邻两件之间的空隙（后一件的上沿 y 减前一件的下沿 y + h'）不得超过 `gap_max`；与 0 件或 1 件相交时无需检查，贴容器上、下边一侧的空白不算空隙。水平方向对每个整数 y ∈ [0, H) 同理。
5. **朝向合法**：`rotated` 只取 true/false，且不违反 `rotatable`。

### 2.5 确定性

同一输入连跑两遍（再换 `PYTHONHASHSEED=0/1/2` 跑），`solve` 的输出文件与 `page` 的页面除 `elapsed_ms` 读数外逐字节相同；不得依赖哈希随机化、容器遍历顺序、时钟、locale 或未固定的随机数。

## 三、数据结构与流程

引擎放仓库根的 `pack.py`，既是可 import 的库，也是命令行入口；库的公开 API 自定，但两条命令必须共用同一份布局结果。内存里按输入顺序保存矩形表；布局结果是每件的 `(id, x, y, rotated)`。流程不限：只要守住 2.4 的不变量与第五节的预算，排序与找落点的做法自定；挑得越细利用率越高也越慢，取舍自己掌握。

## 四、输入输出与文件格式

一律 UTF-8、LF、末行有换行；输入输出都是 JSON。

### 4.1 输入 `samples/cases/*.json`

```json
{
  "mode": "fixed",
  "gap_max": 8,
  "container": {"width": 240, "height": 160},
  "rectangles": [
    {"id": "r001", "width": 120, "height": 100, "rotatable": true}
  ]
}
```

- `mode` 取 `"fixed"` 或 `"auto"`；`fixed` 给 `container`（正整数宽高），`auto` 给 `max_area`（正整数）、不给 `container`。
- `gap_max` 见 2.4，`rectangles` 1~20000 条。
- `id` 在一份输入里唯一、非空；`width`、`height` 是正整数；`rotatable` 是布尔。

### 4.2 命令行

    python pack.py solve <输入.json> <输出.json>
    python pack.py page  <输入.json> <页面文件>

`solve` 写 4.3 的布局 JSON；`page` 用同一份布局写一个自包含页面，提交时放 `web/index.html`（用 `01` 号样例生成）。退出码 0 成功、1 输入不可用、2 用法错误；非 0 不写输出。

### 4.3 输出 JSON

```json
{
  "mode": "fixed",
  "container": {"width": 240, "height": 160},
  "utilization": 0.960000,
  "elapsed_ms": 12,
  "placements": [
    {"id": "r001", "x": 0, "y": 0, "rotated": false}
  ]
}
```

- `placements` 与输入 `rectangles` 逐条对应、顺序一致；`x`、`y` 是左上角整数坐标；`rotated=true` 表示宽高已互换。
- `utilization` 按 2.3 保留 6 位小数；`elapsed_ms` 是引擎自报的非负整数毫秒，只供页面显示，验收计时以外部队表为准。
- `auto` 的 `container` 就是引擎选定并实际使用的尺寸。

### 4.4 页面

`page` 写的页面数据内联在 `<script type="application/json" id="layout-data">`（容器、每件的 id 与摆放尺寸、坐标、朝向、利用率、耗时），不 `fetch`、不引外部资源、双击即开；页面里 Canvas 带 `id="layout"`，按比例画出容器与每个矩形，矩形标出编号与朝向，下方显示利用率与耗时（节点带 `data-metric="utilization"`、`data-metric="elapsed_ms"`）。页面只渲染引擎给的那份布局，不重排、不重算。

## 五、性能与验收口径

预算：单份输入最多 20000 件；验收机是 Windows 上的 WSL `python3`（Python 3.13 标准库口径），单核、冷启动计入：单次调用含读写不超过 20 s，峰值常驻内存不超过 512 MiB；每组样例的上限见 `samples/expected/*.txt`。

逐条核对：

1. 每组按同名 expected 的 `utilization_min` 与 6 条 `invariant` 逐条核对；公式与口径来自第二、三节。
2. 确定性：连跑两遍、换 `PYTHONHASHSEED=0/1/2` 再跑，输出与页面（除 `elapsed_ms` 读数外）逐字节相同。
3. 旋转：`05` 组不旋转就放不下；`rotatable=false` 的件任何时候都不能被转过。
4. 不许写死：不得读取 `samples/expected/**`，不得按样例名、id 或数量特判；评测会换矩形集合与容器，也会上 2 万件规模的数据。
5. 页面：`web/index.html` 双击可开，编号、朝向、利用率、耗时四项齐全，数字只能来自引擎那一次输出。
6. 环境：只用标准库、不联网、无构建步骤；`python -m unittest` 能跑通，测试只读 `samples/**`。

## 六、样例说明

| 样例 | 场景 | 容器与门槛 |
| --- | --- | --- |
| `cases/01-scale-contrast.json` | 大小悬殊：最大 120×100、最小 4×4，38 件 | fixed 240×160，门槛 0.960000 |
| `cases/02-many-identical.json` | 大量同尺寸：1400 件 5×5 | fixed 200×200，门槛 0.875000 |
| `cases/03-exact-fill.json` | 恰好铺满：15 件拼满 | fixed 100×100，门槛 1.000000 |
| `cases/04-sparse-big-empty.json` | 留大片空档：22 件只占一角 | fixed 320×200，门槛 0.184375 |
| `cases/05-rotation-required.json` | 强制旋转才放得下 | fixed 30×25，门槛 1.000000 |
| `cases/06-auto-adaptive.json` | 自适应容器：36 件，面积 3600 | auto，max_area 4000，门槛 0.900000 |

`expected/*.txt` 与 `cases/*.json` 同名成套，逐行给出门槛、上限与核对项；`samples/notes.md` 是现场记录。

## 七、待补的文档

- 2 万件规模的公开基准还没有固化，先按第五节的预算验收。
- 真实纸型清单（从几种标准纸里挑一种）与批量任务的成本核算本期不做，`auto` 只按面积上限验收。
- 无解或非法输入的报错文案与退出码细节留待实现时补；样例均有可行布局。
- 页面配色、缩放与悬停高亮自定；4.4 的 id、`data-metric` 与「只渲染」口径固定。
