title: RN 打包那些事儿
date: 2017-11-15
category: 技术
tags: [RN, QRN]
photos: [https://ws1.sinaimg.cn/large/c4b5f11bly1flggag4y6fj20du06vt9r.jpg]
---
[React Native](https://github.com/facebook/react-native/tree/v0.48.1) （以下简称 RN）是 Facebook 开源的一款跨平台移动端开发框架，它与 Hybrid 方案相比，性能更好，拥有更流畅的用户体验，而与原生客户端开发相比，开发成本低，效率高，能够快速进行版本迭代和产品上线。我们从16年3月份开始在公司大范围使用 [QRN](https://ued.qunar.com/qrn/)（基于 RN 进行深度定制的移动开发框架），截至目前已有四十多个项目接入，我们在 RN 开发方面有着大量的踩坑经验，本文以 RN 打包作为切入点，分享一下 RN 官方在打包方面做了哪些处理，以及 QRN 在打包上进行了怎样的定制。
<!-- more -->

![rn-metro-bundler](https://ws1.sinaimg.cn/large/c4b5f11bly1flggag4y6fj20du06vt9r.jpg)

## RN 打包
官方 RN 打包，基于一款 node 命令行工具 react-native-cli，通过执行 `bundle` 命令，将项目所有的的 js 文件打包成一个 bundle 文件，例如：

```
react-native bundle --entry-file index.js --platform ios --dev false --bundle-output ./index.ios.bundle --sourcemap-output ./index.ios.map --assets-dest ./asserts
```

iOS 通过 `NSBundle URLForResource` 方法加载，Android 通过 `loadScriptFromString` 加载 bundle 文件。RN 打包简而言之可以理解为对项目的所有模块（RN 认为一切皆模块，包括 js、json 和图片等）进行拆包、去重过滤、代码转化、重组成新的 bundle 的过程，以下笔者将会结合官方的 [metro-bundler](https://github.com/facebook/metro-bundler/tree/2a64bd390e1d546bf62e1437b8a8aa8a839990a2) 工具来一探打包的奥妙。

![总体流程](https://ws1.sinaimg.cn/large/c4b5f11bly1flhfzogn2mj20ac0bhjrp.jpg)

### 命令
在分析官方打包的具体实现前，可以先从命令行入手，来提前了解下官方打包，通过指定参数来进行打包，以下主要介绍 `entry-file、platform、dev、bundle-output、sourcemap-output` 等几个重要的参数：

+ **entry-file**：即入口文件，打包时以该文件作为入口，一步步进行模块分析处理。
+ **platform**：用于区分打包什么平台的 bundle
+ **dev**：用于区分 bundle 使用环境，非 dev 时，会对代码进行 minified
+ **bundle-output**：打包产物输出地址，即打包好的 bundle 存放地址
+ **sourcemap-output**：打包时生成对应的 [sourcemap](http://www.ruanyifeng.com/blog/2013/01/javascript_source_map.html) 文件存放地址，在跟踪查找错误或崩溃时，能帮助开发快速定位到代码
+ **assets-dest**：bundle 中使用的静态资源文件存放地址

### metro-bundler
RN 官方从 0.46.0 版本开始，出于对打包速度、可靠性和更好的可拓展性等方面的[考虑](https://github.com/facebook/react-native/issues/13976)，将 RN 的打包模块 packager 从 RN 项目中分离，重新命名为 metro-bundler。
#### polyfill
metro-bundler 在进入正式拆包前，先在 js 解释器上挂载  `global.__DEV__` 变量，该变量主要用于区分打包执行环境，同时定义了模块系统函数，包括 `__d`（即 `define`）函数、`require` 函数，这样 RN 也就有了自身的模块系统。

```javascript
// metro-bundler/src/Resolver/index.js
getModuleSystemDependencies({dev = true}: {dev?: boolean}): Array<Module> {
    // global.__DEV__
    const prelude = dev
        ? pathJoin(__dirname, 'polyfills/prelude_dev.js')
        : pathJoin(__dirname, 'polyfills/prelude.js');

    // 返回 polyfills/require.js，定义模块系统函数
    const moduleSystem = defaults.moduleSystem;
    return [prelude, moduleSystem].map(moduleName =>
        // polyfill 处理
        this._depGraph.createPolyfill({
            file: moduleName,
            id: moduleName,
            dependencies: [],
        }),
    );
}
```
最后，这两个模块会被插入到 `response.dependencies` 最前列。
另外，RN 对部分 es6、es7 的方法 polyfill 支持，包括 Array、Object 和 Number 等，具体可以查阅 [polyfills](https://github.com/facebook/react-native/tree/v0.48.1/Libraries/polyfills) 文件，此处不再赘述，感兴趣可以自行研究。
#### 拆包
拆包可以理解为对依赖进行模块预处理，根据资源入口，解析其所依赖的直接模块，调用 Module 构造器解析出模块 `path、_options 和 _transformCode` 等信息，并返回这些依赖自身所依赖的模块，然后，依次以返回的这些模块作为新的资源入口，进行解析直至广度遍历所有的依赖。
拆包具体实现是通过 `_preprocessPotentialDependencies` 方法，以指定的 `entry-file` 作为资源入口，依次解析它的依赖，并利用这些模块，依据 `filePath` 循环调用 `resolveDependency`，存入到 `ModuleCache` 实例中，最后以数组 `dependencyModules` 返回这些实例化后的 module；再次循环，从 `dependencyModules` 数组中依次取最后一个元素，递归调用 `preprocessModule` 和 `tryResolveModuleDependencies` 方法预处理所有的模块；当然，metro-bundler 设置了一个 `Set` 类型的 `visitedModulePaths` 变量进行过滤，实例化一个新的模块时，会将之添加到 `visitedModulePaths` 中，当出现已实例化过的模块，则不会再放入 `dependencyModules` 中，这避免了重复实例化同一个模块的问题。

![模块拆包](https://ws1.sinaimg.cn/large/c4b5f11bly1flhln6pxygj20np0kvjt4.jpg)

```javascript
// metro-bundler/src/node-haste/DependencyGraph/ResolutionRequest.js
_preprocessPotentialDependencies(
    transformOptions: TransformWorkerOptions,
    module: TModule,
    onProgress: (moduleCount: number) => mixed,
): void {
    const visitedModulePaths = new Set();
    // 此处的 module 为入口文件模块
    const pendingBatches = [
        this.preprocessModule(transformOptions, module, visitedModulePaths),
    ];
    onProgress(visitedModulePaths.size);
    while (pendingBatches.length > 0) {
        // dependencyModules：module（作为参数传给 preprocessModule） 依赖的子模块数组，已经过预处理过，数组元素将作为新的入口模块，预处理其子依赖
        const dependencyModules = pendingBatches.pop();
        while (dependencyModules.length > 0) {
            // dependencyModule 作为新的入口模块，预处理其子依赖，通过遍历依次处理所有模块
            const dependencyModule = dependencyModules.pop();
            const deps = this.preprocessModule(
                transformOptions,
                dependencyModule,
                visitedModulePaths,
            );
            // 将预处理过的模块 deps 放入 pendingBatches，达到递归调用目的
            pendingBatches.push(deps);
            onProgress(visitedModulePaths.size);
        }
    }
}
```

#### 去重过滤
在拆包的过程中，会发现多个模块依赖同一个模块的情况，在进行模块预处理时进行的过滤，只能避免对统一模块重复预处理，没有真正将重复的模块过滤掉，因此还需要对重复模块做去重过滤。
去重的实现同样是以 `entry-file` 文件作为入口模块 `entry`，调用 `collectedDependencies.get` 方法，通过其中的 `collect` 函数，将 `entry` 模块的依赖拆分成 `[[depNames1, depNames2, ...], [dependencies1, dependencies2, ...]]` 形式的数组（`depNames` 和 `dependencies` 一一对应，前者为依赖名，后者为具体的预处理过的依赖）；再在 `crawlDependencies` 方法中遍历 `[dependencies1, dependencies2, ...]` 数组，将每个元素，即 `dependencyModules` 依次作为新的资源入口，递归调用 `collectedDependencies.get`，`get` 方法会将入口模块以 `{entry: collect(entry)}` 形式保存到一个 `Map` 中，在保存前会通过 `Map.has(entry)` 检查是否存在，以达到去重的目的。最后，返回 `entry-file` 模块的直接依赖。

```javascript
// metro-bundler/src/node-haste/DependencyGraph/ResolutionRequest.js
// depNames：模块名数组，dependencies：对应模块数组
const crawlDependencies = (mod, [depNames, dependencies]) => {
    const filteredPairs = [];

    dependencies.forEach((modDep, i) => {
        const name = depNames[i];
        if (modDep == null) {
            debug(
                'WARNING: Cannot find required module `%s` from module `%s`',
                name,
                mod.path,
            );
            return false;
        }
        return filteredPairs.push([name, modDep]);
    });

    response.setResolvedDependencyPairs(mod, filteredPairs);
    // 遍历取到所有的模块
    const dependencyModules = filteredPairs.map(([, m]) => m);
    // 统计 newDependencies 的长度主要用于 onProgress 显示进度，可忽略
    const newDependencies = dependencyModules.filter(
        m => !collectedDependencies.has(m),
    );

    if (onProgress) {
        finishedModules += 1;
        totalModules += newDependencies.length;
        onProgress(
            finishedModules,
            Math.max(totalModules, preprocessedModuleCount),
        );
    }

    if (recursive) { // recursive 为标志位，默认为 true
        // doesn't block the return of this function invocation, but defers
        // the resulution of collectionsInProgress.done.then(...)
        // 遍历 dependencyModules，递归调用 collectedDependencies.get
        dependencyModules.forEach(dependency =>
            collectedDependencies.get(dependency),
        );
    }
    // 返回 rootDependencies
    return dependencyModules;
};

// metro-bundler/src/node-haste/lib/MapWithDefaults.js
get(key: TK): TV {
    // 去重
    if (this.has(key)) {
      return Map.prototype.get.call(this, key);
    }
    // _factory 即为 collect 方法
    const value = this._factory(key);
    // 保存 {entry: collect(entry)}
    this.set(key, value);
    return value;
}
```
在上述过滤过程中，仅仅只返回了 `entry-file` 的直接模块，而其他模块只是保存在 `Map` 中，还缺少最后一步 —— 调用 `traverse` 方法依次将这些模块保存到 `response.dependencies` 中，具体实现可以查看流程图和相应代码，由于不难理解，故在此不做更细致的分析了。

![去重过滤](https://ws1.sinaimg.cn/large/c4b5f11bly1flhln6q4ruj20u10i9aca.jpg)


```javascript
return Promise.all([
        // 获取 entry-file 模块所有的直接依赖
        collectedDependencies.get(entry),
        collectionsInProgress.done,
    ])
    .then(([rootDependencies]) => {
        return Promise.all(
            // 遍历 collectedDependencies，获取保存的所有 dependency
            Array.from(collectedDependencies, resolveKeyWithPromise),
        ).then(moduleToDependenciesPairs => [
            rootDependencies,
            new MapWithDefaults(() => [], moduleToDependenciesPairs),
        ]);
    })
    .then(([rootDependencies, moduleDependencies]) => {
        // serialize dependencies, and make sure that every single one is only
        // included once
        const seen = new Set([entry]);
        function traverse(dependencies) {
            dependencies.forEach(dependency => {
                // 再次去重，moduleDependencies 中包含了 rootDependencies，在 插入到 response.dependencies 前需要做这层过滤
                if (seen.has(dependency)) {
                    return;
                }
                seen.add(dependency);
                // 依次将 dependency 插入到 response.dependencies
                response.pushDependency(dependency);
                // 递归调用 traverse
                traverse(moduleDependencies.get(dependency));
            });
        }

        traverse(rootDependencies);
    });
```
#### ModuleTransport 和合并
经过拆分、去重等处理，整个打包过程即将进入尾声 —— 将 `response.dependencies` 中的模块重新组合成一个新的 bundle。
首先，遍历 `response.dependencies` ，对其每个 module 进行 `ModuleTransport` 处理，这里需要区分普通 module 和 asset module，因为对应的解析方法是不同的；解析 module，获取并返回 transformedModule（包括 `code、sourceCode、sourcePath、map 和 polyfill` 等）。
此时，原材料均已准备完毕，开始进入组装阶段 `addModule`，针对 polyfill 处理过的模块调用 `definePolyfillCode` 方法进行包装，即用立即执行函数包装，polyfill 过的代码作为立即执行函数的主体；其他模块则调用 `defineModuleCode` 方法包装，即调用 `__d` 函数来定义模块，同时，将通过构建 `resolveDeps` 对象来处理其他多个模块依赖某一模块问题，在此处不过多分析，有兴趣的可以自行[探索](https://github.com/facebook/metro-bundler/blob/2a64bd390e1d546bf62e1437b8a8aa8a839990a2/packages/metro-bundler/src/Resolver/index.js#L184)。
至此，所有的模块都已定义，将代码放在闭包中，注册到全局的 module 变量中。此时仅仅只是定义了模块，如果要让代码执行起来，最后需要一根线将所有 module 串起来，调用必要的代码 —— 通过 `finalize` 方法 require `InitializeCore` 和 `MainModule`（即 `entry-file` 资源入口）模块。当完成这一步，RN 的打包基本上就完成了。

![代码转化与重组](https://ws1.sinaimg.cn/large/c4b5f11bly1flhfzog8brj20ft0dn74s.jpg)

```javascript
// metro-bundler/src/Bundler/Bundle.js
finalize(options: FinalizeOptions) {
    options = options || {};
    if (options.runModule) {
        // 此处 require InitializeCore 模块
        // RN 在 metro-bundler/src/defaults.js 中默认添加了 InitializeCore 模块
        options.runBeforeMainModule.forEach(this._addRequireCall, this);
        // require MainModule
        this._addRequireCall(this.getMainModuleId());
    }
    // freeze 处理新 bundle 的 __modules 和 __assets
    super.finalize(options);
}
```

## QRN 打包定制化
尽管 RN 官方的打包工具的功能比较完善了，但的确仍有些地方可以进行优化：

+ 打包生成的 bundle 文件太大
+ 对 bundle 进行拆分后，模块关系变动使得分拆后的 bundle 难以保持一致
+ 静态资源离线


### bundle 拆分
metro-bundler 在打包时，会将所有的文件，包括框架组件、业务逻辑代码，都打包到一个 bundle 中，笔者试着将一个没有任何业务逻辑代码的 AwesomeProject 项目，打包生成的 bundle 文件 size 约为 490 k。Qunar 拥有酒店、机票、旅游度假、门票和火车票等多个业务线，随着业务不断扩大、业务复杂性不断上升，生成的 bundle 文件会逐渐增大，将会使得客户端 size 的压力越来越大（虽然从 iOS 的 App Store 上使用流量下载客户端的限制从 100 M 放宽到 150 M）。
metro-bundler 这种打包成一个 bundle 的策略对于 Qunar 来说，必定是不能接受的。在 Hybrid 开发中，在解决文件过大问题的一个策略 —— 拆包，同样，对 RN bundle 拆分，也是适用的。metro-bundler 将项目的所有代码打包成一个 bundle，其中包括 RN 框架 js 文件和一些十分常用的第三方模块，不难发现，多个项目使用 RN 方案，生成的 bundle 文件将会包含多份一模一样的框架 js 和常用的第三方模块文件；因此，我们采取的策略是将框架代码和业务逻辑拆成两个 bundle，这样一方面解决了 bundle 文件大，代码重复问题，另一方面业务只需要将重心放在业务逻辑上，框架由我们来维护。

![jsbundle](https://ws1.sinaimg.cn/large/c4b5f11bly1flggh2fumnj20gy07egly.jpg)

我们在打包时先后调用两次 `bundle` 方法，通过 `bundleType` 参数进行区分框架 bundle 和业务 bundle。同时，在打包时默认不指定 `entry-file`，完全交给打包工具 QRNPackager 来处理，打框架 bundle 时默认指定一个 `platform.js` 文件，该文件 require RN 框架、QRN（包括自定义的组件和模块等），以及一些常用的第三方模块；而业务 bundle 则以项目入口文件为 `entry-file` 进行打包，当然仅仅是这样还不行，我们在 metro-bundler 拆包、去重过滤完成时，对 `response.dependencies` 再处理，通过 `platformDependencies`, `bizDependencies` 两个配置项，分别生成新的 `response.dependencies`。然后执行 metro-bundler 的后续流程。

### 以实际路径代替 module ID 进行引用
metro-bundler 打包处理模块时，以递增的方式给每个模块一个 module ID，使得文件直接通过 `require(module ID)` 的方式引用其他模块；当然，一个项目的所有模块都在一个 bundle 中是没问题的，但进行 bundle 拆分后，当框架新增或删除一个依赖时，因为模块编号的方式使得在该状况下，后续的模块的 module ID 将会错位，这将造成升级框架 bundle 的难度，每次升级，其他业务 bundle 相应的需要回归测试，加大了开发测试成本。我们的选择是直接舍弃 module ID 的引用方式，直接使用模块路径进行引用，这样无论怎么增减模块，都不会出现模块引用错误的问题，做法也比较简单，重写了 `resolutionResponse.getModuleId` 方法，虽然会增大一点 bundle 的 size。

### 静态资源离线
项目开发中少不了使用图片、字体等离线资源的使用，如果这些静态文件直接内置到本地，一方面不方便管理，另一方面也会增大 size 的压力。因此，一般采取的策略是将静态资源上传到服务器，但这也会导致其他问题 —— 用户流量消耗，增加请求，加载资源变慢影响性能。
针对种种问题，我们采取的策略是将开发时将资源上传到服务器，代码中直接通过 url 地址获取资源，同时对打包工具进行了改造，通过不同的配置文件记录项目使用的图片和字体等资源的 url 地址，打包时将之写入到 `staticAssets.json` 文件；另外客户端也进行相应的处理，第一次加载 bundle 时，会进行资源离线处理。当客户端加载静态资源时，请求会被拦截器拦截，优先使用本地的离线静态资源，如果本地没有对应资源，才会去服务器上获取。

## 写在最后
笔者近期在升级 RN 的同时较系统地阅读了 metro-bundler 源码，结合自身的理解整理成该篇文章，希望对想要了解 RN 打包或正在解决 RN 打包问题的朋友有一点帮助。如果行文中有错误，敬请斧正。
