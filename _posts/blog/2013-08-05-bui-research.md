---
layout: post
title: 学习研究Bui
description: 前几天看到一个类似Kissy的框架，基于学习的目的，做点小记录
category: blog
---

##目录结构与打包流程的学习

###Bui的文件结构
* assets: css文件，基于bootstrap的css样式， 可以在此基础上编译出新的版本
* build: js和css文件打包好的目录
* src: js的源文件
* test: 单元测试， 所有空间的单元测试都在内部
* tools: 文件打包，以及生成文件的工具
* docs: 源文件中未提供， 可以自己执行tools/jsduck/run.bat文件

###打包
* 合并js， 压缩js
* 编译less生成css, 压缩css
* 复制文件， 将所有js合并成一个bui.js
* 执行build.bat文件

###生成文档
* 使用jsduck进行编译文档, tools/jsduc/run.bat
* 配置文件在tools/jsduck/config.json


###来看看代码

    var UA = $.UA || (function(){
        var browser = $.browser,
            versionNumber = numberify(browser.version),
            /**
             * 浏览器版本检测
             * @class BUI.UA
                     * @singleton
             */
            ua = 
            {
                /**
                 * ie 版本
                 * @type {Number}
                 */
                ie : browser.msie && versionNumber,

                /**
                 * webkit 版本
                 * @type {Number}
                 */
                webkit : browser.webkit && versionNumber,
                /**
                 * opera 版本
                 * @type {Number}
                 */
                opera : browser.opera && versionNumber,
                /**
                 * mozilla 火狐版本
                 * @type {Number}
                 */
                mozilla : browser.mozilla && versionNumber
            };
        return ua;
    })();