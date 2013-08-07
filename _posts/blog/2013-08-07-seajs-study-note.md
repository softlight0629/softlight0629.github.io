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
