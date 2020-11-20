---
title: webpack源码之运行流程
date: 2018-04-06 
tags: [webpack,源码分析]
author: gcyStar
---

## 引言
通过前面几张的铺垫,下面开始分析webpack源码核心流程,大体上可以分为初始化,编译,输出三个阶段,下面开始分析
## 初始化
> 这个阶段整体流程做了什么? 启动构建，读取与合并配置参数，加载 Plugin，实例化 Compiler。
### 详细分析
```
//通过yargs获得shell中的参数
yargs.parse(process.argv.slice(2), (err, argv, output) => {
    //把webpack.config.js中的参数和shell参数整合到options对象上
    let options;
        options = require("./convert-argv")(argv);

    function processOptions(options) {

        const firstOptions = [].concat(options)[0];
        const webpack = require("webpack");

        let compiler;
            //通过webpack方法创建compile对象,Compiler 负责文件监听和启动编译。
            //Compiler 实例中包含了完整的 Webpack 配置，全局只有一个 Compiler 实例。
            compiler = webpack(options);


        if (firstOptions.watch || options.watch) {

            compiler.watch(watchOptions, compilerCallback);
            //启动一次新的编译。
        } else compiler.run(compilerCallback);
    }

    processOptions(options);
});
```

**说明** 从源码中摘取了初始化的的第一步,做了简化,当运行webpack命令的的时候,运行的是webpack-cli下webpack.js,其内容是一个自执行函数,上面是执行的第一步,进行参数的解析合并处理,并创建compiler实例,然后启动编译运行run方法,其中关键步骤 compiler = webpack(options); 详细展开如下所示

```
const webpack = (options, callback) => {
    //参数合法性校验
    const webpackOptionsValidationErrors = validateSchema(
        webpackOptionsSchema,
        options
    );

    let compiler;
    if (Array.isArray(options)) {
        compiler = new MultiCompiler(options.map(options => webpack(options)));
    } else if (typeof options === "object") {
        options = new WebpackOptionsDefaulter().process(options);
        //创建compiler对象
        compiler = new Compiler(options.context);
        compiler.options = options;
        new NodeEnvironmentPlugin().apply(compiler);
        //注册配置文件中的插件,依次调用插件的 apply 方法，让插件可以监听后续的所有事件节点。同时给插件传入 compiler 实例的引用，以方便插件通过 compiler 调用 Webpack 提供的 API。
        if (options.plugins && Array.isArray(options.plugins)) {
            for (const plugin of options.plugins) {
                plugin.apply(compiler);
            }
        }
        //开始应用 Node.js 风格的文件系统到 compiler 对象，以方便后续的文件寻找和读取。
        compiler.hooks.environment.call();
        compiler.hooks.afterEnvironment.call();
        //注册内部插件
        compiler.options = new WebpackOptionsApply().process(options, compiler);
    }

    return compiler;
};
```
**说明**  注册插件过程不在展开,webpack内置插件真的很多啊
## 编译
> 这个阶段整体流程做了什么? 从 Entry 发出，针对每个 Module 串行调用对应的 Loader 去翻译文件内容，再找到该 Module 依赖的 Module，递归地进行编译处理。
### 详细分析
```
this.hooks.beforeRun.callAsync(this, err => {
			if (err) return finalCallback(err);

			this.hooks.run.callAsync(this, err => {
				if (err) return finalCallback(err);

				this.readRecords(err => {
					if (err) return finalCallback(err);

					this.compile(onCompiled);
				});
			});
		});
```


**说明** 从执行run方法开始,开始执行编译流程,run方法触发了before-run、run两个事件，然后通过readRecords读取文件，通过compile进行打包,该方法中实例化了一个Compilation类

```
compile(callback) {
		const params = this.newCompilationParams();
		this.hooks.beforeCompile.callAsync(params, err => {
			if (err) return callback(err);

			this.hooks.compile.call(params);
// 每编译一次都会创建一个compilation对象（比如watch 文件时，一改动就会执行），但是compile只会创建一次
			const compilation = this.newCompilation(params);
// make事件触发了  事件会触发SingleEntryPlugin监听函数，调用compilation.addEntry方法
			this.hooks.make.callAsync(compilation, err => {
				if (err) return callback(err);
				
			});
		});
	}
```

**说明**  打包时触发before-compile、compile、make等事件,同时创建非常重要的compilation对象,内部有声明了很多钩子,初始化模板等等


```
this.hooks = {
    buildModule: new SyncHook(["module"]),
    seal: new SyncHook([]),
    optimize: new SyncHook([]),
};
//拼接最终生成代码的主模板会用到
this.mainTemplate = new MainTemplate(this.outputOptions);
//拼接最终生成代码的chunk模板会用到
this.chunkTemplate = new ChunkTemplate(this.outputOptions); 
 //拼接最终生成代码的热更新模板会用到
this.hotUpdateChunkTemplate = new HotUpdateChunkTemplate()

```

```
//监听comple的make hooks事件，通过内部的 SingleEntryPlugin 从入口文件开始执行编译
		compiler.hooks.make.tapAsync(
			"SingleEntryPlugin",
			(compilation, callback) => {
				const { entry, name, context } = this;

				const dep = SingleEntryPlugin.createDependency(entry, name);
				compilation.addEntry(context, dep, name, callback);
			}
		);
```

**说明**  监听compile的make hooks事件，通过内部的 SingleEntryPlugin 从入口文件开始执行编译,调用compilation.addEntry方法,根据模块的类型获取对应的模块工厂并创建模块,开始构建模块

```

doBuild(options, compilation, resolver, fs, callback) {
    const loaderContext = this.createLoaderContext(
        resolver,
        options,
        compilation,
        fs
    );
    //调用loader处理模块
    runLoaders(
        {
            resource: this.resource,
            loaders: this.loaders,
            context: loaderContext,
            readResource: fs.readFile.bind(fs)
        },
        (err, result) => {
           
            
            const resourceBuffer = result.resourceBuffer;
            const source = result.result[0];
            const sourceMap = result.result.length >= 1 ? result.result[1] : null;
            const extraInfo = result.result.length >= 2 ? result.result[2] : null;
            

            this._source = this.createSource(
                this.binary ? asBuffer(source) : asString(source),
                resourceBuffer,
                sourceMap
            );
            //loader处理完之后 得到_source  然后ast接着处理
            this._ast =
                typeof extraInfo === "object" &&
                extraInfo !== null &&
                extraInfo.webpackAST !== undefined
                    ? extraInfo.webpackAST
                    : null;
            return callback();
        }
    );
}
```

**说明**  SingleEntryPlugin这个内存插件主要作用是从entry读取文件,根据文件类型和配置的 Loader 执行runLoaders,然后将loader处理后的文件通过acorn抽象成抽象语法树AST,遍历AST，构建该模块的所有依赖。

## 输出
> 这个阶段整体流程做了什么? 把编译后的 Module 组合成 Chunk，把 Chunk 转换成文件，输出到文件系统。
### 详细分析
```
 //所有依赖build完成，开始对chunk进行优化（抽取公共模块、加hash等）
compilation.seal(err => {
	if (err) return callback(err);

	this.hooks.afterCompile.callAsync(compilation, err => {
		if (err) return callback(err);

		return callback(null, compilation);
	});
});

```
**说明**  compilation.seal主要是对chunk进行优化,生成编译后的源码,比较重要,详细展开如下所示

```
//代码生成前优化
this.hooks.optimize.call();
this.hooks.optimizeTree.callAsync(this.chunks, this.modules, err => {
 
    this.hooks.beforeHash.call();
    this.createHash();
    this.hooks.afterHash.call();

    if (shouldRecord) this.hooks.recordHash.call(this.records);

    this.hooks.beforeModuleAssets.call();
    this.createModuleAssets();
    if (this.hooks.shouldGenerateChunkAssets.call() !== false) {
        this.hooks.beforeChunkAssets.call();
        //生成最终打包输出的chunk资源,根据template文件,详细步骤如下所示
        this.createChunkAssets();
    }
    
});
--------------------------------------
//取出最后文件需要的模板
const template = chunk.hasRuntime()
					? this.mainTemplate
					: this.chunkTemplate;
//通过模板最终生成webpack_require格式的内容,他这个是内部封装的拼接渲染逻辑,也没用什么ejs,handlebar等这些模板工具
source = fileManifest.render();
//生成的资源保存在compilation.assets,方便下一步emitAssets步骤中,把文件输出到硬盘
this.assets[file] = source;
```

```
    //把处理好的assets输出到output的path中
	emitAssets(compilation, callback) {
        let outputPath;
    
        const emitFiles = err => {
            if (err) return callback(err);
    
            asyncLib.forEach(
                compilation.assets,
                (source, file, callback) => {
                    const writeOut = err => {
                        //输出打包后的文件到配置中指定的目录下
                        this.outputFileSystem.writeFile(targetPath, content, callback);
                    };
    
                    writeOut();
                }
            );
        };
    
        this.hooks.emit.callAsync(compilation, err => {
            if (err) return callback(err);
            outputPath = compilation.getPath(this.outputPath);
            this.outputFileSystem.mkdirp(outputPath, emitFiles);
        });
    }

```

## 总结
如果单独看这篇文章的话,理解起来会比较困难,推荐一下与之相关的系列铺垫文章,上面是我对webpack源码运行流程的总结,  整个流程已经跑通了,不过还有蛮多点值得深入挖掘的。清明在家宅了3天,过得好快,明天公司组织去奥森公园寻宝行动,期待ing 。

推荐
[webpack源码之tapable](https://segmentfault.com/a/1190000014031536)
[webpack源码之plugin机制](https://segmentfault.com/a/1190000014056619)
[webpack源码之ast简介](https://segmentfault.com/a/1190000014178462)
[webpack源码之loader机制](https://segmentfault.com/a/1190000014205729)

参考源码
webpack: "4.4.1"
webpack-cli: "2.0.13"
 
