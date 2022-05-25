---
title: JS常用代码
tags:
  - JS
  - 工具类
categories: JS
thumbnail: 'https://s1.ax1x.com/2022/03/14/bLyk8K.md.png'
date: 2022-05-25 15:39:14
---
> 部分Js常用方法

<!--more-->
# 世界坐标和节点坐标转换  
//把cocos的坐标转成世界坐标pos1 （只能父节点转）
var pos1 = this.cocos1.parent.convertToWorldSpaceAR(this.cocos1.getPosition());
cc.log(pos1)

//把（世界坐标pos1）转成相对于节点cocos的坐标
var pos2 = this.cocos1.convertToNodeSpaceAR(pos1);
cc.log(pos2)

```
    * 节点坐标转换
    * @param node 欲要转换的节点
    * @param newParent 新父节点
    */
    convertToParent(node: cc.Node, newParent: cc.Node, offset?: cc.Vec2): cc.Vec2 {
        const worldPos = node.convertToWorldSpaceAR(offset || cc.v2(0, 0));
        return newParent.convertToNodeSpaceAR(worldPos);
    },
```

# 通过两点算出角度的方法  
```
getAngle:function(start,end){
    var x = end.x - start.x
    var y = end.y - start.y
    var hypotenuse = Math.sqrt(x*x + y*y)

    var cos = x / hypotenuse
    var radian = Math.acos(cos)

    //求出弧度
    var angle = 180 / (Math.PI / radian)
    //用弧度算出角度
    if(y < 0){
        angle = 0-angle
    }else if(y == 0 && x < 0){
        angle = 180
    }
    return 90-angle
},
```

# Js去掉空格方法  
```
//实现
String.prototype.trim = function() {
  var str = this,
  str = str.replace(/^\s\s*/, ''),
  ws = /\s/,
  i = str.length;
  while (ws.test(str.charAt(--i)));
  return str.slice(0, i + 1);
}


const greeting = '   Hello world!   ';
console.log(greeting.trim()); //执行
```

# 输入一个值，返回其数据类型
```
function type(para) {
    return Object.prototype.toString.call(para)
}
```

# 数组去重  
```
function unique1(arr) {
    return [...new Set(arr)]
}

function unique2(arr) {
    var obj = {};
    return arr.filter(ele => {
        if (!obj[ele]) {
            obj[ele] = true;
            return true;
        }
    })
}

function unique3(arr) {
    var result = [];
    arr.forEach(ele => {
        if (result.indexOf(ele) == -1) {
            result.push(ele)
        }
    })
    return result;
}
```

# 字符串去重 
```
String.prototype.unique = function () {
    var obj = {},
        str = '',
        len = this.length;
    for (var i = 0; i < len; i++) {
        if (!obj[this[i]]) {
            str += this[i];
            obj[this[i]] = true;
        }
    }
    return str;
}

###### //去除连续的字符串 
function uniq(str) {
    return str.replace(/(\w)\1+/g, '$1')
}
```

# 深拷贝 浅拷贝
```
//深克隆（深克隆不考虑函数）
function deepClone(obj, result) {
    var result = result || {};
    for (var prop in obj) {
        if (obj.hasOwnProperty(prop)) {
            if (typeof obj[prop] == 'object' && obj[prop] !== null) {
                // 引用值(obj/array)且不为null
                if (Object.prototype.toString.call(obj[prop]) == '[object Object]') {
                    // 对象
                    result[prop] = {};
                } else {
                    // 数组
                    result[prop] = [];
                }
                deepClone(obj[prop], result[prop])
    } else {
        // 原始值或func
        result[prop] = obj[prop]
    }
  }
}
return result;
}

// 深浅克隆是针对引用值
function deepClone(target) {
    if (typeof (target) !== 'object') {
        return target;
    }
    var result;
    if (Object.prototype.toString.call(target) == '[object Array]') {
        // 数组
        result = []
    } else {
        // 对象
        result = {};
    }
    for (var prop in target) {
        if (target.hasOwnProperty(prop)) {
            result[prop] = deepClone(target[prop])
        }
    }
    return result;
}
```

# 防抖
```
function debounce(handle, delay) {
    var timer = null;
    return function () {
        var _self = this,
            _args = arguments;
        clearTimeout(timer);
        timer = setTimeout(function () {
            handle.apply(_self, _args)
        }, delay)
    }
}
```

# 节流
```
function throttle(handler, wait) {
    var lastTime = 0;
    return function (e) {
        var nowTime = new Date().getTime();
        if (nowTime - lastTime > wait) {
            handler.apply(this, arguments);
            lastTime = nowTime;
        }
    }
}
```