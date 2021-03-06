## When input webpack in command line, what happened?

First of all, we need to check webpack command entry in package.json, the field of "bin" is "./bin/webpack.js".
This file will check which cli commander is installed,
if we are using `webpack-cli`, the "webpack-cli/lib/cli.js" will be operated. Its code can be simplified to:

```JavaScript
const webpack = require("webpack"); //v4.30.0
let compiler = webpack(options);    //options is what we exports in webpack.config.js
compiler.run((err, stats) => {
    compilerCallback(err, stats);   //just as its name implies， compilerCallback will run after compile finished.
});
```

Next, in `webpack/lib/webpack.js`, we can find the main webpack function which can be simplified as below:

```JavaScript
const webpack = (options, callback) => {
    options = new WebpackOptionsDefaulter().process(options);   //initialize options
    let compiler = new Compiler(options.context);   //options.context is the local path
    compiler.options = options;
    new NodeEnvironmentPlugin().apply(compiler);    //compiler gets node environment arguments
    
    if (options.plugins && Array.isArray(options.plugins)) {
        for (const plugin of options.plugins) {
            if (typeof plugin === "function") {
                plugin.call(compiler, compiler);
            } else {
                plugin.apply(compiler); //if plugin is an object, it must implement apply function
            }
        }
    }
    compiler.hooks.environment.call();
    compiler.hooks.afterEnvironment.call();
    compiler.options = new WebpackOptionsApply().process(options, compiler);    //return processed options
    return compiler;
}
```

`compiler` object gets all contents about webpack functions, including node environment, the client's options and the local path.

## Compiler Hooks and Complication Hooks

In `webpack/lib/Compiler.js`'s `run(callback)`, which is the function which compiling really start from:

```JavaScript
//......
this.hooks.beforeRun.callAsync(this, err => {
    if (err) return finalCallback(err);

    this.hooks.run.callAsync(this, err => {
        if (err) return finalCallback(err);

        this.readRecords(err => {
            if (err) return finalCallback(err);

            this.compile(onCompiled);   //start compile, will create new compilation by new Compilation(this)
        });
    });
});
```

Both compiler and compilation extends from [tapable](https://github.com/webpack/tapable), which defines all hooks' classes, which can be used to create hooks for plugins. From my understanding, The `Compiler`
is `webpack` itself, `compilation` represents this time's compiling. So when developing `webpack` plugins, `compilation` is the object that developers really operate.

In their `this.hooks`, we can all hooks can be used in plugins. There is an official hook list: [Compiler Hooks](https://webpack.js.org/api/compiler-hooks/), [Compilation Hooks](https://webpack.js.org/api/compilation-hooks/).

## How to develop a webpack plugin

Base on `uglifyjs-webpack-plugin`, see how it works. In the definition of `webpack`, we can see it use `plugin.apply(compiler);`. So firstly, check the apply function, its core code uses compiler is:

```JavaScript
compiler.hooks.compilation.tap(plugin, compilation => {
      if (this.options.sourceMap) {
        compilation.hooks.buildModule.tap(plugin, buildModuleFn);
      }

      const {
        mainTemplate,
        chunkTemplate
      } = compilation; // Regenerate `contenthash` for minified assets

      for (const template of [mainTemplate, chunkTemplate]) {
        template.hooks.hashForChunk.tap(plugin, hash => {
          const data = (0, _serializeJavascript.default)({
            uglifyjs: _package.default.version,
            uglifyjsOptions: this.options.uglifyOptions
          });
          hash.update('UglifyJsPlugin');
          hash.update(data);
        });
      }
      compilation.hooks.optimizeChunkAssets.tapAsync(plugin, optimizeFn.bind(this, compilation));
});
```

So basically, it is to use hook functions to get chunks object and process it. 

According to it, there is a simple example to print all modules used:

```JavaScript
class CustomClass{
    apply(compiler){
        let plugin = {name:this.constructor.name};
        compiler.hooks.compilation.tap(plugin, compilation => {
            compilation.hooks.afterOptimizeChunkAssets.tap(plugin, chunks => {
                chunks.forEach(chunk=>{
                    chunk.getModules().forEach(module=>{
                        console.log(module.id);
                    });
                })
            });
        });
    }
};
```

Base on it, I developed a tiny tool for checking, updating and combining NPM packgaes, [webpack-package-manager](https://github.com/Bert0324/webpack-package-manager). If you have time, it wwould be amazing to give me a star!

## Loader

Actually, as I think, in plugin people can do more things than in loader, which targets to transform loaded files. Loader can get files' content as string, a simple loader plugin example is like below:

```JavaScript
//webpack.config.js
    module: {
        rules: [{
            test:/\.js?$/,
            exclude: /(node_modules)/,
            use: [{
                loader: 'babel-loader', //loaders will be operated in inverted order, first is loader.js, next is babel-loader
            },{
                loader: require('path').resolve('loader.js'),
                options:{
                    value:1
                }
            }]
        }]
    }
//loader.js          //transform js code to es5
module.exports = function (source) {    //source is string
    let options = require('loader-utils').getOptions(this); //get options
    let result = require('@babel/core').transformSync(source, {
        plugins:[],
        presets:["@babel/preset-env"]
    });
    return result.code;
};
```

There is a nice image that can show webpack whole life cycle from [source](http://taobaofed.org/blog/2016/09/09/webpack-flow/):

<img src='../assets/webpack_lifecycle.jpeg' width="1200">

This is just a quick and sketchy looking at webpack. If you want to right a wrong or have some suggestions about it, welcome to contact me.




