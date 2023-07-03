---
title: 使用vite-plugin-monkey优化油猴脚本
date: 2023-07-03 15:09:42
tags:
- coding
---
之前写过的脚本全部混杂在一个文件里，非常原始，不利于维护，所以想着用vite来优化一下。
<!--more-->
在v站看过有v友写的vite插件，用于编写油猴脚本的，一直想拿来用用，今天成功把`阿里云盘树状目录`跑起来了。

## 创建项目

```bash
pnpm create monkey
```
我选择`vanilla-ts`模板，因为脚本很简单。

## 安装依赖

用到的模块：
- jQuery 管理dom
- jQuery.fancytree 展示列表

重点：
这俩模块需要处理一下，因为咱用到了vite，配置vite进行正确地编译。

关于常见的模块格式
- CommonJS（通用模块规范）是一种用于服务器端JavaScript的模块系统。它的目标是使JavaScript能够在不同的环境中共享和重用代码。在CommonJS中，模块使用require函数导入，使用module.exports或exports对象导出。
- UMD（通用模块定义）是一种兼容多种模块系统的模块格式。它可以在浏览器环境和服务器环境中使用。UMD模块首先检测是否支持AMD模块，如果不支持，则检测是否支持CommonJS模块，最后，如果都不支持，则将模块作为全局变量导出。
- AMD（异步模块定义）是一种用于浏览器端JavaScript的模块系统。它的设计目标是在浏览器中异步加载模块，以提高性能。AMD模块使用define函数定义模块，并使用require函数异步加载依赖模块。
- IIFE（立即调用函数表达式）是一种常见的JavaScript模块模式，它使用函数作用域来封装代码，并立即执行该函数。这种模式可以用于创建私有作用域，避免变量冲突，并将部分代码暴露给全局作用域。
- ESM（ECMAScript模块）是一种现代的JavaScript模块系统，它是ECMAScript标准的一部分，从ES6（ES2015）开始引入。ESM模块使用import语句导入模块，使用export关键字导出模块。ESM模块可以在浏览器和服务器环境中使用，并且可以进行静态分析和优化。

下面分别解释vite在dev模式和prod模式中的区别：
```
在 Vite 中，开发模式（dev）和生产模式（prod）之间的区别主要体现在打包工具和模块规范上。

开发模式（dev）：
打包工具：在开发模式下，Vite 使用了一种称为「原生 ESM」的打包方式。它利用浏览器原生的 ES Modules 支持，不进行传统的打包和捆绑操作。这意味着每个模块都是作为一个独立的文件进行加载，而不是将所有模块合并为一个或多个捆绑文件。
模块规范：在开发模式下，Vite 支持 ES Modules（ESM）规范，你可以使用 import 和 export 来导入和导出模块。此外，Vite 也支持 CommonJS 规范，因此你可以使用 require 和 module.exports。

生产模式（prod）：
打包工具：在生产模式下，Vite 使用 Rollup 作为打包工具。Rollup 是一个优化的 JavaScript 模块打包器，它会将应用程序的模块捆绑成更小、更高效的文件。
模块规范：在生产模式下，Vite 会将所有模块转换为符合 ES Modules（ESM）规范的代码，并且会进行代码压缩、混淆和优化，以减小文件大小并提高加载性能。这意味着你只能使用 import 和 export 来导入和导出模块，而不支持 CommonJS 规范。
总结起来，开发模式下的 Vite 使用原生 ESM 打包方式和支持多种模块规范，而生产模式下的 Vite 使用 Rollup 打包工具，并且只支持 ES Modules（ESM）规范。这种区别使得开发者可以在开发过程中享受到更快的冷启动和模块热替换等优势，并在生产环境中获得更高的性能和优化的代码。
```
- 在开发模式时，通过pnpm安装jquery和jquery.fancytree，两个模块都是支持esm导入的
- 在生产模式时，将jquery和jquery.fancytree都定义在vite-plugin-monkey的`externalGlobals`中，在此定义后，意味着该模块不经过处理而直接放到油猴脚本中的`@require`中，作为cdn依赖

```ts
// https://vitejs.dev/config/
export default defineConfig({
  plugins: [
    // 将 jQuery 注入到每个模块中
    inject({
      $: "jquery",  // 这里会自动载入 node_modules 中的 jquery
      jQuery: "jquery",
      'jquery.fancytree': 'jquery.fancytree',
    }),

    monkey({
      entry: 'src/main.ts',
      userscript: {
        icon: 'https://vitejs.dev/logo.svg',
        namespace: 'npm/vite-plugin-monkey',
        match: ['https://www.aliyundrive.com/s/*'],
      },
      build: {
        // 替换外部依赖为头部@require
        externalGlobals:{
          jquery: $.concat(
            await util.fn2dataUrl(() => {
              // @ts-ignore
              window.jQuery = window.jquery = $;
            })
          ),
          'jquery.fancytree': fancytree,
        },
        externalResource: {
          'jquery.fancytree/dist/skin-lion/ui.fancytree.css': cdn.bootcdn()
        }
      }
    }),
  ],
  build: {
    commonjsOptions: {
      // 支持 commonjs 的 require
      transformMixedEsModules: true
    }
  }
});
```
## 模块中css引用相对路径资源问题
其中遇到了些问题，需要先了解dev模式下，`vite-plugin-monkey`是如何做到热更新的？
[https://github.com/lisonge/vite-plugin-monkey/discussions/72](关于vite-plugin-monkey的启动实时更新的模式是如何实现的讨论)

作者说到：脚本dev模式下生成的代码，实际更新只在脚本服务器中更新，插件页面上用到的环境是脚本服务器的
```js
;(({ entrySrc = `` }) => {
  window.GM;
  document.__monkeyWindow = window;
  console.log(`[vite-plugin-monkey] mount monkeyWindow to document`);
  const entryScript = document.createElement("script");
  entryScript.type = "module";
  entryScript.src = entrySrc;
  document.head.insertBefore(entryScript, document.head.firstChild);
  console.log(`[vite-plugin-monkey] mount entry module to document.head`);
})(...[
  {
    "entrySrc": "http://127.0.0.1:5173/__vite-plugin-monkey.entry.js"
  }
]);
```
同时这也造成一个问题，在模块中引入的图片路径是相对`/`目录时，相对的是浏览器中的根目录，而不是脚本服务器的。
从而导致我的图片加载失败。
![image-20230703153946523](使用vite-plugin-monkey优化油猴脚本\image-20230703153946523.png)

解决方案就是写一个链接替换插件，将原来指向根目录的链接替换为脚本服务器的链接。
```ts
function modifyCssUrlsPlugin() {
  let rootUrl = '/'
  return {
    name: 'modify-css-urls',
    configureServer(_server) {
      rootUrl = 'http://' + server.config.server.host + ':' + server.config.server.port;
    },
    transform(code, id) {
      if (id.endsWith('.css')) {
        // 读取 CSS 文件内容
        const cssContent = code;
        // 修改 CSS 文件中的 URL 相对路径
        const modifiedCssContent = cssContent.replace(/url\((.*?)\)/g, (match, sourceUrl) => {
          // 处理 URL 相对路径
          const modifiedUrl = resolveRelativeUrl(rootUrl, sourceUrl);
          return `url(${modifiedUrl})`;
        });

        // 返回修改后的 CSS 代码
        return {
          code: modifiedCssContent,
          map: null, // 如果需要生成 Source Map，请配置正确的 Source Map
        };
      }
    },
  };
}
// vite.config.ts
export default defineConfig({
  plugins: [
    modifyCssUrlsPlugin(),
    // ...
  ]
})
```

![image-20230703154420746](使用vite-plugin-monkey优化油猴脚本\image-20230703154420746.png)

问题成功解决！
