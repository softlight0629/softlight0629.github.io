---
layout: post
title: seajs学习笔记
description: 虽然了解模块化开发， 但对于内部的实现原理尚不明了，边看源码边记录
category: blog
---

这几天研究Bui的源码， 发现里面已经集成了seajs的Loader功能， 我一直想了解这块的东西， 以前也接触过seajs的源码， 但没有深究， 所以， 在此做点学习记录， 好以后可以回顾

##seajs的模块划分

seajs在内部把功能划分了几大模块， 直接上代码可以看看

    /**
     * 顶层命名空间
     */
    this.seajs = { _seajs: this.seajs }

    /**
     * seajs 的版本
     */
    seajs.version = '1.3.0'

    /**
     * 私有变量， 仅供内部使用
     * 内部方法基本挂在_util上
     */
    seajs._util = {}

    /**
     * 私有变量，仅供内部使用
     * 存储配置信息
     */
     seajs._config = {
        /**
         * debug模式. js压缩时候自动关闭
         */
        debug: '%DEBUG%',

        /**
         * 需要预先加载的模块
         */
        preload: []
     }

     /**
      * 简化版的语法糖
      */
     ;(function(util) {

     })(seajs._util)

     /**
      * 日志方法
      */
    ;(function(util) {

     })(seajs._util)

     /**
      * 路径解析
      */
     ;(function(util,config, global) {

     })(seajs._util,config, global)

     /**
      *  加载js和css文件
      */
     ;(function(util, config) {

     })(seajs._util, seajs._config, this)

     /**
      * 依赖解析器
      */
     ;(function(util) {

     })(seajs._util)

     /**
      * 加载器
      */
     ;(function(util) {

     })(seajs._util)

     /**
      * seajs配置项
      */
     ;(function(util) {

     })(seajs._util)


     /**
      * 准备引导程序
      */
     ;(function(util) {

     })(seajs._util)

     /**
      * 启动入口
      */
     ;(function(util) {

     })(seajs._util)


基于学习的目的， 也因为篇幅有限， 然后seajs又划分了这么多模块， 那就针对每个模块， 做点小分析， 或者有哪些东西可以借鉴的吧

###语法糖

语法糖无非就是一些工具方法可以帮助我们处理一些常用的功能， 一些流行的类库基本都有， 而且提供的大致都是相似的。本来没有什么好说的， 但是， 有一点这里做的比较的好， 就是在弄每个语法糖之前都会判断当前的上下文是否已经拥有了类似的语法糖，如果有就用已经存在的， 否则就去创建。想做到这点再简单不过， 就判断该方法是否已经存在就行了。看看代码

    util.indexOf = AP.indexOf ?
      function(arr, item) {
        return arr.indexOf(item)
      } :
      function(arr, item) {
        for (var i = 0; i < arr.length; i++) {
          if (arr[i] === item) {
            return i
          }
        }
        return -1
      }

###日志方法

    /**
    * 对浏览器的日志方法进行一层包装
    * 主要是对安全方面做了一层检测
    */
    util.log = function() {
        if (typeof console === 'undefined') return

        var args = Array.prototype.slice.call(arguments)

        var type = 'log'
        var last = args[args.length - 1]
        console[last] && (type = args.pop())

        // 只在Debug模式下才进行日志的输出
        if (type === 'log' && !seajs.debug) return

        if (console[type].apply) {
          console[type].apply(console, args)
          return
        }

        // See issue#349
        var length = args.length
        if (length === 1) {
          console[type](args[0])
        }
        else if (length === 2) {
          console[type](args[0], args[1])
        }
        else if (length === 3) {
          console[type](args[0], args[1], args[2])
        }
        else {
          console[type](args.join(' '))
        }
    }

###路径解析


###加载js和css文件

    /** 
     * Utilities for fetching js and css files. 
     * 获取js和css文件的工具集 
     */  
    ;(function(util, config) {  
      
      var doc = document  
      var head = doc.head ||  
          doc.getElementsByTagName('head')[0] ||  
          doc.documentElement  
      
      var baseElement = head.getElementsByTagName('base')[0]  
      
      var IS_CSS_RE = /\.css(?:\?|$)/i  
      var READY_STATE_RE = /loaded|complete|undefined/  
      
      var currentlyAddingScript  
      var interactiveScript  
      
      
      //加载资源文件文件  
      
      util.fetch = function(url, callback, charset) {  
      
        //获取的文件是不是css  
        var isCSS = IS_CSS_RE.test(url)  
      
        //如果是css创建节点 link  否则 则创建script节点  
        var node = document.createElement(isCSS ? 'link' : 'script')  
      
        //如果存在charset 如果charset不是function类型，那就直接对节点设置charset ，如果是function如下例：  
        /* 
        seajs.config({ 
          charset: function(url) { 
          // xxx 目录下的文件用 gbk 编码加载 
            if (url.indexOf('http://example.com/js/xxx') === 0) { 
              return 'gbk'; 
            } 
          // 其他文件用 utf-8 编码 
            return 'utf-8'; 
          } 
     
        }); 
        */  
        if (charset) {  
          var cs = util.isFunction(charset) ? charset(url) : charset  
          cs && (node.charset = cs)  
        }  
      
        //assets执行完毕后执行callback ，如果自定义callback为空，则赋予noop 为空函数  
        assetOnload(node, callback || noop)  
      
        //如果是样式 ……  如果是 脚本 …… async 详见：https://github.com/seajs/seajs/issues/287  
        if (isCSS) {  
          node.rel = 'stylesheet'  
          node.href = url  
        }  
        else {  
          node.async = 'async'  
          node.src = url  
        }  
      
        // 在IE6-9的某些情况下， 在执行完insertBefore方法后
        // js脚本会立即执行， 所以用`currentlyAddingScript`
        // 来引用当前的节点, 为了在'define'方法中能获取到  
        currentlyAddingScript = node  
      
        // ref: #185 & http://dev.jquery.com/ticket/2709   
        // 关于base 标签 http://www.w3schools.com/tags/tag_base.asp  
      
        baseElement ?  
            head.insertBefore(node, baseElement) :  
            head.appendChild(node)  
      
        currentlyAddingScript = null  
      }  
      
      //资源文件加载完毕后执行回调callback  
      function assetOnload(node, callback) {  
        if (node.nodeName === 'SCRIPT') {  
          scriptOnload(node, callback)  
        } else {  
          styleOnload(node, callback)  
        }  
      }  
      
      //资源文件加载完执行回调不是所有浏览器都支持一种形式，存在兼容性问题  
      //http://www.fantxi.com/blog/archives/load-css-js-callback/ 这篇文章非常不错  
      
      //加载脚本完毕后执行回调  
      function scriptOnload(node, callback) {  
      
        // onload为IE6-9/OP下创建CSS的时候，或IE9/OP/FF/Webkit下创建JS的时候    
        // onreadystatechange为IE6-9/OP下创建CSS或JS的时候  
      
        node.onload = node.onerror = node.onreadystatechange = function() {  
      
          //正则匹配node的状态  
          //readyState == "loaded" 为IE/OP下创建JS的时候  
          //readyState == "complete" 为IE下创建CSS的时候 -》在js中做这个正则判断略显多余  
          //readyState == "undefined" 为除此之外浏览器  
          if (READY_STATE_RE.test(node.readyState)) {  
      
            // Ensure only run once and handle memory leak in IE  
            // 配合 node = undefined 使用 主要用来确保其只被执行一次 并 处理了IE 可能会导致的内存泄露  
            node.onload = node.onerror = node.onreadystatechange = null  
      
            // Remove the script to reduce memory leak  
            // 在存在父节点并出于非debug模式下移除node节点  
            if (node.parentNode && !config.debug) {  
              head.removeChild(node)  
            }  
      
            // Dereference the node  
            // 废弃节点，这个做法其实有点巧妙，对于某些浏览器可能同时支持onload或者onreadystatechange的情况，只要支持其中一种并执行完一次之后，把node释放，巧妙实现了可能会触发多次回调的情况  
            node = undefined  
      
            //执行回调  
            callback()  
          }  
        }  
      
      }  
      
      //加载样式完毕后执行回调  
      function styleOnload(node, callback) {  
      
        // for Old WebKit and Old Firefox  
        // iOS 5.1.1 还属于old --！ 但是 iOS6中 536.13  
        // 这里用户采用了代理可能会造成一点的勿扰，可能代理中他是一个oldwebkit浏览器 但是实质却不是  
        if (isOldWebKit || isOldFirefox) {  
          util.log('Start poll to fetch css')  
      
          setTimeout(function() {  
            poll(node, callback)  
          }, 1) // Begin after node insertion   
          // 延迟执行 poll 方法，确保node节点已被插入  
        }  
        else {  
          node.onload = node.onerror = function() {  
            node.onload = node.onerror = null  
            node = undefined  
            callback()  
          }  
        }  
      
      }  
      
      function poll(node, callback) {  
        var isLoaded  
      
        // for WebKit < 536  
        // 如果webkit内核版本低于536则通过判断node节点时候含属性sheet  
        if (isOldWebKit) {  
          if (node['sheet']) {  
            isLoaded = true  
          }  
        }  
        // for Firefox < 9.0  
        else if (node['sheet']) {  
          try {  
            //如果存在cssRules属性  
            if (node['sheet'].cssRules) {  
              isLoaded = true  
            }  
          } catch (ex) {  
            // The value of `ex.name` is changed from  
            // 'NS_ERROR_DOM_SECURITY_ERR' to 'SecurityError' since Firefox 13.0  
            // But Firefox is less than 9.0 in here, So it is ok to just rely on  
            // 'NS_ERROR_DOM_SECURITY_ERR'  
      
            // 在Firefox13.0开始把'NS_ERROR_DOM_SECURITY_ERR'改成了'SecurityError'  
            // 但是这边处理是小于等于firefox9.0的所以在异常处理上还是依赖与'NS_ERROR_DOM_SECURITY_ERR'  
            if (ex.name === 'NS_ERROR_DOM_SECURITY_ERR') {  
              isLoaded = true  
            }  
          }  
        }  
      
        setTimeout(function() {  
          if (isLoaded) {  
            // Place callback in here due to giving time for style rendering.  
            callback()  
          } else {  
            poll(node, callback)  
          }  
        }, 1)  
      }  
      
      function noop() {  
      }  
      
      
      //用来获取当前插入script  
      util.getCurrentScript = function() {  
        // 如果已经获取到当前插入的script节点，则直接返回  
        if (currentlyAddingScript) {  
          return currentlyAddingScript  
        }  
      
        // For IE6-9 browsers, the script onload event may not fire right  
        // after the the script is evaluated. Kris Zyp found that it  
        // could query the script nodes and the one that is in "interactive"  
        // mode indicates the current script.  
        // Ref: http://goo.gl/JHfFW  
        // 在IE6-9浏览器中，script的onload事件有时候并不能在script加载后触发  
        // 需要遍历script的节点才能知道，当前脚本状态为 'interactive'  
        if (interactiveScript &&  
            interactiveScript.readyState === 'interactive') {  
          return interactiveScript  
        }  
      
        var scripts = head.getElementsByTagName('script')  
      
        for (var i = 0; i < scripts.length; i++) {  
          var script = scripts[i]  
          if (script.readyState === 'interactive') {  
            interactiveScript = script  
            return script  
          }  
        }  
      }  
      
      //获取script的绝对路径  
      util.getScriptAbsoluteSrc = function(node) {  
        return node.hasAttribute ? // non-IE6/7  
            node.src :  
            // see http://msdn.microsoft.com/en-us/library/ms536429(VS.85).aspx  
            node.getAttribute('src', 4)  
      }  
      
      
      // 创建样式节点  
      util.importStyle = function(cssText, id) {  
        // Don't add multi times   
        //一个id不要添加多次 如果页面中已经存在该id则直接返回  
        if (id && doc.getElementById(id)) return  
      
        //创建style标签，并指定标签id，并插入head，ie和其他标准浏览器插入css样式存在兼容性问题，具体如下：  
        var element = doc.createElement('style')  
        id && (element.id = id)  
      
        // Adds to DOM first to avoid the css hack invalid  
        head.appendChild(element)  
      
        // IE  
        if (element.styleSheet) {  
          element.styleSheet.cssText = cssText  
        }  
        // W3C  
        else {  
          element.appendChild(doc.createTextNode(cssText))  
        }  
      }  
      
      //获取 UA 信息  
      var UA = navigator.userAgent  
      
      // `onload` event is supported in WebKit since 535.23  
      // Ref:  
      //  - https://bugs.webkit.org/show_activity.cgi?id=38995  
      // css onload 事件的支持 从webkit 内核版本 535.23 开始  
      var isOldWebKit = Number(UA.replace(/.*AppleWebKit\/(\d+)\..*/, '$1')) < 536  
      
      // `onload/onerror` event is supported since Firefox 9.0  
      // onload/onerror 这个事件是从firefox9.0开始支持的，在判断中首先判断UA是否是Firefox 并且 在存在onload  
      // Ref:  
      //  - https://bugzilla.mozilla.org/show_bug.cgi?id=185236  
      //  - https://developer.mozilla.org/en/HTML/Element/link#Stylesheet_load_events  
      var isOldFirefox = UA.indexOf('Firefox') > 0 &&  
          !('onload' in document.createElement('link'))  
      
      
      /** 
       * References: 
       *  - http://unixpapa.com/js/dyna.html 
       *  - ../test/research/load-js-css/test.html 
       *  - ../test/issues/load-css/test.html 
       *  - http://www.blaze.io/technical/ies-premature-execution-problem/ 
       */  
      
    })(seajs._util, seajs._config, this)  

  


###解析模块依赖

    /** 
     * The parser for dependencies 
     * 解析模块依赖 
     */  
    ;(function(util) {  
      
      var REQUIRE_RE = /(?:^|[^.$])\brequire\s*\(\s*(["'])([^"'\s\)]+)\1\s*\)/g  
      
      
      util.parseDependencies = function(code) {  
        // Parse these `requires`:  
        //   var a = require('a');  
        //   someMethod(require('b'));  
        //   require('c');  
        //   ...  
        // Doesn't parse:  
        //   someInstance.require(...);  
        // 解析含有require字段的依赖 ， 但是不解析实例化之后的实例的依赖  
        var ret = [], match  
      
        code = removeComments(code)  
        //从头开始匹配 lastIndex的值会随着匹配过程的进行而修改  
        REQUIRE_RE.lastIndex = 0  
      
        // var asd = require('a'); 经过正则匹配后结果为 ["require('asd')","'" ,"asd"]  
        while ((match = REQUIRE_RE.exec(code))) {  
          if (match[2]) {  
            ret.push(match[2])  
          }  
        }  
      
        //返回数组去重值  
        return util.unique(ret)  
      }  
      
      //删除注释 , 其实就是避免在正则匹配的时候，注释内还含有require信息，导致加载不必要的模块  
      // See: research/remove-comments-safely  
      function removeComments(code) {  
        return code  
            .replace(/^\s*\/\*[\s\S]*?\*\/\s*$/mg, '') // block comments  
            .replace(/^\s*\/\/.*$/mg, '') // line comments  
      }  
      
    })(seajs._util)  
