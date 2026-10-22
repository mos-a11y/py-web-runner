# 浏览器里跑 Python（.py）

一个单文件 HTML，打开就能在浏览器里运行 `.py` 文件。不用装 Python、不用开服务器、不联网也能跑（环境缓存后）。

## 后缀是哪个

**`.py`** —— Python 源代码文件的后缀。这门语言全名就叫 **Python**，文件写作 `xxx.py`。

## 怎么用

1. 下载 `py-runner.html`，双击用浏览器打开（手机浏览器也行）
2. 点「示例」看个能跑的例子，直接 `Ctrl+Enter` 运行
3. 或点「📂 打开 .py」选本地文件，也可以把 `.py` 文件**拖进**虚线框
4. 需要第三方包：左下角填包名点「📦 装包」

页面里点「原理」按钮也有完整说明，不用联网看文档。

## 原理

### 1. 浏览器本来不认识 Python

浏览器原生只执行 JavaScript。要跑 Python，只能把 Python 解释器本身搬进浏览器。

### 2. Pyodide：把 CPython 编译成 WebAssembly

[Pyodide](https://pyodide.org) 做的事就是这个：

```
CPython 源码（C 语言）
    ↓  Emscripten 编译
WebAssembly 字节码（.wasm）
    ↓  浏览器加载
一个跑在标签页里的真 Python 解释器
```

WebAssembly 是浏览器能直接执行的低级字节码，性能接近原生。**所以这不是"用 JS 模拟 Python"，是货真价实的 CPython 在跑。**

代价是体积：整个运行环境约 **10MB**，首次要下载。

### 3. 这份 HTML 做的事

```js
// ① 引入 Pyodide
<script src="https://cdn.jsdelivr.net/pyodide/v0.26.2/full/pyodide.js">

// ② 初始化解释器
const py = await loadPyodide();

// ③ 接管 print，把输出写进页面
py.setStdout({ batched: t => log(t) });
py.setStderr({ batched: t => logErr(t) });

// ④ 执行代码（Async 版本支持顶层 await）
const result = await py.runPythonAsync(code);

// ⑤ 装第三方包
await py.loadPackage(['micropip']);
const micropip = py.pyimport('micropip');
await micropip.install(['requests']);
```

文件读取走 `FileReader`，拖拽走 `dataTransfer.files` —— 都是为了拿到 `.py` 的文本内容填进编辑区。

### 4. 为什么安全

**代码只在你自己的浏览器标签页里执行，不上传、不经任何服务器。**

对比在线 Python 运行器（把代码 POST 到别人的服务器执行），这个方案：

- 代码不离开本机
- 断网也能跑（环境已缓存）
- 拿不到你的文件系统、开不了 socket —— 浏览器沙箱本来就限制着

### 5. 多 CDN 备选

那 10MB 的文件托管在公共 CDN 上，某些网络访问不了。所以放了 4 个备选：

```js
jsdelivr  →  jsdelivr(旧版)  →  unpkg  →  Pyodide 官方
```

一个失败自动换下一个，成功的那个记进 `localStorage`，下次直接用，**不用再试一遍**。

## 能做什么 / 不能做什么

**能用：**
- 全部标准库：`math` `json` `re` `datetime` `collections` `itertools` …
- 纯 Python 的第三方包（micropip 装）
- 数据处理：pandas、numpy（Pyodide 有预编译版）
- 画图：matplotlib（输出到 canvas）
- 顶层 `await`（`runPythonAsync` 支持）

**不能用：**
- 读写本地文件系统（浏览器沙箱）
- 开 socket 连任意主机（要服务端配合）
- 装带 C 扩展且没有预编译版的包
- 特别重的计算会把页面卡住（主线程同步执行）

**想跑重型计算**，可以用 Web Worker 版（把 Pyodide 丢进 worker，主线程不卡）。

## 集成到你自己的页面

只要三行核心代码：

```html
<script src="https://cdn.jsdelivr.net/pyodide/v0.26.2/full/pyodide.js"></script>
<script>
  const py = await loadPyodide();
  py.setStdout({ batched: t => console.log(t) });
  await py.runPythonAsync('print("hi")');
</script>
```

要判断文件是不是 Python：

```js
if (/\.py$/i.test(fileName)) { /* 运行它 */ }
```

## 许可

MIT

---

原项目：https://github.com/mos-a11y/py-web-runner
原作者：mos-a11y
