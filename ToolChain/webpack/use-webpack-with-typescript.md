# 使用 Webpack 编译并且打包你的 TypeScript 库

[查看原文](https://marcobotto.com/blog/compiling-and-bundling-typescript-libraries-with-webpack/)

## 在 JavaScript 中使用 TypeScript 库

我们会发现编写库和应用程序代码是两种截然不同的体验。它们服务于不同的目的和目标：应用程序代码以应用程序的用户为目标，通常不与底层代码交互，而**库以其他开发人员为目标**，因此，它需要有**更好的可重用性**和开发人员体验。

除了影响我们的代码风格和架构（将来应该成为一个单独的主题）之外，这种差异会对我们的工具产生直接影响。在使用 `TypeScript` 进行库的开发时要记住一些实践方法，以便为其他开发人员提供最佳的支持并且避免他们碰到一些不必要的困难，这一点很重要。

如今，为前端（浏览器）编写的任何代码必须与当前的情况做兼容，即（即使在 2017 年）老的 ES5 规范。

`TypeScript` 附带了 `tsc`（TS 编译器）来处理：它将代码编译为 ES5 并创建包含类型声明的 `*.d.ts` 文件，以保留 `TypeScript` 支持。

> 那些不依赖打包工具的用户（例如 Webpack 和 Rollup）呢？

> 当使用打包工具是，`tree shaking` 又是如何实现的呢？

我们可以通过一个小而有效的 pipeline 实现所有这些不同设置的兼容性，这就是我们在 `UI-Router` 中使用的。

## Webpack + tsc + npm script = ♥️

我们的目标是创建三种不同的输出：

1. `tsc` 编译完的代码 + `*.d.ts` 类型声明 + `source map`（非常方便调试 TS 源代码）。模块语法将是 CommonJS，用于支持大多数打包工具。

2. 与上面相同，使用 ES6 模块语法，以便使最新工具（Webpack 2 / Rollup）能够执行 `tree shaking` 以减少打包文件的大小。

3. 编译为标准 ES5 的 `UMD` 包，可在浏览器中工作并公开全局变量。

通常我们不需要在 `UMD` 的编译代码中提供 TS 相关的类型声明文件，因为 `UMD` 被定义为应该是一个通用的库。类型声明文件不是有效的 JavaScript 文件（因为是 `.ts` 扩展名），并且在浏览器和非 TS 环境中使用时会导致包中出现问题。

通过这种方式，我们可以保留对 TS 应用程序以及通用模块打包工具的完全支持，同时在需要时回退到 UMD。

我们通过使用 `tsc` 来解决前两个方案，然后使用 `Webpack` 来解决第三种方案。我们可以在 shx 的帮助下将其包装成 npm 脚本。

## 目录结构

想象一下一个库的目录结构是这样的：

```
node_modules/
src/
package.json
README.md
tsconfig.json
yarn.lock
```

我们想要在 npm 上发布的代码是我们刚刚讨论的三份编译后的代码，以及其他几个文件：

```
_bundles/		// UMD bundles
lib/			// ES5(commonjs) + source + .d.ts
lib-esm/		// ES5(esmodule) + source + .d.ts
package.json
README.md
```

为了排除其余的文件，`npm` 接受一个与 `.gitignore` 相同的 `.npmignore`。如果我们不提供这个，`npm` 会忽略 `.gitignore` 里面的东西，当然这不是我们想要的东西。

> 译者注：建议使用 `package.json` 中的 `files` 字段来指定你想要发布到 `npm` 的文件。

您可以在此仓库中找到代码，因为它可能有助于帮你作为文章其余部分的参照。

## TypeScript 配置

TS 只需要一个 `tsconfig.json` 文件来配置编译器选项并指定该文件夹是一个 TS 项目。

根据库的需要，有许多配置选项可能会有所不同。我们需要关注的选项包括：

```json
{
  "module": "commonjs",
  "target": "es5",
  "lib": ["es2015", "dom"],
  "outDir": "lib",
  "sourceMap": true,
  "declaration": true
}
```

#### "module": "commonjs"

我们告诉编译器默认情况下我们希望我们的模块声明采用 CommonJS 语法。这将编译每个导入和导出到 `require()` 和 `module.exports` 声明，这是 `NodeJS` 环境中的默认语法。

#### "target": "es5"

目标代码会是 ES5 的语法，因为正如我们之前解释的那样，我们需要提供可以在没有进一步编译/转换的情况下运行的代码。

#### "lib": [ "es2015", "dom" ]

`lib` 是 TS 包含的特殊声明文件。它包含 JS 在运行时或者 DOM 环境等。

根据 `target` 不同，TS 会自动包含 dom 和 ES5 语法。这就是为什么我们需要指定我们需要的类型为 es2015 和 dom。

这样我们就可以在输出 ES5 的代码同时使用所有的 ES6 类型。

#### "outDir": "lib"

如前所述，编译好的代码将保存到 lib 文件夹中。

#### "sourceMap": true & "declaration": true

我们同时需要来自我们源代码的 source maps 以及 声明文件。

---

使用如上配置，当我们运行 tsc 命令时，编译器将会为我们创建上述三个输出中的第一种。

我们可以为第二个项目创建第二个 tsconfig 文件，但由于配置几乎相同，我们将使用命令行的 `--flag` 来覆盖这些选项：

```
tsc -m es6 --outDir lib-esm
```

编译器将使用 ES6 模块而不是 CommonJS（-m es6）并将其放在指定文件夹（--outDir lib-esm）中。

我们可以通过执行 `tsc && tsc -m es6 --outDir lib-esm` 来组合这两个命令。

## Webpack 配置

最后一块拼图是 Webpack，它将为我们进行打包。由于 webpack 不理解 TypeScript，我们需要使用 `loader`，就像我们使用 `babel-loader` 来指定 Webpack 通过 Babel 编译源代码一样。

有几个 TS loader，我们最终决定使用 `awesome-typescript-loader`， 有几个原因：它更快，通过将类型检查和触发器分配到单独的进程中，并且与 Babel 有很好的集成，以防万一需要使用一些方便的 Babel 插件进一步编译你的代码（我们不会在这篇文章中讨论）。

Webpack 配置文件因项目而异，因为它是一个非常强大的工具，开发人员使用它来做各种事情。但对于（相对简单）库，我们需要做的就是打包 ES5 语法的代码（使用上述 tsconfig 中的配置输出的编译后代码）成为两个单独的文件（一个压缩版本）。

Webpack 和 loader 还将创建包含原始 TS 代码的文件的 source map，这对于调试非常方便。

我假设您已经熟悉 Webpack，或者至少您之前已经尝试过。如果您有任何疑问，可以查看版本 2 的新网站，深入了解该工具背后的概念以及示例和文档。

基本配置是这样的：

```js
{
  "entry": {
    "my-lib": "./src/index.ts",
    "my-lib.min": "./src/index.ts"
  },
  "output": {
    "path": path.resolve(__dirname, "_bundles"),
    "filename": "[name].js",
    "libraryTarget": "umd",
    "library": "MyLib",
    "umdNamedDefine": true
  },
  "resolve": {
    "extensions": [".ts", ".tsx", ".js"]
  },
  "devtool": "source-map",
  "plugins": [
    new webpack.optimize.UglifyJsPlugin({
      "minimize": true,
      "sourceMap": true,
      "include": /\.min\.js$/
    })
  ],
  "module": {
    "loaders": [
      {
        "test": /\.tsx?$/,
        "loader": "awesome-typescript-loader",
        "exclude": /node_modules/,
        "query": {
          "declaration": false
        }
      }
    ]
  }
}
```

这是一个标准的 Webpack 配置，但要记住以下几点：

#### output

我们通过设置一些属性来告诉 Webpack 我们正在打包一个库。该值是库的名称。`libraryTarget` 和 `umdNamedDefine` 告诉 Webpack 创建一个 UMD 模块，并用我们刚刚设置的 lib 的名称命名它。

#### extensions

Webpack 通常将 `.js` 视为模块，因此我们需要告诉它还要查找 `.ts` 和 `.tsx` 文件（如果您使用的是 React + JSX）。

#### module

最后我们使用 `awesome-typescript-loader` 来解析源代码。这里使用查询参数来定制 atl 并关闭类型声明输出非常重要。加载器将使用 `tsconfig.json` 文件来指示编译器，但我们在此定义的所有内容都将覆盖配置文件。

> atl 就是 awesome-typescript-loader 的缩写，坑爹，找了半天啥东西……

> Tip: 如果我们决定使用 React 和 JSX 语法，只要添加正则表达式 `/\.tsx?$/` 匹配`.ts`和`.tsx`文件即可。

#### devtool & plugins

`source-map` 生成一个生成环境所使用的代码映射，拥有源代码的位置信息等内容。压缩版本将由 `UglifyJSPlugin` 创建，因此我们必须指定我们想要的 sourcemap，因为插件默认值为 false。

---

一旦设置了配置文件，如果我们在终端中运行 webpack，它应该创建 `\_bundles` 文件夹，其中包含我们的 4 个文件：

```
my-lib.js
my-lib.js.map
my-lib.min.js
my-lib.min.js.map
```

#### 包含一切

现在我们已经完成了所有设置，我们可以创建几个 npm 脚本来自动化该过程并在每次编译时清理文件夹。

我喜欢使用 `shx`，因为它是跨平台的，所以我不必担心我正在使用哪种操作系统。

`clean` 脚本可以负责删除新版本的文件夹（如果存在）：

```
shx rm -rf _bundles lib lib-esm
```

`build` 脚本关心如何将内容放在一起：

```
npm run clean && tsc && tsc -m es6 --outDir lib-esm && webpack
```

按顺序：它将删除三个文件夹中的任何一个（如果存在），而不是构建三个版本。

#### .gitignore & .npmignore

为了处理代码仓库并发布到 npm 注册表，我们必须记住在 ignore 文件中添加一些内容：

.gitignore:

```
node_modules
lib
lib-esm
_bundles
```

作为模块或构建的所有东西都不应该留在存储库中，因为我们可以在需要时在本地构建它。

.npmignore:

```
.*
**/tsconfig.json
**/webpack.config.js
node_modules
src
```

最终用户将使用已编译的文件（无论哪个版本），因此我们可以删除整个源代码，依赖项和 TS / Webpack 配置。

这些文件不会产生任何问题，但最好尽量减少文件数量，因为每次有人使用它都会被下载，这只会浪费带宽和存储空间。

## 结论

我相信还有许多其他的配置也能正常工作，但是目前的功能已经足够适用于我们，并且可以轻松配置以满足更复杂的需求。

理想情况是只有两个输出：commonjs 和 es 模块。如果当前没有办法在单个打包文件中创建类型定义（镜像打包的 UMD 模块），UMD 文件将满足我们作为 commonjs 的需求。

此外，自 TypeScript 1.8 以来，编译器添加了选项（-outFile）用来遍历导入单个打包文件。我更喜欢使用 Webpack，因为它让我可以更好地控制代码转换 pipeline（而且仅仅因为我习惯了 Webpack）。
