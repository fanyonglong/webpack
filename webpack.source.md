# webpac4 阅读
- [webpac4 阅读](#webpac4-%E9%98%85%E8%AF%BB)
	- [Webpack](#Webpack)
	- [Compiler](#Compiler)
	- [Compilation](#Compilation)
	- [Resolver](#Resolver)
	- [SingleEntryDependency](#SingleEntryDependency)
	- [NormalModuleFactory](#NormalModuleFactory)


## 构建就是把源代码转换成发布到线上的可执行JavaScrip、CSS、HTML代码，包括如下内容：

* 代码校验：在代码被提交到仓库前需要校验代码是否符合规范，以及单元测试是否通过。
* 代码转换：TypeScript 编译成 JavaScript、SCSS 编译成 CSS 等。
* 模块合并：在采用模块化的项目里会有很多个模块和文件，需要构建功能把模块分类合并成一个文件。
* 代码分割：提取多个页面的公共代码、提取首屏不需要执行部分的代码让其异步加载。
* 文件优化：压缩 JavaScript、CSS、HTML 代码，压缩合并图片等。
* 自动刷新：监听本地源代码的变化，自动重新构建、刷新浏览器。
* 自动发布：更新完代码后，自动构建出线上发布代码并传输给发布系统。

## webapck核心概念

* Entry：入口，Webpack 执行构建的第一步将从 Entry 开始，可抽象成输入。
* Output：输出结果，在Webpack经过一系列处理并得出最终想要的代码后输出结果。
* Module：模块，在 Webpack里一切皆模块，一个模块对应着一个文件。Webpack会从配置的 Entry 开始递归找出所有依赖的模块。
* Chunk：代码块，一个 Chunk 由多个模块组合而成，用于代码合并与分割。
* Loader：模块转换器，用于把模块原内容按照需求转换成新内容。
* Plugin：扩展插件，在Webpack构建流程中的特定时机注入扩展逻辑来改变构建结果或做你想要的事情。

## webpack工作过程

Webpack 启动后会从Entry里配置的Module开始递归解析 Entry 依赖的所有 Module。
每找到一个 Module， 就会根据配置的Loader去找出对应的转换规则，对Module 进行转换后，再解析出当前 Module 依赖的 Module。 这些模块会以 Entry 为单位进行分组，一个 Entry 和其所有依赖的 Module 被分到一个组也就是一个 Chunk。
最后 Webpack 会把所有 Chunk 转换成文件输出。 在整个流程中 Webpack 会在恰当的时机执行Plugin 里定义的逻辑。

## Webpack
下面是webpack.js部分源码，可以从代码看编译一部分运行流程.
从下这个例子去看webpack是怎么一步一步去解析.

```js
/**
 * 目录 
 * src
 *    b.js
 *    index.js
 * 
 * ##index.js 
 * export default 'index';
*/
var compiler=webpack({
    entry:'./src/index',
    plugins[
        // 定义自定义插件
        {
            apply(compiler){
              
            }
        }
    ]
});
compiler.run(callback);//编译
```
```js
    // 设置配置项
	options = new WebpackOptionsDefaulter().process(options);
    compiler = new Compiler(options.context);// 创建编译器
    compiler.options = options;
    /*
        node环境文件系统配置
        compiler.inputFileSystem 文件读取
		compiler.outputFileSystem 文件输出
		compiler.watchFileSystem 监听文件
    */
	new NodeEnvironmentPlugin().apply(compiler);
	if (options.plugins && Array.isArray(options.plugins)) {
			for (const plugin of options.plugins) {
				if (typeof plugin === "function") {
					plugin.call(compiler, compiler);
				} else {
					plugin.apply(compiler);
				}
			}
		}
		compiler.hooks.environment.call();
        compiler.hooks.afterEnvironment.call();
        // 执行一系列内置插件,定义和触发一些钩子
		compiler.options = new WebpackOptionsApply().process(options, compiler);
```



## Compiler
```js
run(callback){

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
}	
createNormalModuleFactory() {
		const normalModuleFactory = new NormalModuleFactory(
			this.options.context,
			this.resolverFactory,
			this.options.module || {}
		);
		this.hooks.normalModuleFactory.call(normalModuleFactory);
		return normalModuleFactory;
}
newCompilationParams() {
		const params = {
			normalModuleFactory: this.createNormalModuleFactory(),
			contextModuleFactory: this.createContextModuleFactory(),
			compilationDependencies: new Set()
		};
		return params;
}
newCompilation(params) {
		const compilation = this.createCompilation();
		compilation.fileTimestamps = this.fileTimestamps;
		compilation.contextTimestamps = this.contextTimestamps;
		compilation.name = this.name;
		compilation.records = this.records;
		compilation.compilationDependencies = params.compilationDependencies;
		this.hooks.thisCompilation.call(compilation, params);
		this.hooks.compilation.call(compilation, params);
		return compilation;
}
compile(callback) {
		const params = this.newCompilationParams();
		this.hooks.beforeCompile.callAsync(params, err => {
			if (err) return callback(err);

			this.hooks.compile.call(params);

			const compilation = this.newCompilation(params);
			// 编译,如果有异步编译需求，可以在此钩子下处理
			this.hooks.make.callAsync(compilation, err => {
				if (err) return callback(err);

				compilation.finish(err => {
					if (err) return callback(err);

					compilation.seal(err => {
						if (err) return callback(err);

						this.hooks.afterCompile.callAsync(compilation, err => {
							if (err) return callback(err);

							return callback(null, compilation);
						});
					});
				});
			});
		});
}


```
## Compilation
```js

	finish(callback) {
		const modules = this.modules;
		this.hooks.finishModules.callAsync(modules, err => {
			if (err) return callback(err);

			for (let index = 0; index < modules.length; index++) {
				const module = modules[index];
				this.reportDependencyErrorsAndWarnings(module, [module]);
			}

			callback();
		});
	}
	/**
	 * @param {Callback} callback signals when the seal method is finishes
	 * @returns {void}
	 */
	seal(callback) {
		this.hooks.seal.call();
		// 优化输出
		while (
			this.hooks.optimizeDependenciesBasic.call(this.modules) ||
			this.hooks.optimizeDependencies.call(this.modules) ||
			this.hooks.optimizeDependenciesAdvanced.call(this.modules)
		) {
			/* empty */
		}
		this.hooks.afterOptimizeDependencies.call(this.modules);

		this.hooks.beforeChunks.call();
		for (const preparedEntrypoint of this._preparedEntrypoints) {
			const module = preparedEntrypoint.module;
			const name = preparedEntrypoint.name;
			const chunk = this.addChunk(name);
			const entrypoint = new Entrypoint(name);
			entrypoint.setRuntimeChunk(chunk);
			entrypoint.addOrigin(null, name, preparedEntrypoint.request);
			this.namedChunkGroups.set(name, entrypoint);
			this.entrypoints.set(name, entrypoint);
			this.chunkGroups.push(entrypoint);

			GraphHelpers.connectChunkGroupAndChunk(entrypoint, chunk);
			GraphHelpers.connectChunkAndModule(chunk, module);

			chunk.entryModule = module;
			chunk.name = name;

			this.assignDepth(module);
		}
		this.processDependenciesBlocksForChunkGroups(this.chunkGroups.slice());
		this.sortModules(this.modules);
		this.hooks.afterChunks.call(this.chunks);

		this.hooks.optimize.call();

		while (
			this.hooks.optimizeModulesBasic.call(this.modules) ||
			this.hooks.optimizeModules.call(this.modules) ||
			this.hooks.optimizeModulesAdvanced.call(this.modules)
		) {
			/* empty */
		}
		this.hooks.afterOptimizeModules.call(this.modules);

		while (
			this.hooks.optimizeChunksBasic.call(this.chunks, this.chunkGroups) ||
			this.hooks.optimizeChunks.call(this.chunks, this.chunkGroups) ||
			this.hooks.optimizeChunksAdvanced.call(this.chunks, this.chunkGroups)
		) {
			/* empty */
		}
		this.hooks.afterOptimizeChunks.call(this.chunks, this.chunkGroups);

		this.hooks.optimizeTree.callAsync(this.chunks, this.modules, err => {
			if (err) {
				return callback(err);
			}

			this.hooks.afterOptimizeTree.call(this.chunks, this.modules);

			while (
				this.hooks.optimizeChunkModulesBasic.call(this.chunks, this.modules) ||
				this.hooks.optimizeChunkModules.call(this.chunks, this.modules) ||
				this.hooks.optimizeChunkModulesAdvanced.call(this.chunks, this.modules)
			) {
				/* empty */
			}
			this.hooks.afterOptimizeChunkModules.call(this.chunks, this.modules);

			const shouldRecord = this.hooks.shouldRecord.call() !== false;

			this.hooks.reviveModules.call(this.modules, this.records);
			this.hooks.optimizeModuleOrder.call(this.modules);
			this.hooks.advancedOptimizeModuleOrder.call(this.modules);
			this.hooks.beforeModuleIds.call(this.modules);
			this.hooks.moduleIds.call(this.modules);
			this.applyModuleIds();
			this.hooks.optimizeModuleIds.call(this.modules);
			this.hooks.afterOptimizeModuleIds.call(this.modules);

			this.sortItemsWithModuleIds();

			this.hooks.reviveChunks.call(this.chunks, this.records);
			this.hooks.optimizeChunkOrder.call(this.chunks);
			this.hooks.beforeChunkIds.call(this.chunks);
			this.applyChunkIds();
			this.hooks.optimizeChunkIds.call(this.chunks);
			this.hooks.afterOptimizeChunkIds.call(this.chunks);

			this.sortItemsWithChunkIds();

			if (shouldRecord) {
				this.hooks.recordModules.call(this.modules, this.records);
				this.hooks.recordChunks.call(this.chunks, this.records);
			}

			this.hooks.beforeHash.call();
			this.createHash();
			this.hooks.afterHash.call();

			if (shouldRecord) {
				this.hooks.recordHash.call(this.records);
			}

			this.hooks.beforeModuleAssets.call();
			this.createModuleAssets();
			if (this.hooks.shouldGenerateChunkAssets.call() !== false) {
				this.hooks.beforeChunkAssets.call();
				this.createChunkAssets();
			}
			this.hooks.additionalChunkAssets.call(this.chunks);
			this.summarizeDependencies();
			if (shouldRecord) {
				this.hooks.record.call(this, this.records);
			}

			this.hooks.additionalAssets.callAsync(err => {
				if (err) {
					return callback(err);
				}
				this.hooks.optimizeChunkAssets.callAsync(this.chunks, err => {
					if (err) {
						return callback(err);
					}
					this.hooks.afterOptimizeChunkAssets.call(this.chunks);
					this.hooks.optimizeAssets.callAsync(this.assets, err => {
						if (err) {
							return callback(err);
						}
						this.hooks.afterOptimizeAssets.call(this.assets);
						if (this.hooks.needAdditionalSeal.call()) {
							this.unseal();
							return this.seal(callback);
						}
						return this.hooks.afterSeal.callAsync(callback);
					});
				});
			});
		});
	}
/**
	 * Builds the module object
	 *
	 * @param {Module} module module to be built
	 * @param {boolean} optional optional flag
	 * @param {Module=} origin origin module this module build was requested from
	 * @param {Dependency[]=} dependencies optional dependencies from the module to be built
	 * @param {TODO} thisCallback the callback
	 * @returns {TODO} returns the callback function with results
	 */
	buildModule(module, optional, origin, dependencies, thisCallback) {
		let callbackList = this._buildingModules.get(module);
		if (callbackList) {
			callbackList.push(thisCallback);
			return;
		}
		this._buildingModules.set(module, (callbackList = [thisCallback]));

		const callback = err => {
			this._buildingModules.delete(module);
			for (const cb of callbackList) {
				cb(err);
			}
		};

		this.hooks.buildModule.call(module);
		module.build(
			this.options,
			this,
			this.resolverFactory.get("normal", module.resolveOptions),
			this.inputFileSystem,
			error => {
				const errors = module.errors;
				for (let indexError = 0; indexError < errors.length; indexError++) {
					const err = errors[indexError];
					err.origin = origin;
					err.dependencies = dependencies;
					if (optional) {
						this.warnings.push(err);
					} else {
						this.errors.push(err);
					}
				}

				const warnings = module.warnings;
				for (
					let indexWarning = 0;
					indexWarning < warnings.length;
					indexWarning++
				) {
					const war = warnings[indexWarning];
					war.origin = origin;
					war.dependencies = dependencies;
					this.warnings.push(war);
				}
				const originalMap = module.dependencies.reduce((map, v, i) => {
					map.set(v, i);
					return map;
				}, new Map());
				module.dependencies.sort((a, b) => {
					const cmp = compareLocations(a.loc, b.loc);
					if (cmp) return cmp;
					return originalMap.get(a) - originalMap.get(b);
				});
				if (error) {
					this.hooks.failedModule.call(module, error);
					return callback(error);
				}
				this.hooks.succeedModule.call(module);
				return callback();
			}
		);
}
_addModuleChain(context, dependency, onModule, callback) {
    //代码....
    const Dep = /** @type {DepConstructor} */ (dependency.constructor);
    const moduleFactory = this.dependencyFactories.get(Dep);
    this.semaphore.acquire(() => {
			moduleFactory.create(
				{
					contextInfo: {
						issuer: "",
						compiler: this.compiler.name
					},
					context: context,
					dependencies: [dependency]
				},
				(err, module) => {				
                    // 添加module 到this.modules中
					const addModuleResult = this.addModule(module);
					module = addModuleResult.module;
                    
					onModule(module);

					dependency.module = module;
					module.addReason(null, dependency);

					const afterBuild = () => {
						if (currentProfile) {
							const afterBuilding = Date.now();
							currentProfile.building = afterBuilding - afterFactory;
						}

						if (addModuleResult.dependencies) {
                            //收集module依赖，添加到依赖项
							this.processModuleDependencies(module, err => {
								if (err) return callback(err);
								callback(null, module);
							});
						} else {
							return callback(null, module);
						}
					};

					if (addModuleResult.issuer) {
						if (currentProfile) {
							module.profile = currentProfile;
						}
					}

					if (addModuleResult.build) {
                        // 生成模块
						this.buildModule(module, false, null, null, err => {
							if (err) {
								this.semaphore.release();
								return errorAndCallback(err);
							}

							if (currentProfile) {
								const afterBuilding = Date.now();
								currentProfile.building = afterBuilding - afterFactory;
							}

							this.semaphore.release();
							afterBuild();
						});
					} else {
						this.semaphore.release();
						this.waitForBuildingFinished(module, afterBuild);
					}
				}
			);
		});
}
addEntry(context, entry, name, callback) {
		this.hooks.addEntry.call(entry, name);

		const slot = {
			name: name,
			// TODO webpack 5 remove `request`
			request: null,
			module: null
		};

		if (entry instanceof ModuleDependency) {
			slot.request = entry.request;
		}

		// TODO webpack 5: merge modules instead when multiple entry modules are supported
		const idx = this._preparedEntrypoints.findIndex(slot => slot.name === name);
		if (idx >= 0) {
			// Overwrite existing entrypoint
			this._preparedEntrypoints[idx] = slot;
		} else {
			this._preparedEntrypoints.push(slot);
		}
		this._addModuleChain(
			context,
			entry,
			module => {
				this.entries.push(module);
			},
			(err, module) => {
				// 入口文件编译完成
				if (err) {
					this.hooks.failedEntry.call(entry, name, err);
					return callback(err);
				}

				if (module) {
					slot.module = module;
				} else {
					const idx = this._preparedEntrypoints.indexOf(slot);
					if (idx >= 0) {
						this._preparedEntrypoints.splice(idx, 1);
					}
				}
				this.hooks.succeedEntry.call(entry, name, module);
				return callback(null, module);
			}
		);
	}
```

## Resolver
```js
const fs = require("fs");
const path=require('path')
const { CachedInputFileSystem, ResolverFactory } = require("enhanced-resolve");

// create a resolver
const myResolver = ResolverFactory.createResolver({
	// Typical usage will consume the `fs` + `CachedInputFileSystem`, which wraps Node.js `fs` to add caching.
	fileSystem: new CachedInputFileSystem(fs, 4000),
    alias: {
        Utilities: require.resolve('webpack')
    },
    extensions:['.js','.json'],
    aliasFields:['web'],
    descriptionFiles:['package.json'],
    enforceExtension:false,
    modules:['node_modules'],
    mainFields: ['web', 'module','main'],// 读取模块是根据描述文件加载
    mainFiles:['index'], // 默认读取目录时加载的文件
    resolveToContext:true,
	/* any other resoelver options here. Options/defaults can be seen below */
});

// resolve a file with the new resolver
const context = {
    issuer:""
};
const resolveContext = {

};
const lookupStartPath = path.resolve(__dirname);
const request = "./src/app";
myResolver.resolve(context, lookupStartPath, request, resolveContext, (
	err /*Error*/,
	filepath /*string*/
) => {
    // Do something with the path
    console.log(filepath)
});
```
## SingleEntryDependency
[addEntry](#addEntry)
```js
	/**
	 * An entry plugin which will handle
	 * creation of the SingleEntryDependency
	 *
	 * @param {string} context context path
	 * @param {string} entry entry path
	 * @param {string} name entry key name
	 */
	constructor(context, entry, name) {
		this.context = context;
		this.entry = entry;
		this.name = name;
	}

	/**
	 * @param {Compiler} compiler the compiler instance
	 * @returns {void}
	 */
	apply(compiler) {
		compiler.hooks.compilation.tap(
			"SingleEntryPlugin",
			(compilation, { normalModuleFactory }) => {
                // 添加依赖项处理工厂映射
				compilation.dependencyFactories.set(
					SingleEntryDependency,
					normalModuleFactory
				);
			}
		);

		compiler.hooks.make.tapAsync(
			"SingleEntryPlugin",
			(compilation, callback) => {
				const { entry, name, context } = this;
                // 创建一个实现ModuleDependency类的实例对象
                const dep = SingleEntryPlugin.createDependency(entry, name);
                // 把依赖对象添加到入口编译数组中
				compilation.addEntry(context, dep, name, callback);
			}
		);
	}

	/**
	 * @param {string} entry entry request
	 * @param {string} name entry name
	 * @returns {SingleEntryDependency} the dependency
	 */
	static createDependency(entry, name) {
		const dep = new SingleEntryDependency(entry);
		dep.loc = { name };
		return dep;
	}
```

## NormalModuleFactory

```js


```
