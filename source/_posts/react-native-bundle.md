title: react-native bundle 解释与拆解
date: 2017-09-06 12:37:53
tags:
---

## 0. 成果

> 声明: 避免修改 RN 依赖下面代码，fork [facebook/metro-bundler](https://github.com/facebook/metro-bundler) 进行修改
> github: [https://github.com/4ndroidev/metro-bundler](https://github.com/4ndroidev/metro-bundler)

原理：

细心分析， bundle 文件中每一行都是一个`Module.js`对应的数据结构；打包过程中，在分析依赖时，引入`base.js`先进行基础依赖遍历，并对`Module`元素标记`base: true`，然后再对入口文件进行依赖分析，这种先后顺序能保证基础模块 id 在前，业务模块 id 在后；在打包输出时，将标记`base: true`的`Module`打包到`base.bundle`，否则打包到`business bundle`

使用方式：

```shel
# 安装打包工具
npm install rocket-bundler

# or

yarn add rocket-bundler

# 打包bundle

test ! -d output && mkdir output

node node_modules/rocket-bundler/src/cli.js bundle \
  --dev false \
  --platform android \
  --entry-file index.android.js \
  --bundle-output output/index.android.bundle \
  --base-file base.js \
  --base-output output/base.bundle \
  --assets-dest output/ \
  --sourcemap-output output/sourcemap.txt \
```

结果：

![bundle-result](/images/react-native-bundle/bundle-result.png)

<!-- more -->

android 加载示例：

```java

//不修改源码，不通过反射方式加载，事实上加载脚本的代码是 JSCExecutor.cpp 的 evaluateScript 方法

private class BaseBundleLoader extends JSBundleLoader {

  private String bundleLocation;

  BaseBundleLoader(String location) {
    bundleLocation = location;
  }

  private boolean existAssetBaseBundle() {
    try {
      String[] assets = application.getAssets().list(BUNDLE_ASSET_FOLDER);
      for (String asset : assets) {
        if (BASE_BUNDLE_NAME.equals(asset))
          return true;
      }
    } catch (IOException e) {
      e.printStackTrace();
    }
    return false;
  }

  @Override
  public String loadScript(CatalystInstanceImpl instance) {
    File bundle = new File(bundleLocation);
    File base = new File(bundle.getParent(), BASE_BUNDLE_NAME);
    if (base.exists()) {
      JSBundleLoader.createFileLoader(base.getPath()).loadScript(instance);
    } else if (existAssetBaseBundle()) {
      JSBundleLoader.createAssetLoader(application, BASE_BUNDLE_ASSET, false).loadScript(instance);
    }
    JSBundleLoader.createFileLoader(bundleLocation).loadScript(instance);
    return bundleLocation;
  }
}
```

## 1. 需求

在实际 RN 开发中，往往涉及多个业务，业务间可能不存在耦合，而且业务需要独立的`bundle`，由此就会出现多个`bundle`的情况，而每个`bundle`基本上都包含`react`和`react-native`依赖，导致总体积较大；一般存在线上下发需求，还会耗费用户流量。

由经验可知，JS 中的`react`和`react-native`依赖版本基本上与原生 RN 版本对应， 可以抽离这两个依赖打包成`base.bundle`内置于`app`中

## 2. 目标

- 抽离`react`和`react-native`打包成`base.bundle`
- 减小线上下发业务`bundle`体积，节省用户流量
- 可预加载`base.bundle`，提升打开页面速度

## 3. 分析

> 通过分析bundle结构和依赖查找，最终可以通过标记法进行分包，下文逐一讲解

### 3.1 bundle 结构分析

![image](/images/react-native-bundle/bundle-structure.png)

- polyfills : 最早执行的一些function，声明es语法新增的接口，定义模块声明方法`__d`等
- module difinitations : 模块声明，以`__d`开头，每一行代表一个JS的定义
- require calls : 执行`InitializeCore`和`Entry File`，最后一行`require(0);`

### 3.2 bundle 代码分析

由 3.1 可知，JSLoader加载`bundle`时，优先执行 `polyfill function`，然后定义`JS模块`，接着使用 `require` 调用 `init` 和 `入口文件`

`Resolver/polyfills/require.js` 分析 （代码缩略展示）

```javascript
...

type FactoryFn = (
  global: Object,
  require: RequireFn,
  moduleObject: {exports: {}},
  exports: {},
  dependencyMap: ?DependencyMap,
) => void;

global.require = require;
global.__d = define;

function define(
  factory: FactoryFn,
  moduleId: number,
  dependencyMap?: DependencyMap,
) {
  ...
}

function require(moduleId: ModuleID | VerboseModuleNameForDev) {
  ....
}
...

```

由上述代码可知，
- `__d`实际是`require.js`的`define`方法，
- `__d`参数列表为(`factory: FactoryFn`, `moduleId: number`, `dependencyMap?: DependencyMap`)

再贴上混淆打包后的入口模块代码，分析其意义

```javascript
__d(function(e,t,r,i){"use strict";var n=t(24),p=t(279),s=babelHelpers.interopRequireDefault(p);n.AppRegistry.registerComponent("index",s.default)},0);
```

- `__d` : require.js 的 define 方法

|__d参数|意义|
|---|---|
|function(e, t, r, i)|factory方法|
|0|模块id，目前入口文件的id必为0，id按照深度遍历方式递增|

|factory参数|意义|
|---|---|
|e|global对象|
|t|require方法|
|r|模块对象|
|i|模块暴露|


### 3.3 bundle 依赖分析

> 打包bundle时，根据entryFile进行深度遍历依赖分析，模块id不断递增，即越早引用的模块，id越小
> 下文针对0.46.0+代码为抽离`react`和`react-native`作`base.bundle`分析

从 `node node_modules/react-native/local-cli/cli.js bundle ....`出发，调用链如下：

```javascript
// package: react-native  file: cli.js
// 开始执行
cliEntry.run(); 

// package: react-native  file: cliEntry.js
// 解释命令
commander.parse(process.argv); 

// package: react-native  file: bundle.js
// 开始打包bundle
buildBundle(args, config, output, packagerInstance); 

// package: react-native  file: buildBundle.js
// 创建 packagerInstance，调用metro-bundler的build方法，packagerInstance接着会查找依赖
packagerInstance = new Server(options);
output.build(packagerInstance, requestOpts); 

// package: metro-bundler  file: Server/index.js
// 根据 entryFile 查找依赖
getDependencies(
    options: DependencyOptions,
  ): Promise<ResolutionResponse<Module, *>> {
    return Promise.resolve().then(() => {
      const platform =
        options.platform != null
          ? options.platform
          : parsePlatformFilePath(options.entryFile, this._platforms).platform;
      const {entryFile, dev, minify, hot, rootEntryFile} = options;
      return this._bundler.getDependencies({
        entryFile,
        platform,
        dev,
        minify,
        hot,
        generateSourceMaps: false,
        rootEntryFile,
      });
    });
  }

// package: metro-bundler  file: Bundler.js
// 作为中转，叫小弟Resolver进行依赖查找，中间有自己的一些操作，不详介绍
async getDependencies({
    entryFile,
    platform,
    dev = true,
    minify = !dev,
    hot = false,
    recursive = true,
    generateSourceMaps = false,
    isolateModuleIDs = false,
    rootEntryFile,
    onProgress,
  }): Promise<ResolutionResponse<Module, BundlingOptions>> {
    ...

    const resolver = await this._resolverPromise;
    const response = await resolver.getDependencies(
      entryFile,
      {dev, platform, recursive},
      bundlingOptions,
      onProgress,
      isolateModuleIDs ? createModuleIdFactory() : this._getModuleId,
    );
    return response;
  }

// package: metro-bundler  file: Resolver/index.js
// 作为中转，叫小弟DependencyGraph进行依赖查找，中间有自己的一些操作，不详介绍
getDependencies<T: ContainsTransformerOptions>(
  entryPath: string,
  options: {platform: ?string, recursive?: boolean},
  bundlingOptions: T,
  onProgress?: ?(finishedModules: number, totalModules: number) => mixed,
  getModuleId: mixed,
): Promise<ResolutionResponse<Module, T>> {
  const {platform, recursive = true} = options;
  return this._depGraph
    .getDependencies({
      entryPath,
      platform,
      options: bundlingOptions,
      recursive,
      onProgress,
    })
    .then(resolutionResponse => {
      this._getPolyfillDependencies(platform)
        .reverse()
        .forEach(polyfill => resolutionResponse.prependDependency(polyfill));

        resolutionResponse.getModuleId = getModuleId;
        return resolutionResponse.finalize();
    });
}

// package: metro-bundler  file: DependencyGraph
// 劳动人民，这个很关键，可作为抽离base.bundle入口
getDependencies<T: {+transformer: JSTransformerOptions}>({
    entryPath,
    options,
    platform,
    onProgress,
    recursive = true,
  }): Promise<ResolutionResponse<Module, T>> {
    platform = this._getRequestPlatform(entryPath, platform);
    const absPath = this._getAbsolutePath(entryPath);

    const entry = this._moduleCache.getModule(absPath);

    const response = new ResolutionResponse(options);

    const req = new ResolutionRequest({
      moduleResolver: this._moduleResolver,
      entryPath: absPath,
      helpers: this._helpers,
      platform: platform != null ? platform : null,
      moduleCache: this._moduleCache,
    });

    return req.getOrderedDependencies({
      response,
      transformOptions: options.transformer,
      onProgress,
      recursive,
    })
    .then(()=>response);
  }


```

## 4. 拆包实现
> 上文说到使用标记法进行分包，是指在分析依赖期间，标记哪些模块属于base.bundle，哪些模块属于业务bundle；
> 特别地，polyfills属于base.bundle，require-calls和entry-file属于业务bundle

实现： 在`DependencyGraph.js`的`getDependencies`方法中，引入`base.js`，先收集base.bundle的模块，标记成base，接着再根据`entryFile`进行依赖收集，保证了base.bundle的模块id都在前面。[最少修改代码实现](https://github.com/4ndroidev/metro-bundler/commit/f6f45a3cf0572e97b83adc7c6d5aa9d18cc0cebc)

```javascript

// package: rocket-bundler  file: DependencyGraph
getDependencies<T: {+transformer: JSTransformerOptions}>({
    entryPath,
    options,
    platform,
    onProgress,
    recursive = true,
  }): Promise<ResolutionResponse<Module, T>> {
    platform = this._getRequestPlatform(entryPath, platform);
    const absPath = this._getAbsolutePath(entryPath);

    const entry = this._moduleCache.getModule(absPath);

    const response = new ResolutionResponse(options);

    response.pushDependency(entry);

    const seen = new Set([entry]);

    const req = new ResolutionRequest({
      moduleResolver: this._moduleResolver,
      entryPath: absPath,
      helpers: this._helpers,
      platform: platform != null ? platform : null,
      moduleCache: this._moduleCache,
    });

    const basePath = global.baseFile ? this._getAbsolutePath(global.baseFile) : undefined;

    const basePromise = !basePath ? Promise.resolve(true) : 
      new ResolutionRequest({
        moduleResolver: this._moduleResolver,
        entryPath: basePath,
        helpers: this._helpers,
        platform: platform != null ? platform : null,
        moduleCache: this._moduleCache,
      }).getOrderedDependencies({
        response,
        transformOptions: options.transformer,
        onProgress,
        recursive,
        base: true,
        seen,
      });

    return basePromise.then(()=>req.getOrderedDependencies({
      response,
      transformOptions: options.transformer,
      onProgress,
      recursive,
      base: false,
      seen,
    }))
    .then(()=>response);
  }

// package: rocket-bundler  file: ResolutionRequest
function traverse(dependencies) {
  dependencies.forEach(dependency => {
    if (seen.has(dependency)) {
      return;
    }

    dependency.base = base;
    seen.add(dependency);
    response.pushDependency(dependency);
    traverse(moduleDependencies.get(dependency));
  });
}

```
输出bundle文件代码：

```
// package:rocket-bundler  file: BundleBase.js
getBase(options: GetSourceOptions) {
  this.assertFinalized();

  if (this._base) {
    return this._base;
  }

  this._base = this.__modules.filter(module => module.base).map(module => module.code).join('\n');
  return this._base;
}

getSource(options: GetSourceOptions) {
  this.assertFinalized();

  if (this._source) {
    return this._source;
  }

  this._source = this.__modules.filter(module => !module.base).map(module => module.code).join('\n');
  return this._source;
}

// package: rocket-bundler  file: bundle.js
function saveBundleAndMap(
  bundle: Bundle,
  options: OutputOptions,
  log: (...args: Array<string>) => {},
): Promise<> {
  const {
    bundleOutput,
    bundleEncoding: encoding,
    dev,
    sourcemapOutput,
    sourcemapSourcesRoot,
  } = options;

  log('start');
  const base = createBase(bundle, !!dev);
  const origCodeWithMap = createCodeWithMap(bundle, !!dev, sourcemapSourcesRoot);
  const codeWithMap = bundle.postProcessBundleSourcemap({
    ...origCodeWithMap,
    outFileName: bundleOutput,
  });
  log('finish');

  log('Writing bundle output to:', bundleOutput);

  const {code} = codeWithMap;
  const baseOutput = options.baseFile ? options.baseOutput : undefined;
  const writeBase = baseOutput ? writeFile(baseOutput, base.code, encoding) : Promise.resolve(true);
  const writeBundle = writeFile(bundleOutput, code, encoding);
  const writeMetadata = writeFile(
    bundleOutput + '.meta',
    meta(code, encoding),
    'binary');
  Promise.all([writeBase, writeBundle, writeMetadata])
    .then(() => log('Done writing bundle output'));

  if (sourcemapOutput) {
    log('Writing sourcemap output to:', sourcemapOutput);
    const map = typeof codeWithMap.map !== 'string'
      ? JSON.stringify(codeWithMap.map)
      : codeWithMap.map;
    const writeMap = writeFile(sourcemapOutput, map, null);
    writeMap.then(() => log('Done writing sourcemap output'));
    return Promise.all([writeBundle, writeMetadata, writeMap]);
  } else {
    return writeBundle;
  }
}
```

## 5. 总结

优点： 

- 一次性打出base.bundle和业务bundle，效率高
- 可自定义哪些模块属于base.bundle
- 原生代码，可预加载base.bundle

缺点：

- 维护成本较高
- 事实上，直接引用`react-native`作为基础，可能会引入一些你用不到的模块。其实我不推介直接引用`react-native`，用到其中模块直接引用，这样也能减小部分体积

总体来说，利大于弊，目前真没看到几个开发同事不直接引`react-native`，哈哈

## 6. 展望

从上述分包方案，理论上可以制定规则，解耦业务，根据不同业务模块，划分更多bundle，每个bundle的模块id按照某个值开始，避免重复，类似android插件化处理资源id策略，按需加载业务bundle，另外可能带来管理困难的问题。

## 7. 附加

事实上，还有更易于维护的方法，前后对`base.js`和`index.js`进行`bundle`，然后以这两个bundle作为输入，进行字符串操作，输出最后我们想要的分包结果。但前提是：`index.js`最开始的依赖引用必须与`base.js`一致，保证两者打包的基础模块 id 一致。分包代码如下：

```javascript
const fs = require('fs');
const os = require('os'); 
const readline = require('readline');
const REQUIRE_CALL_PATTERN = /^;require\(\d+\);$/
const ENTRY_FILE_PATTERN = /^__d\(.*,0\);$/

function readcontent(path, filter){
  return new Promise(function(resolve, reject){
    if(!fs.existsSync(path)) {
      reject('fileNotFound: '+ path);
    }else{
      var lines = [];
      var stream = fs.createReadStream(path);
      var lineInterface = readline.createInterface({input: stream});
      lineInterface.on('line', function(line){
        if(!filter || filter(line))
          lines.push(line); 
      });
      lineInterface.on('close', function(){ 
        resolve(lines); 
      });
    }
  });
}

function cut(contents){
  var base = contents[0];
  var business = contents[1];
  var temp = []; // avoid problem of read and write synchronously 
  for(var i=0;i<business.length;i++){
    var line = business[i];
    if(base.indexOf(line)>=0) continue;
    temp.push(line);
  }
  business.splice(0);
  business = temp;
  return [base, business];
}

function save(paths, bundles){
  paths.forEach(function(path, pathIndex){
    var stream = fs.createWriteStream(path);
    bundles[pathIndex].forEach(function(line, lineIndex){
      if(!!lineIndex) stream.write(os.EOL);
      stream.write(line);
    });
  });
}

function run(paths){
  if(paths.length!=2) throw new Error('you can only pass two arguments, one is `path of base.bundle`, the other is `path of business bundle`!');
  var basefile = paths[0];
  var businessfile = paths[1];
  var basefilter = function(line){ return !REQUIRE_CALL_PATTERN.test(line) && !ENTRY_FILE_PATTERN.test(line); }
  Promise.all([
    readcontent(basefile, basefilter), 
    readcontent(businessfile)
  ])
  .then(function(contents) {
    return cut(contents);
  })
  .then(function(bundles){ 
    save(paths, bundles); 
  })
  .catch(function(reason){
    throw new Error(reason);
  });
}

function main(){
  run(process.argv.splice(2));
}

if (require.main === module) {
  main();
}

module.exports = {
  run: run
}
```