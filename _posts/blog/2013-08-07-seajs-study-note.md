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

    ;(function(util, config) {

      var doc = document
      var head = doc.head ||
          doc.getElementsByTagName('head')[0] ||
          doc.documentElement

      var baseElement = head.getElementsByTagName('base')[0]

      // 判断是否是css文件
      var IS_CSS_RE = /\.css(?:\?|$)/i
      var READY_STATE_RE = /loaded|complete|undefined/

      var currentlyAddingScript
      var interactiveScript

      // 加载资源文件
      util.fetch = function(url, callback, charset) {
        var isCSS = IS_CSS_RE.test(url)
        var node = document.createElement(isCSS ? 'link' : 'script')

        if (charset) {
          var cs = util.isFunction(charset) ? charset(url) : charset
          cs && (node.charset = cs)
        }

        assetOnload(node, callback || noop)

        if (isCSS) {
          node.rel = 'stylesheet'
          node.href = url
        } else {
          node.async = 'async'
          node.src = url
        }

        // For some cache cases in IE 6-9, the script executes IMMEDIATELY after
        // the end of the insertBefore execution, so use `currentlyAddingScript`
        // to hold current node, for deriving url in `define`.
        currentlyAddingScript = node

        // ref: #185 & http://dev.jquery.com/ticket/2709
        baseElement ?
            head.insertBefore(node, baseElement) :
            head.appendChild(node)

        currentlyAddingScript = null
      }

      function assetOnload(node, callback) {
        if (node.nodeName === 'SCRIPT') {
          scriptOnload(node, callback)
        } else {
          styleOnload(node, callback)
        }
      }

      function scriptOnload(node, callback) {

        node.onload = node.onerror = node.onreadystatechange = function() {
          if (READY_STATE_RE.test(node.readyState)) {

            // Ensure only run once and handle memory leak in IE
            node.onload = node.onerror = node.onreadystatechange = null

            // Remove the script to reduce memory leak
            if (node.parentNode && !config.debug) {
              head.removeChild(node)
            }

            // Dereference the node
            node = undefined

            callback()
          }
        }

      }

      function styleOnload(node, callback) {

        // for Old WebKit and Old Firefox
        if (isOldWebKit || isOldFirefox) {
          util.log('Start poll to fetch css')

          setTimeout(function() {
            poll(node, callback)
          }, 1) // Begin after node insertion
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
        if (isOldWebKit) {
          if (node['sheet']) {
            isLoaded = true
          }
        }
        // for Firefox < 9.0
        else if (node['sheet']) {
          try {
            if (node['sheet'].cssRules) {
              isLoaded = true
            }
          } catch (ex) {
            // The value of `ex.name` is changed from
            // 'NS_ERROR_DOM_SECURITY_ERR' to 'SecurityError' since Firefox 13.0
            // But Firefox is less than 9.0 in here, So it is ok to just rely on
            // 'NS_ERROR_DOM_SECURITY_ERR'
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


      util.getCurrentScript = function() {
        if (currentlyAddingScript) {
          return currentlyAddingScript
        }

        // For IE6-9 browsers, the script onload event may not fire right
        // after the the script is evaluated. Kris Zyp found that it
        // could query the script nodes and the one that is in "interactive"
        // mode indicates the current script.
        // Ref: http://goo.gl/JHfFW
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

      util.getScriptAbsoluteSrc = function(node) {
        return node.hasAttribute ? // non-IE6/7
            node.src :
            // see http://msdn.microsoft.com/en-us/library/ms536429(VS.85).aspx
            node.getAttribute('src', 4)
      }


      util.importStyle = function(cssText, id) {
        // Don't add multi times
        if (id && doc.getElementById(id)) return

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


      var UA = navigator.userAgent

      // `onload` event is supported in WebKit since 535.23
      // Ref:
      //  - https://bugs.webkit.org/show_activity.cgi?id=38995
      var isOldWebKit = Number(UA.replace(/.*AppleWebKit\/(\d+)\..*/, '$1')) < 536

      // `onload/onerror` event is supported since Firefox 9.0
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





