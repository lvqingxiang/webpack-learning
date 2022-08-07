# webpack4 VS webpack5

学习 webpack5 新框架，比较和 webpack4 的区别和联系

#### 前提条件

- node:`v16.16.0`
- npm:`v8.11.0`
- yarn:`v3.2.2`
- `yarn create react-app [name] --template typescript`获取使用`webpack5`进行打包的项目。
-  `yarn create react-app [name] --template typescript --scripts-version 4.0.3`获取使用`webpack4`进行打包的项目。
- `yarn eject`

保留2个项目中通过`yarn eject`暴露出来的`config`、·`script`文件，为方便比较，分别命名为`config5`、`config4`、`script5`、`script4`。

使用`beyond compare` 软件对文件夹内容进行比较。

------

#### `config`文件

- ##### 新增

`webpack5`新增`webpack/persistentCache/createEnvironmentHash.js`文件

##### webpack4

```js
{
  test: /\.(js|mjs|jsx|ts|tsx)$/,
  include: paths.appSrc,
  loader: require.resolve('babel-loader'),
  options: {
    customize: require.resolve('babel-preset-react-app/webpack-overrides'),
    presets: [
      [
        require.resolve('babel-preset-react-app'),
        {
          runtime: hasJsxRuntime ? 'automatic' : 'classic',
        },
      ],
    ],

    plugins: [
      [
        require.resolve('babel-plugin-named-asset-import'),
        {
          loaderMap: {
            svg: {
              ReactComponent: '@svgr/webpack?-svgo,+titleProp,+ref![path]',
            },
          },
        },
      ],
      isEnvDevelopment &&
        shouldUseReactRefresh &&
        require.resolve('react-refresh/babel'),
    ].filter(Boolean),
    // This is a feature of `babel-loader` for webpack (not Babel itself).
    // It enables caching results in ./node_modules/.cache/babel-loader/
    // directory for faster rebuilds.
    cacheDirectory: true,
    // See #6846 for context on why cacheCompression is disabled
    cacheCompression: false,
    compact: isEnvProduction,
  },
}
```

`hash`

和整个项目构建有关，全部文件共用一个`hash`值。有任何文件修改，都会在打包时生成新的`hash`值，同时缓存将全部失效。

`chunkhash`

`hash`方式即便有文件未修改，仍然会在打包后生成新的`hash`值。`chunkhash`根据不同的入口文件进行文件解析，构建对应的`chunk`，生成对应的`hash`值，因此我们可以选择文件单独打包构建，实现`chunk`内容不变时，将不会生成新的`hash`值。

`contenthash`

`chunkhash`如果`chunk`内包含`js`文件及`css`文件，共用相同的`hash`值，即便只修改`js`文件内容，也会引发`css`文件的`hash`值改变。`contenthash`根据文件内容生成`hash`值。

`babel-loader`：可以配置 `cacheDirectory` 来将 `babel` 编译的结果缓存下来

`webpack4`缓存只存在于内存中，当程序重新构建，内存就会丢失。

##### webpack5

```js
cache: {
  type: 'filesystem',
  version: createEnvironmentHash(env.raw),
  cacheDirectory: paths.appWebpackCache,
  store: 'pack',
  buildDependencies: {
    defaultWebpack: ['webpack/lib/'],
    config: [__filename],
    tsconfig: [paths.appTsConfig, paths.appJsConfig].filter((f) =>
      fs.existsSync(f)
    ),
  },
}
```

`webpack5` 统一了持久化缓存的方案，有效降低了配置的复杂性。

`cache: true` 与 `cache: { type: 'memory' }` 配置作用一致，表示缓存在内存中。`cache.type` 设置为 `'filesystem'` 时，表示缓存在文件系统（本地硬盘）中。

`cache.version`当配置文件和代码都没有发生变化，但是构建的外部依赖（如环境变量）发生变化时，预期的构建产物代码也可能不同。这时就可以使用 `version` 配置来防止在外部依赖不同的情况下混用了相同的缓存。

- ##### 删除

`webpack5`删除`pnpTs.js`文件

##### webpck4

```js
useTypeScript &&
  new ForkTsCheckerWebpackPlugin({
    typescript: resolve.sync('typescript', {
      basedir: paths.appNodeModules,
    }),
    async: isEnvDevelopment,
    checkSyntacticErrors: true,
    resolveModuleNameModule: process.versions.pnp
      ? `${__dirname}/pnpTs.js`
      : undefined,
    resolveTypeReferenceDirectiveModule: process.versions.pnp
      ? `${__dirname}/pnpTs.js`
      : undefined,
    tsconfig: paths.appTsConfig,
    reportFiles: [
      '../**/src/**/*.{ts,tsx}',
      '**/src/**/*.{ts,tsx}',
      '!**/src/**/__tests__/**',
      '!**/src/**/?(*.)(spec|test).*',
      '!**/src/setupProxy.*',
      '!**/src/setupTests.*',
    ],
    silent: true,
    formatter: isEnvProduction ? typescriptFormatter : undefined,
  })
```

##### webpack5

```js
useTypeScript &&
  new ForkTsCheckerWebpackPlugin({
    async: isEnvDevelopment,
    typescript: {
      typescriptPath: resolve.sync('typescript', {
        basedir: paths.appNodeModules,
      }),
      configOverwrite: {
        compilerOptions: {
          sourceMap: isEnvProduction ? shouldUseSourceMap : isEnvDevelopment,
          skipLibCheck: true,
          inlineSourceMap: false,
          declarationMap: false,
          noEmit: true,
          incremental: true,
          tsBuildInfoFile: paths.appTsBuildInfoFile,
        },
      },
      context: paths.appPath,
      diagnosticOptions: {
        syntactic: true,
      },
      mode: 'write-references',
    },
    issue: {
      include: [
        { file: '../**/src/**/*.{ts,tsx}' },
        { file: '**/src/**/*.{ts,tsx}' },
      ],
      exclude: [
        { file: '**/src/**/__tests__/**' },
        { file: '**/src/**/?(*.){spec|test}.*' },
        { file: '**/src/setupProxy.*' },
        { file: '**/src/setupTests.*' },
      ],
    },
    logger: {
      infrastructure: 'silent',
    },
  })
```

`TypeScript`中的类型检测可以规范我们变量的定义，但同时会降低构建速度。`babel`+`ForkTsCheckerWebpackPlugin`，`babel`只负责`ts`文件的解析及转换为`js`, 并不会做对应的类型检查。使用 `ForkTsCheckerWebpackPlugin`会将类型检查过程移至单独的进程，可以加快 `TypeScript` 的类型检查和 ESLint 插入的速度。

`pnpTs.js` 为了解决`require()`时过多的`I/O`操作带来的性能消耗，`pnp`思想来自`Yarn`团队，目的是为了解决安装和引用依赖效率过低问题。其建立了一张映射表，这张表记录了依赖版本关联和依赖与依赖之间的关联以及依赖存放的位置。有了这张表，就可以跳过繁琐的查找过程直接确定依赖在文件中的位置，从而提高性能。

`webpack5`默认支持。

##### 修改

`webpack4`->`webpack5`具体改动有：

- 移除`PnpWebpackPlugin/'pnp-webpack-plugin'`，webpack5默认支持。

- 更换`OptimizeCSSAssetsPlugin/'optimize-css-assets-webpack-plugin'`为`CssMinimizerPlugin/'css-minimizer-webpack-plugin'`。`css-minimizer-webpack-plugin`插件使用 [cssnano](https://www.cssnano.cn/) 优化和压缩 `CSS`。在`source maps`和`assets`中使用查询字符串比`optimize-css-assets-webpack-plugin`更加准确，支持缓存和并发模式下运行。

- 移除`safePostCssParser/'postcss-safe-parser'`，原先`optimize-css-assets-webpack-plugin`依赖`postcss-safe-parser`，移除`optimize-css-assets-webpack-plugin`则`postcss-safe-parser`也不再需要。

- 移除`WatchMissingNodeModulesPlugin/'react-dev-utils/WatchMissingNodeModulesPlugin'`。当你在项目里引用了没有安装的`node_modules`,`webpack`会报错`module not found`，当你安装完成后需要手动重启服务。`WatchMissingNodeModulesPlugin`插件省去手动步骤，依赖安装完成之后`webpack`会自动重启。

- 移除`typescriptFormatter/'react-dev-utils/typescriptFormatter'`

- 移除`webpackDevClientEntry/'react-dev-utils/webpackHotDevClient'`

- 增加`reactRefreshRuntimeEntry/'react-refresh/runtime'`，`React Fast Refresh`是 `React` 官方为`React Native` 开发的**模块热替换（HMR）方案**，兼具稳定性和可维护性，核心实现与平台无关，适用于 `Web`。

  **`reload`策略** **完全支持函数式组件和`Hooks`**

  `Live Loading`：模块文件发生变化时，重新加载整个应用程序。

  `Hot Loading`：保留应用程序的运行状态，只对有变化的部分进行局部刷新。

  - 如果所编辑的模块仅导出`react`组件，则只更新该模块代码并重新渲染对应代码。
  - 如果所编辑的模块不仅是`react`组件，将更新该模块代码及所有依赖它的模块。
  - 如果所编辑的文件被`react`组件树之外的模块引用，则会降级为`Live Loading`

  **容错处理**

  语法错误：`Fast Refresh`期间的语法错误会被捕捉，修复并保存文件无需重新刷新即可恢复运行。

  运行时错误：模块初始化时的运行时错误同样会被捕捉，对于组件的运行时错误，`fast Refresh` 会在没有`Error Boundary`的情况下重新刷新整个应用。

  对于语法错误和部分拼写错误，修复后`Fast Refresh`即可恢复正常，而对于组件运行时错误，在没有`Error Boundary`情况下，会降级到`Live Loading`，否则局部刷新`Error Boundary`。

  **限制**

  在有些情况下，维持状态并不十分安全，为可靠起见，`Fast Refresh`遇到以下情况一概不保留状态：

  - `Class`组件及高阶组件返回的Class组件一律`remount`，`state`会被重置。
  - 不纯组件模块，所编辑的模块除了导出`React`组件，还导出了其他内容。

- 移除`reactRefreshOverlayEntry/'react-dev-utils/refreshOverlayInterop'`

- 增加`babelRuntimeEntryHelpers/'@babel/runtime/helpers/esm/assertThisInitialized'`

  `babel` 默认会在每个文件顶部放置所需要的辅助函数，当文件数量较多，会造成重复不利于体积优化。通过统一从` babel-runtime/helpers `中引入函数，减少代码体积。

- 增加`babelRuntimeRegenerator/'@babel/runtime/regenerator'`，当使用`generator`或 `async`函数时，用`babel-runtime/regenerator`导出的函数取代。

- 兼容`Tailwind CSS`，`Tailwind CSS` 是一个功能类优先的`CSS`框架，它集成了诸如 `flex`, `pt-4`, `text-center`和`rotate-90`这样的的类，它们能直接在脚本标记语言中组合起来，构建出任何设计。

- 新增配置`assetModuleFilename: 'static/media/[name].[hash][ext]'`，默认情况下，`asset/resource`模块以`[hash][ext][query]`文件名发送到输出目录，可以通过在`webpack`配置中设置`output.assetModuleFilename`来修改此模板字符串。

  ##### 资源模块

  资源模块`(asset module)`是一种模块类型，它允许使用资源文件（字体，图标等）而无需配置额外`loader`。

  在`webpack5`之前，通常使用`raw-loader`将文件导入为字符串，`url-loader`将文件作为`data URI`内联到`bundle`中，`file-loader`将文件发送到输出目录。

  资源模块类型`(asset module type)`，通过添加 4 种新的模块类型，来替换所有这些`loader`：

  - `asset/resource` 发送一个单独的文件并导出 `URL`。之前通过使用`file-loader`实现。
  - `asset/inline` 导出一个资源的 `data URI`。之前通过使用`url-loader`实现。
  - `asset/source` 导出资源的源代码。之前通过使用`raw-loader`实现。
  - `asset` 在导出一个 `data URI` 和发送一个单独的文件之间自动选择。之前通过使用`url-loader`，并且配置资源体积限制实现。

- 新增配置`target: ['browserslist']`。

  **构建目标(Targets)**

  由于`javascript`既可以编写服务端代码也可以编写浏览器代码，所以`webpack`提供了多种部署`target`，可以在`webpack`的配置选项中进行配置。

  `webpack`能够为多种环境或`target`构建编译，默认值为`browserslist`，如果没有找到`browserslist`的配置则默认为`web`。如果一个项目`browserslist`有配置，将会使用它确定可用于运行时代码的`ES`特性及推断环境。

  当传递多个目标时，将使用共同的特性子集。当没有提供 `target`或`environment`特性的信息时，将默认使用`ES2015`。

  支持的`browserslist`值：

  - `browserslist` - 使用自动解析的`browserslist`配置和环境（从最近的 `package.json` 或 `BROWSERSLIST` 环境变量中获取）
  - `browserslist:modern` - 使用自动解析的`browserslist`配置中的`modern`环境
  - `browserslist:last 2 versions` - 使用显式`browserslist`查询，此时配置将被忽略
  - `browserslist:/path/to/config` - 明确指定`browserslist`配置路径
  - `browserslist:/path/to/config:modern` - 明确指定`browserslist`的配置路径和环境

- 移除`babel-plugin-named-asset-import`，支持以`React Component`的方式引入`svg`文件（[参考](https://stackoverflow.com/questions/63531768/what-does-babel-plugin-named-asset-import-do)）

- 调用`getStyleLoaders`函数参数对象新增`mode`

  `mode`类型：`String|Function` 默认：`'local'`，需要 `local` 模式时可以忽略该值。控制应用于输入样式的编译级别。

  `local`、`global` 和 `pure` 处理 `class` 和 `id` 域以及 `@value` 值。 `icss` 只会编译底层的`Interoperable CSS`格式，用于声明`CSS`和其他语言之间的`:import`和`:export`依赖关系。

  `ICSS`提供`CSS Module`支持，并且为其他工具提供了一个底层语法，以实现它们自己的`css-module`变体。

  可选值 - `local`、`global`、`pure` 和 `icss`。

##### 补充知识

- `postcssNormalize/'postcss-normalize'`

`normalize`是一个很小的`css`文件，但它在默认的`HTML`元素样式上提供了跨浏览器的高度一致性，是一种现代的、为`HTML5`准备的优质方案。

`normalize.css`有以下几个目的：保护有用的浏览器默认样式而不是完全去掉它们；为大部分`HTML`元素提供一般化的样式；修复浏览器自身的`bug`并保证各浏览器的一致性；优化`CSS`可用性；用注释和详细的文档解释代码。

`postcss-normalize`支持使用配置的`browserslist`中所需的`normalize.css`。

- `HtmlWebpackPlugin/'html-webpack-plugin'`

[`HtmlWebpackPlugin`](https://github.com/jantimon/html-webpack-plugin) 简化了`HTML`文件的创建，以便为你的`webpack`包提供服务。这对于那些文件名中包含哈希值，并且哈希值会随着每次编译而改变的`webpack`包特别有用。你可以让该插件为你生成一个`HTML`文件，使用 [`lodash` 模板](https://lodash.com/docs#template)提供模板，或者使用你自己的 [`loader`](https://webpack.docschina.org/loaders)。

该插件将为你生成一个`HTML5`文件， 在`body`中使用 `script` 标签引入所有`webpack`生成的`bundle`。 

------

##### 概念摘录

`webpack` 只能理解`JavaScript`和`JSON`文件，这是` webpack`开箱可用的自带能力。**loader** 让 webpack`能够去处理其他类型的文件，并将它们转换为有效模块，以供应用程序使用，以及被添加到依赖图中。**loader** 有两个属性：

1. `test` 属性，识别出哪些文件会被转换。
2. `use` 属性，定义出在进行转换时，应该使用哪个`loader`。

`loader` 用于转换某些类型的模块，而插件则可以用于执行范围更广的任务。包括：打包优化，资源管理，注入环境变量。

想要使用一个插件，你只需要 `require()` 它，然后把它添加到 `plugins` 数组中。多数插件可以通过选项(option)自定义。你也可以在一个配置文件中因为不同目的而多次使用同一个插件，这时需要通过使用 `new` 操作符来创建一个插件实例。

通过选择 `development`, `production` 或 `none` 之中的一个，来设置 `mode` 参数，你可以启用 webpack 内置在相应环境下的优化。其默认值为 `production`。

在[模块化编程](https://en.wikipedia.org/wiki/Modular_programming)中，开发者将程序分解为功能离散的 chunk，并称之为 **模块**。

每个模块都拥有小于完整程序的体积，使得验证、调试及测试变得轻而易举。 精心编写的 **模块** 提供了可靠的抽象和封装界限，使得应用程序中每个模块都具备了条理清晰的设计和明确的目的。

------

##### 相对区别

- `Tree Shaking`

*tree shaking* 是一个术语，通常用于描述移除`JavaScript`上下文中的未引用代码(dead-code)。它依赖于 ES2015 模块语法的 [静态结构](http://exploringjs.com/es6/ch_modules.html#static-module-structure) 特性，例如 [`import`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/import) 和 [`export`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/export)。`webpack5`的 `mode=“production” `自动开启 `tree-shaking`。`webpack5` 增加了对嵌套模块的导出跟踪功能，能够找到那些嵌套在最内层而未被使用的模块属性，增加了分析模块中导出项与导入项的依赖关系的功能。通过 `optimization.innerGraph`（生产环境下默认开启）选项，`webpack5` 可以分析特定类型导出项中对导入项的依赖关系，从而找到更多未被使用的导入模块并加以移除。 

- `Module Federation`

微前端是一种类似于微服务的架构，是一种由独立交付的多个前端应用组成整体的架构风格，将前端应用分解成一些更小、更简单的能够独立开发、测试、部署的应用，而在用户看来仍然是内聚的单个产品。有一个**基座应用**（主应用），来管理各个**子应用**的加载和卸载。

微前端的核心三大原则就是：**独立运行、独立部署、独立开发**

在微前端中，会存在一个容器应用，它的任务就是加载各个微应用。

微应用需要做一些事：

- 提供两个方法：一个是挂载方法，容器将调用它来渲染微应用。另一个是卸载方法，用于卸载微应用，并且他们都将以接口方法的形式提供给容器调用；
- 提供远程入口文件的地址，容器应用选择合适的时机动态加载该文件，在获得挂载方法后，执行微应用渲染；
- 提供微应用 ID，用于标识自己，对微应用的操作需要该标识；
- 首个路由地址，在挂载微应用后，决定微应用的视图展示。

容器应用要做的事是：

- 加载远程的微应用（下载远程 js 入口文件），并执行渲染；
- 在合理的契机，卸载微应用。

模块联邦是 webpack5 引入的特性，能**轻易实现**在两个使用`webpack`构建的项目之间**共享代码**，甚至**组合不同的应用为一个应用**。

通过webpack的动态加载，就已经实现了容器应用该做的事情。所以我们完全可以认为微应用本身可以具备容器应用的功能。当微应用成为容器应用后，每个应用在架构里都平等得存在，容器应用之间可以相互的依赖和相互的加载及使用。

`Host`：消费方，它动态加载并运行其他 Container 的代码。

`Remote`：提供方，它**暴露属性**（如组件、方法等）供 **Host** 使用

`expose`：导出应用，被其他应用导入

`remote`：引入其他应用

在微前端中：

- 加载微应用必须预定义接口方法（`mounted`、`unmount` 等）来实现微应用的动态挂载和卸载等功能，这意味着每个微应用必须手动实现这些接口方法；
- 微应用的切换通常由路由状态改变来触发的。
- 如果我们想要共享某个微应用的模块给其它微应用使用，这并不是轻松地事。这意味着你需要把该模块独立出去，并以合理调用方式被其它微应用远程加载。

在模块联合中：

- 模块联合每个微应用也可以是一个容器应用，所以他们之间可以相互依赖及加载；

- 每个应用允许暴露（`exposes`）多个接口，其它应用可以在动态远程加载该应用后，直接使用其接口。这解决了上面微前端提到的的模块共享问题；
- 在模块使用上非常灵活，当你引用一个远程模块时，可以像使用普通的 `npm` 包一样使用它，当然也允许懒加载模块；
- 远程模块和路由没有任何关联，加载的契机完全由 `host` 应用自己灵活决定。

联合模块作为微前端的技术延展，其依然具备着微前端的特性，即每个容器应用应该独立开发和独立部署，并团队自治。
