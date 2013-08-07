---
layout: post
title: 像面向对象那样写js(nija)
description: 其实这种写法自己在工作中已经用到，在此想做个总结
category: blog
---

在最近的项目中， 开始用到一些面向对象的写法， 以及组件之间存在继承的关系， 所以， 就想写篇东西来讲讲如何去做到面向对象。

我们知道， 在js中的Function可以相当于我们所说的类的功能， 定义一个function，也就类似的可以说定义完了一个构造函数， 在构造函数中引用this就能开始创造一个对象。

现在来看看我们平时是怎么创建一个对象的吧

    function MyFunc(){};
    var obj = new MyFunc();

这样我们就创建好了一个实例， 其实这种代码可以改成这种等价的形式

    function MyFunc(){};
    var obj = {};
    MyFunc.call(obj);

js的内部就是这么做的。这就是一个简单的创建对象的方式


既然谈到了面向对象， 那么肯定就会想到要有继承的关系。而js并不是原生的就提供了面向对象的机制， 所以也不存在extend的方式， 不过， 它提供了一种原型链的方式，也就是当一个使用一个对象自身并不存在的属性或者方法的时候， 那么，js会向上去追寻这个对象的原型上的方法和属性一直追溯到Object对象的这个最顶层的原型.

那么， 一个原型是怎么去产生的？对象的原型是从哪里来的。一般来说一个对象的原型最初始的时候， 就拥有了Object的原型。如果，我们通过new 的方式去创建一个对象， 那么创建出来的对象就拥有了那个构造函数的原型

来看个例子

    function MyFunc() {};

    MyFunc.prototype.say = function() {
        alert('hi, prototype');
    };

    var obj = new MyFunc();

这样 obj对象就拥有MyFunc构造函数中的原型.而原型并不是只追溯到一层， 如果MyFunc中的prototype中也拥有原型的话， 那么这个就能产生原型链， js也就可以通过这个能力， 来实现继承.

###继承的例子:
 就拿最简单的例子来说明下这种原型继承吧

我们本身是做项目的， 那就拿项目经理做例子吧。

项目经理 是一个雇员， 雇员又是最原始的一个人， 所以， 项目经理继承雇员，雇员继承人

    function Person() {}

    Person.prototype.eat = function() {
        alert('人可以吃饭');
    }

    function Employee(){}

    Employee.prototype = new Person(); // 员工继承了人的能力

    Employee.prototype.work = function() {
        alert('员工可以工作');
    }

    function Manager() {}

    Manager.prototype = new Employee(); //项目经理继承了员工的能力

    Manager.prototype.report = function() {
        alert('项目经理可以报告成果');
    }

    var manager = new Manager();
    manager.eat();
    manager.work();
    manager.report();

这样， 就能够产生了一个继承的关系。 最简单的继承就是这样了。但是， 如果我们在项目中每个继承都需要这么做的话，那不是会麻烦死了。而且看起来一点都不美观， 根本就不适合拿来做开发项目中， 只是简简单单的玩玩倒是可以。所以我们就是要封装。

还有一个问题就是， 在面向对象中， 我们一般肯定需要能调用父类构造函数的能力， 类似于java中的super功能.可惜我们这里并没有体现， 如果需要， 我们只能显示的去调用父类构造函数， 类似这样 Person.call(this), 这样我们必须得知道，我们继承了哪个类， 那么有没有方式， 直接使用super的方式呢.


肯定是有的。领悟javascript里面说到一种甘露模型， 就能很好的封装了这个类的继承以及构造函数的调用
看下代码：

    function Class() {
        // 定义类的自身实例方法
        var aDefine = arguments[arguments.length - 1];

        if (!aDefine) {
            return;
        }

        // 父类的构造函数
        var aBase = arguments.length > 1?arguments[0]:object;

        // 在这里为什么要创建一个prototype_?
        // 以我的估计，主要是为了避免去调用父类的构造， 因为我们只需要父类的原型方法就好
        function prototype_(){}
        prototype_.prototype = aBase.prototype;
        var aPrototype = new prototype_();// 创建父类的原型对象

        for (var member in aDefine) {
            if (member != "actor") {
                aPrototype[member] = aDefine[member]; // 自定义的实例方法添加到原型对象
            }
        }

        var aType;
        if (!aDefine.actor) {
            aType = function() {
                this.base.apply(this, arguments);
            }
        } else {
            aType = aDefine.actor;
        }

        aType.prototype = aPrototype; // 原型对象赋值给一个构造方法
        aType.Base = aBase; // 保存当前类的父类
        aType.prototype.Type = aType;

        return aType;
    }

    function object(){}
    object.prototype.base = function() {
        var aBase = this.Type.Base;
        if (!aBase.Base) {
            aBase.apply(this,arguments);
        } else {
            this.base = makeBase(aBase);
            aBase.apply(this, arguments);
            delete this.base;
        }

        function makeBase(Type) {
            var aBase = Type.Base;
            if(!aBase.Base) return aBase;

            return function() {
                this.base = makeBase(aBase);
                aBase.apply(this, arguments);
            }
        }
    }

以上就是甘露模型， 可能初看起来时比较的复杂的，尤其是对于base这个方法，究竟是怎么去实现调用父类的构造的呢？
这里的base就是相当于super,只是名称改变了。而这个方法为什么会出现闭包呢？

我们知道，我们调用父类的构造方法的时候， 直接使用this.base()就可以， 而在这个时候， 一般继承结构少于2层的时候是没有问题的， 但一旦继承结构多于2层那么， 可能会出现循环调用。问题出在哪？

就在this这个作用域上。在java中我们知道， this其实指的就是当前的类的实例， 而js中的this并不是如此， 它是谁调用该方法this就是指向那个对象， 也就是说我们在this.base()刚好去调用父类构造函数的时候， 而父类中也出现了this.base(),那就麻烦了， 因为this并不是说是指向父类， 其实指向的依然是子类的实例， 而子类的实例的base调用的不就是当前这个父类么， 所以， 可能就是会进入循环的状态。

那么，怎么去解决啊？只要我们对this.base进行包装， 在调用this.base的时候,能够正确的去做到指向正确的父类，这样才能实现多层的继承关系。

在this.base的makeBase方法中 ， 借用Type.Base可以一直追溯到最顶层的那个父类， 然后通过aBase.apply的方式去调用父类的构造， 那么就能不会因为this的作用域的问题而陷入循环。这种apply方式， 也就是上面提到的Person.call的形式， 只不过，这里我们无需去知道具体的类的信息， 因为已经记录在了Type.Base里了

这就是甘露模型， 看下我们该怎么改造原先的那个例子

    var Person = Class({
        actor: function() {
            alert('调用了Person构造');
        },

        eat: function() {
            alert('人可以吃饭');
        }
    });

    var Employee = Class(Person, {

        actor: function() {
            this.base();
            alert('调用了Employee构造');
        },

        work: function() {
            alert('员工可以工作');
        }
    });

    var Manager = Class(Employee, {
        actor: function() {
            this.base();
            alert('调用了Manager构造')
        },

        report: function() {
            alert('项目经理可以做报告');
        }
    });

    var manager = new Manager();
    manager.eat();
    manager.work();
    manager.report();

这样对于开发项目来说就是清爽的多了， 也简洁了许多， 像是个面向对象的开发方式。

可是还是不够。我们仅仅只是让构造函数有了super的能力， 但是， 对于其他的方法呢， 并没有这样的能力啊。我们总不可以每个方法都使用这种闭包的方式去构造吧。 这里， 就要说说另外一种方式了

在screate of javascript nija中有种方式， 就似乎真的可以让js就像写java类那样的去开发了

    (function() {
        var initializing = false,
            superPattern = /xyz/.test(function() {xyz;})? /\b_super\b/ : /.*/;

        // 给Object对象添加subClass方法
        Object.subClass =  function(properties) {
            var _super = this.prototype;

            // 初始化父类
            initializing = true;
            var proto = new this();
            initializing = false;

            // 复制属性到原型
            for (var name in properties) {

                proto[name] = typeof properties[name] == "function" && typeof _super[name] == "function" && superPattern.test(properties[name]) ? 
                // 覆盖父类方法
                (function (name, fn) {
                    return function() {
                        var tmp = this._super;

                        this._super = _super[name];

                        var ret = fn.apply(this, arguments);

                        this._super = tmp;

                        return ret;
                    }
                })(name, properties[name]) : 
                properties[name];
            }

            // 创建伪类构造函数
            function Class {
                if (!initializing && this.init) {
                    this.init.apply(this, arguments);
                }
            }

            // 赋给原型
            Class.prototype = proto;

            // 指向构造函数
            Class.constructor = Class;

            // 使得该类拥有可继承的能力
            Class.subClass = arugments.callee;

            return Class;

        }
    })();

这里最主要的就是初始化和super方法这两部分。只要对这里理解了， 那么对于整个实现也就大致掌握了。

为了使得子类拥有父类的功能， 我们一开始的时候， 就创建了父类的实例，然后对于子类的属性进行合并

    initializing = true;
    var proto = new this();
    initializing = false;

initializing在这里的作用， 主要是为了能够对父类的初始化进行保护， 避免在构造函数调用的时候， 父类还处于实例化的阶段。

现在来看看_super的功能是如何实现的

    proto[name] = typeof properties[name] == "function" && typeof _super[name] == "function" && superPattern.test(properties[name]) ? 
        // 覆盖父类方法
        (function (name, fn) {
            return function() {
                var tmp = this._super;

                this._super = _super[name];

                var ret = fn.apply(this, arguments);

                this._super = tmp;

                return ret;
            }
        })(name, properties[name]) : 
        properties[name];

在这段代码中， 检测了三个东西

* 子类的属性是否是一个方法
* 父类是否也存在该属性并且也是一个方法
* 子类方法中是否引用了_super的调用

如果这三个东西都存在， 我们就不能简简单单的只是复制下属性就可以。
在这里我们注意到， 对于_super调用的检测， 是通过正则表达式去检测Funciton的序列化字符串来做到的。Function的这种能力可以说让我们在运行时有了改变代码的能力。是种比较强悍的能力吧。有些类库基于这种能力，写出很多不错的功能。比如， seajs里面的require引用， 也是基于这种检测的方式， 来搜集某一模块的依赖。

现在我们来看看这段代码做了什么事情

    (function (name, fn) {
        return function() {
            var tmp = this._super;

            this._super = _super[name];

            var ret = fn.apply(this, arguments);

            this._super = tmp;

            return ret;
        }
    })(name, properties[name])

这段通过自执行的代码包装了实例的方法， 然后里面返回了一个新的方法， 通过改变this._super的指向父类的原型方法， 就可以调用到了父类的方法， 最后我们调用了子类的实例方法， 这样就能够在子类的方法中能够调用到父类的方法了。

对于改造后的这个方法，我们看下如何改造先前的例子

    var Person = Object.subClass({
        init: function() {
            alert('调用Person构造');
        },

        eat: function() {
            alert('人吃饭');
        }
    })

    var Employee = Person.subClass({
        init: function() {
            this._super();
            alert('调用Employee构造');
        },

        eat: function() {
            this._super();
            alert('员工吃套餐');
        }
    })

    var Manager = Employee.subClass({
        init: function() {
            this._super();
            alert('调用Manager构造');
        },

        eat: function() {
            alert('项目经理吃大餐');
        }
    })

    var manager = new Manager();
    manager.eat();

这样，是不是跟面向对象的语言完全一样呢？当然对于方法的重载， js也没有那样的能力， 这个也是需要我们自己去实现， 不过js并没有强制规定说要对方法传递参数个数做强制的限制， 所以我们可以灵活的在方法内部去判断传递过来的参数的个数， 来模拟实现重载的功能。这里就不再多讲了。

这样就可以在js中写Java一样， 可以做到灵活， 继承， 多变， 可以很好的做到复用， 也能运用许多的设计方法， 设计模式， 使得前端的开发， 变得更加有趣和有效率。

