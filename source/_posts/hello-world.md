title: angularjs数据双向绑定
tags:
  - angularjs
categories: []
author: leeyvon
date: 2018-01-11 16:48:00
---
<img src="https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1516010191359&di=2004a483910ffdb42e5895397eb8d5c0&imgtype=0&src=http%3A%2F%2Fyyyweb.qiniudn.com%2Fuploads%2F2014%2F10%2F1398932756598-angularjs.jpg" />

angularjs作为一个mvvm框架，具有数据的**双向绑定**功能，所谓的**双向绑定**就是如果视图改变了某个值，数据模型会通过**脏检查**观察到这个变化，而如果数据模型改变了某个值，视图也会依据变化重新渲染。

<!--more-->

#### $digest():脏值循环检查
例如：
```bash
<div ng-controller="CounterCtrl">
    <span ng-bind="counter"></span>
    <button ng-click="counter=counter+1">increase</button>
</div>
```

```bash
function CounterCtrl($scope) {
    $scope.counter = 1;
}
```
以上代码实现的是每当点击一次按钮，界面上的数字就增加一。然后修改下代码变成：
```bash
var app = angular.module("test", []);

app.directive("myclick", function() {
    return function (scope, element, attr) {
        element.on("click", function() {
            scope.counter++;
            scope.$digest() //增加这一行就可以了
        });
    };
});

app.controller("CounterCtrl", function($scope) {
    $scope.counter = 0;
})
```
```bash
<body ng-app="test">
    <div ng-controller="CounterCtrl">
        <button myclick>increase</button>
        <span ng-bind="counter"></span>
    </div>
</body>
```

这个时候点击按钮，界面上的数字不会增加，打开调试器，发现数据已经增加了，数据变化了，但是视图并没有更新，这时候在scope.count++后面加上scope.$digest就可以了。为什么会这样呢？

如果我们自己实现angularjs的双向绑定该怎么实现呢？

```bash
<!DOCTYPE html>
<html>
    <head>
        <meta charset="utf-8" />
        <title>two-way binding</title>
    </head>
    <body onload="init()">
        <button ng-click="inc">
            increase 1
        </button>
        <button ng-click="inc2">
            increase 2
        </button>
        <span style="color:red" ng-bind="counter"></span>
        <span style="color:blue" ng-bind="counter"></span>
        <span style="color:green" ng-bind="counter"></span>

        <script type="text/javascript">
            /* 数据模型区开始 */
            var counter = 0;

            function inc() {
                counter++;
            }

            function inc2() {
                counter+=2;
            }
            /* 数据模型区结束 */

            /* 绑定关系区开始 */
            function init() {
                bind();
            }

            function bind() {
                var list = document.querySelectorAll("[ng-click]");
                for (var i=0; i<list.length; i++) {
                    list[i].onclick = (function(index) {
                        return function() {
                            window[list[index].getAttribute("ng-click")]();
                            apply();
                        };
                    })(i);
                }
            }

            function apply() {
                var list = document.querySelectorAll("[ng-bind='counter']");
                for (var i=0; i<list.length; i++) {
                    list[i].innerHTML = counter;
                }
            }
            /* 绑定关系区结束 */
        </script>
    </body>
</html>
```

可以看到，在这么一个简单的例子中，我们做了一些双向绑定的事情。从两个按钮的点击到数据的变更，这个很好理解，但我们没有直接使用DOM的onclick方法，而是搞了一个ng-click，然后在bind里面把这个ng-click对应的函数拿出来，绑定到onclick的事件处理函数中。为什么要这样呢？因为数据虽然变更了，但是还没有往界面上填充，我们需要在此做一些附加操作。

从另外一个方面看，当数据变更的时候，需要把这个变更应用到界面上，也就是那三个span里。但由于Angular使用的是脏检测，意味着当改变数据之后，你自己要做一些事情来触发脏检测，然后再应用到这个数据对应的DOM元素上。问题就在于，怎样触发脏检测？什么时候触发？

我们知道，一些基于**setter的框架(vue)**，它可以在给数据设值的时候，对DOM元素上的绑定变量作重新赋值。脏检测的机制没有这个阶段，它没有任何途径在数据变更之后立即得到通知，所以只能在每个事件入口中手动调用apply()，把数据的变更应用到界面上。在真正的Angular实现中，这里先进行脏检测，确定数据有变化了，然后才对界面设值。

所以，我们在ng-click里面封装真正的click，最重要的作用是为了在之后追加一次apply()，把数据的变更应用到界面上去。

那么，为什么在ng-click里面调用$digest的话，会报错呢？因为Angular的设计，同一时间只允许一个$digest运行，而ng-click这种内置指令已经触发了$digest，当前的还没有走完，所以就出错了。

其实当你写下表达式如{{ aModel }}时，AngularJS会为你在scope模型上设置一个watcher，它用来在数据发生变化的时候更新view。这里的watcher和你会在AngularJS中设置的watcher是一样的：

对于所有绑定给同一$scope元素的UI对象，只会添加一个$watch到$watch列表中（一个数据一个$watcher,对象会有一个，里面的值还会有，数组中每个对象都有一个 ）。这些$watch列表会在$digest循环中通过一个叫做“脏值检查”的程序解析

- 假设你在一个ng-click指令对应的handler函数中更改了scope中的一条数据，
- 此时AngularJS会**自动**地通过调用$digest()来触发一轮$digest循环。
- 当$digest循环开始后，它会触发每个watcher。
- 这些watchers会检查scope中的当前model值是否和上一次计算得到的model值不同。
- 如果不同，那么对应的回调函数会被执行。调用该函数的结果，就是view中的表达式内容。

我们在angularjs通过ng-click等指令绑定事件，就是通过这些指令，anuglarjs在数据变化的时候可以自动帮我们$digest完成数据的更新，如果数据更新了，视图没有更新，就需要我们手动调用$digest()来进行更新！

#### $digest和$apply
在angularjs中，有$apply和$digest两个函数，我们刚才是通过$digest来让这个数据应用到界面上。但这个时候，也可以不用$digest，而是使用$apply，效果是一样的，那么，它们的差异是什么呢？

最直接的差异是，$apply可以带参数，它可以接受一个函数，然后在应用数据之后，调用这个函数。所以，一般在集成非Angular框架的代码时，可以把代码写在这个里面调用。

#### 参考资料
[1][Angular沉思录（一）数据绑定](https://github.com/xufei/blog/issues/10)  
[2][权威指南-AngularJS的数据绑定](https://github.com/S-iscoming/angularBlog/wiki/2.4.0-angular%E5%8F%8C%E5%90%91%E6%95%B0%E6%8D%AE%E7%BB%91%E5%AE%9A#%E6%9D%83%E5%A8%81%E6%8C%87%E5%8D%97-angularjs%E7%9A%84%E6%95%B0%E6%8D%AE%E7%BB%91%E5%AE%9A)  
[3][AngularJS的数据双向绑定是怎么实现的？
](https://www.jianshu.com/p/0ce33902a7b7)