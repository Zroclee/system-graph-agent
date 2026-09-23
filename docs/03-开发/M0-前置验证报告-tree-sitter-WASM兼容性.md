# M0 前置验证报告 —— tree-sitter WASM 兼容性与运行时定标

| 项目 | 内容 |
| --- | --- |
| 版本 | v1.0 |
| 状态 | ✅ 已验证（GO） |
| 验证日期 | 2026-09-23 |
| 验证人 | 小喵（AI 助手） |
| 判定依据 | `docs/02-设计/设计-架构方案.md` §12.2、§13 M0 |
| 关联决策 | ADR D-11（运行时 = Bun）、D-16（语法包来源定标） |
| 结论 | **GO —— 全押 Bun，不再编写运行时兼容层** |

---

## 1. 结论摘要

| 问题 | 结论 |
| --- | --- |
| `web-tree-sitter`（WASM）能否在 Bun 上加载并解析？ | ✅ **能。7/7 项验证通过** |
| Bun 与 Node 的行为是否一致？ | ✅ **完全一致**（节点数、错误数、提取结果逐字相同） |
| 是否触发 §12.2 的降级路径（退回 Node 22 LTS）？ | ❌ **不触发** |
| 是否需为 Bun 编写兼容层？ | ❌ **不需要** |

**核心结论**：运行时正式定为 **Bun 1.3.14**。§12.2 中标注的「唯一 go/no-go 级前置风险」已排除。

**但验证过程中发现了一个必须记录的真实陷阱**，与 Bun 无关，见 §4 —— 它会让后来者以完全相同的方式踩坑，且**换到 Node 也一样会失败**。

---

## 2. 验证环境

| 项 | 值 |
| --- | --- |
| 平台 | Darwin arm64（macOS） |
| 运行时 A | Bun 1.3.14 |
| 运行时 B（对照组） | Node.js v22.22.2 |

> 对照组不是"备用方案"，而是**归因工具**：只有两端跑同一份代码，才能区分「Bun 不兼容」与「依赖本身有问题」。本次验证正是靠它避免了一次误判（详见 §4）。

---

## 3. 主验证：tree-sitter WASM on Bun

同一份 `.mjs` 探针在 Bun 与 Node 上分别执行，结果**逐项一致**：

| # | 验证项 | Bun 1.3.14 | Node 22 | 说明 |
| --- | --- | --- | --- | --- |
| T1 | `Parser.init()` 运行时初始化 | ✅ 8ms | ✅ 5ms | 零配置，无需 `locateFile` |
| T2 | `Language.load` 三语言语法包 | ✅ 10ms | ✅ 7ms | java / python / typescript |
| T3 | 解析 + 语法树遍历 | ✅ 6ms | ✅ 8ms | java:189 / python:178 / ts:231 节点，**0 个 ERROR/MISSING** |
| T4 | tree-sitter Query 提取 | ✅ 15ms | ✅ 13ms | Java 注解 9→路由 3；Python decorator 2→路由 2；TS 字面量 6→/api 2 |
| T5 | `Language.load(Uint8Array)` | ✅ 2ms | ✅ 17ms | 支持字节流 → 内联 / 打包场景可用 |
| T6 | 重复解析稳定性 + 吞吐 | ✅ 5684 次/秒 | ✅ 5543 次/秒 | 300 次结果完全一致（189 节点），无漂移 |
| T7 | 并发加载语法包 | ✅ 9ms | ✅ 2ms | 3 个语法包 `Promise.all` 并发加载 |
| | **合计** | **7/7** | **7/7** | **冷启动 → 全部探针完成：103ms / 107ms** |

**T3 的"0 个 ERROR/MISSING"是比"能加载"更强的证据**：它同时证明了语法包与运行时的 ABI 是真正对齐的，而不只是 WASM 实例化成功。

**T4 是提取层的真实性验证**：它模拟的正是 S3 阶段的实际动作 —— 从 Java 注解抽 REST 路由、从 Python decorator 抽 FastAPI 路由、从前端字符串字面量抽 HTTP 调用点。三种语言的提取结果都与源码事实相符。

**T6 的稳定性是关键**：节点数在 300 次重复中完全一致，说明解析结果是确定性的 —— 这是不变量 I1（同一输入必须产出同一张图）能够成立的前提。

---

## 4. ⚠️ 关键发现：一个会让后来者踩坑的依赖陷阱

### 现象

首次验证时，`Parser.init()` 通过，但 **`Language.load()` 全线失败**，且抛出的是一个**空消息异常**：

```
❌ FAIL  T2  Language.load 三语言    Error: 
❌ FAIL  T5  Language.load(Uint8Array)    Error: 
```

异常 `message` 为空字符串，只有 `stack` 可读，指向 `getDylinkMetadata()`。

### 归因过程

用 Node 22 跑**同一份代码**，以完全相同的方式失败 —— 这直接排除了 Bun 的嫌疑。

继续拆到字节层，dump 出 grammar wasm 的段结构：

```
#1 custom  name="dylink"      ← 实际
#2 type
...
```

而 `web-tree-sitter@0.27` 的 loader 源码中：

```js
// web-tree-sitter.js:2001
failIf(name2 !== "dylink.0");   // ← 注意：此 failIf 未传 message，故异常消息为空
```

**根因**：`tree-sitter-wasms@0.1.13`（最后更新 2025-10-07，已停更）提供的语法包 wasm 携带的是**旧的 `dylink` 段名**，而 `web-tree-sitter@0.27` 要求 **`dylink.0`**。这是 Emscripten 侧模块（side module）格式的版本演进，与运行时无关。

### 正确做法（必须写进依赖清单）

**不要使用 `tree-sitter-wasms` 这个聚合包。** 改用**各语言官方维护的语法包**，它们随 `web-tree-sitter` 同步更新，wasm 段格式正确。

实测确认以下来源的 wasm 全部为 `dylink.0`：

| 来源 | 覆盖语言 | 结论 |
| --- | --- | --- |
| ✅ `tree-sitter-<lang>`（官方单语言包） | java / python / typescript / tsx / javascript / go / c-sharp / kotlin / php / ruby / cpp / rust | **推荐，随上游同步更新** |
| ✅ `@vscode/tree-sitter-wasm` | 17 种（含 bash / css / ini / powershell / regex） | 备选，VS Code 在维护 |
| ❌ `tree-sitter-wasms` | 36 种，看似最全 | **停更，段格式过旧，勿用** |

> 注意 `tree-sitter-vue`（2022 年停更）同样陈旧，但**不影响我们** —— Vue SFC 按 §12.1 本来就由 `@vue/compiler-sfc` 解析，不走 tree-sitter。

### 复现要点（约 12 行）

```js
import { Parser, Language } from 'web-tree-sitter';
await Parser.init();                                    // ✅ 零配置即可
const lang = await Language.load(
  'node_modules/tree-sitter-java/tree-sitter-java.wasm' // ⚠️ 必须来自官方单语言包
);
const parser = new Parser();
parser.setLanguage(lang);
const tree = parser.parse(source);
console.log(tree.rootNode.type, tree.rootNode.hasError); // program, false
```

**判定 healthy 的最短标准**：`rootNode.type` 正确 **且** `hasError === false`。只看"没抛异常"是不够的 —— 段格式不匹配时异常消息可能为空，容易被误读为运行时故障。

---

## 5. 附带验证：技术栈可用性（§12.1 选型）

主验证通过后，顺带把 M0–M3 会实际用到的选型在 Bun 上跑了一遍（用 TS 编写，同时验证「Bun 直跑 TypeScript」不是空话）：**9/9 全部通过**。

| # | 选型 | 结果 | 关键信息 |
| --- | --- | --- | --- |
| S0 | TypeScript 直跑 | ✅ | interface + 泛型 + enum 正常求值，**无 tsx/ts-node 中间层** |
| S1 | `bun:sqlite` | ✅ | 内置零安装；500 行事务写入；**`journal_mode=wal` 生效** |
| S2 | zod 4.6.5 | ✅ | 校验通过/拒绝均正确；**`z.toJSONSchema()` 原生导出**（无需 zod-to-json-schema，D-05 路径更短了） |
| S3 | `@babel/parser` + `@vue/compiler-sfc` | ✅ | SFC 正确取出 `script setup` 块；babel 解析 TS 正常 |
| S4 | `node-sql-parser` | ✅ | 从 JOIN 语句正确抽出表名 `orders,users`（裸 SQL 抽取可用） |
| S5 | `graphology` | ✅ | 节点/边/度数计算正常 |
| S6 | `hono` | ✅ | 路由注册 + JSON 响应正常（本地 API 服务可行） |
| S7 | `clipanion` | ✅ | `Command` + `Option` 定义与实例化正常 |
| S8 | `elkjs` | ✅ | layered 布局产出坐标（见下文注意事项） |

### 两个次要发现

**（1）`node-sql-parser` v5 起 `ast()` 已更名为 `astify()`。**
沿用旧方法名会得到 `TypeError: ... .ast is not a function`。这不是运行时问题，是 API 变更。**实现时须用 `astify()`。**

**（2）`elkjs` 必须显式提供 `workerFactory`，且两个运行时的可用方式恰好相反。**

`elkjs` 的 `elk.bundled.js` 会自动探测全局 `self` / `document` 来挑选 worker 机制 —— 而 Bun 与 Node 对这些 Web 全局的暴露不同，导致自动探测在两端走向不同分支：

| elkjs 用法 | Bun 1.3.14 | Node 22 |
| --- | --- | --- |
| 全局 `Worker` → `elk-worker.min.js` | ✅ **成功** | ❌ `Worker is not a constructor` |
| `node:worker_threads` → `elk-worker.min.js` | ❌ 超时 | ❌ 超时 |
| 进程内 fake worker | ❌ | ✅ **成功** |

**两个运行时都能跑，但必须显式指定工厂，不能依赖自动探测。** 另外，布局本身发生在 M5 的浏览器画布侧（那里 `self`/`document` 存在，默认路径正常），此项**不构成风险，仅为 M5 的实现注意点**。

> 附带说明：Bun 默认**拦截依赖的 postinstall 脚本**（本次安装时提示 `Blocked 3 postinstalls`）。对本项目是**利好** —— 我们只需要语法包 tarball 里的 `.wasm`，不需要 `tree-sitter-*` 的原生编译产物，因此拦截不影响功能，反而避免了原生模块构建。

---

## 6. 版本定标（建议写入工程依赖清单）

| 类别 | 依赖 | 锁定版本 |
| --- | --- | --- |
| 运行时 | Bun | 1.3.14 |
| 多语言 AST | `web-tree-sitter` | 0.27.0 |
| 语法包 | `tree-sitter-java` | 0.23.5 |
| 语法包 | `tree-sitter-python` | 0.23.6 |
| 语法包 | `tree-sitter-typescript` | 0.23.2（含 tsx） |
| Schema | `zod` | 4.6.5 |
| Vue 解析 | `@vue/compiler-sfc` | 3.5.43 |
| JS/TS 解析 | `@babel/parser` | 7.29.9 |
| SQL 解析 | `node-sql-parser` | 5.4.0 |
| 图数据 | `graphology` | 0.26.0 |
| 布局 | `elkjs` | 0.12.0 |
| CLI | `clipanion` | 4.0.0-rc.4 |
| Web 服务 | `hono` | 4.13.8 |

> `clipanion` 目前最新为 `4.0.0-rc.4`（预发布）。功能可用，但**建议在 M0 决定：锁 rc 版，或降到稳定的 3.x**。这是一处需要显式拍板的小决策。

---

## 7. 对设计文档的影响

| 章节 | 需要的变更 |
| --- | --- |
| §12.2 | 风险已排除 → 改写为「已验证通过」；并入 §4 的依赖陷阱作为**强制约束** |
| §12.1 | 「多语言 AST」行补注语法包来源（官方单语言包，**禁用 `tree-sitter-wasms`**） |
| §13 M0 | ① 前置验证标记为已完成 |
| §14.1 R9 | 关闭，记录结论 |
| §14.2 ADR | 新增 D-16：语法包来源定标 |

---

## 8. 验证产物处置

按约定，本次验证为**一次性技术验证（spike）**，验证代码不进入产品仓库，已删除。

本报告保留了**结论、判定标准、复现要点与版本定标**四类信息 —— 它们是长期资产；一次性探针脚本不是。
