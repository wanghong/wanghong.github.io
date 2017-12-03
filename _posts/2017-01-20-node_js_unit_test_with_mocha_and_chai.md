---
layout: post
title:  "使用mocha和chai做nodejs单元测试"
permalink: /article/node_js_unit_test_with_mocha_and_chai
categories: programs
author: wanghong
---

# 一，说明

nodejs根据语言版本不同，有许多种不同的写法，异步方法调用也有多种写法，最近写了一个公司内部的npm包，主要使用的nodejs版本为4.3.0，用了ES6支持的generator。因为此npm包功能非常核心，所以写了较详细的单元测试。此处总结一下在写单元测试时的一些方法，以及遇到的一些问题。

# 二，nodejs单元测试包简介

1. nodejs中最负盛名的单元测试框架是mocha，据官方资料，它已经被超过10万个npm包所依赖。其拥有丰富的，可配置和可扩展的测试特性。mocha默认使用nodejs内置的断言库assert，但更好的选择是使用第三方的断言库，根据单元测试和业务的需要。
2. [chia](http://chaijs.com/)第三方断言库，支持各种断言风格：expect，assert，should，详见其官方文档。
3. [sinon](https://github.com/sinonjs/sinon)用于测试stubs和mocks。
4. [rewire](https://github.com/jhnns/rewire)用于重写包引入机制， 使用得我们可以测试module中的私有方法。
5. [supertest](https://github.com/visionmedia/supertest)用于测试web项目，模拟request请求

# 三，mocha

mocha的使用非常简单，官方文档上有详细的说明，但此处还是做一些简单的说明。

**1. 安装(推荐安装成全局的npm)**
{% highlight bash %}
$ npm install --global mocha
{% endhighlight %}
或者将依赖写入了package.json中的devDependencies，例如：
{% highlight javascript %}
{
  "devDependencies": {
    "mocha": "^3.2.0",
    "chai": "^3.5.0",
    "co": "^4.6.0",
    "rewire": "^2.5.2",
    "sinon": "^1.17.7"
  }
}
{% endhighlight %}

**2. 一个简单的测试用例(来自官网)**

创建一个名为test的目录，在目录中新建test.js(~/nodejs/test/test.js)文件，输入以下代码：
{% highlight javascript %}
ert = require('assert');
describe('Array', function() {
  describe('#indexOf()', function() {
    it('should return -1 when the value is not present', function() {
      assert.equal(-1, [1,2,3].indexOf(4));
    });
  });
});
{% endhighlight %}
命令行中执行：

{% highlight bash %}
peachcat@peachcat:~/nodejs $ mocha


  Array
    #indexOf()
      ✓ should return -1 when the value is not present


  1 passing (34ms)
{% endhighlight %}

**3. 一个异步调用的测试例子（来自官方）**

异步调用完成之后需要调用mocha提供的callback，以完成此测试：

{% highlight javascript %}
describe('User', function() {
  describe('#save()', function() {
    it('should save without error', function(done) {
      var user = new User('Luna');
      user.save(done);
    });
  });
});
{% endhighlight %}

**4. Promise：除了直接调用mocha的callback之外，还可以直接返回一个promise**

{% highlight javascript %}
const assert = require('assert');

it('should complete this test', function () {
  return new Promise(function (resolve, reject) {
    assert.ok(true);
    resolve();
  });
});
{% endhighlight %}

**5. 和generator一起使用**

generator既没有callback，也没有返回promise，所以mocha无法直接处理，但我们可以使用co来执行generator，并且co返回的就是promise。例子：
{% highlight javascript %}
'use strict';
var fs = require("fs") ;
let co = require('co');
var assert = require('assert');

function readFile(path){
  return new Promise(function(resolve, reject){
    fs.readFile(path, "utf8", function(err, data){
      if(err){
        reject(err);
      }else{
        resolve(data);
      }
    });
  });
}

describe("Generator", function(){
  it('test with co', function () {
    return co(function*(){
      let txt = yield readFile("a.txt");
      assert.equal(txt, "file A\n");
    });
  });
});
{% endhighlight %}

**6. 使用before, after, beforeEach和afterEach**

mocha支持before(), after(), beforeEach(), and afterEach()这类方法，用于设置预置的测试条件和清理测试之后的资源。使用方式如下：
{% highlight javascript %}
describe('hooks', function() {
  before(function() {
    // runs before all tests in this block
  });
  after(function() {
    // runs after all tests in this block
  });
  beforeEach(function() {
    // runs before each test in this block
  });
  afterEach(function() {
    // runs after each test in this block
  });
  // test cases
});
{% endhighlight %}

# 四，chia

chia支持多种写法： Should，Expect，Assert，这里主要使用的是expect方式，官方有个大概的例子：

{% highlight javascript %}
var expect = chai.expect;

expect(foo).to.be.a('string');
expect(foo).to.equal('bar');
expect(foo).to.have.lengthOf(3);
expect(tea).to.have.property('flavors').with.lengthOf(3);
{% endhighlight %}

**1.  equal和eql**

**equal**： 使用"==="比较两个值是否相等，当值为引用类型时，这只能比较引用的地址是否相等，若两个对象内容相同，但引用地址不同，即为两个对象时，使用equal时，断言会失败。

{% highlight javascript %}
expect({ foo: 'bar' }).to.not.equal({ foo: 'bar' });
{% endhighlight %}

使用deep可以对比两个对象的内容，如：
{% highlight javascript %}
expect({ foo: 'bar' }).to.deep.equal({ foo: 'bar' });
{% endhighlight %}

**eql**： 等价于deap.equal，如：
{% highlight javascript %}
expect({ foo: 'bar' }).to.eql({ foo: 'bar' });
expect([ 1, 2, 3 ]).to.eql([ 1, 2, 3 ]);
{% endhighlight %}

**2. 异常断言**

使用"*.throw*"可以断言会抛出异常的方法，但是直接调用方法时，会抛出异常，导到测试直接失败，所以需要对会抛出异常的方法进行包装。比如：m.js
{% highlight javascript %}
module.exports.throwError = function(a){
  if( a > 100){
    throw new ReferenceError('This is a bad function.');;
  } else {
    return a * 10;
  }
};
{% endhighlight %}

测试代码：
{% highlight javascript %}
'use strict';

let expect = require('chai').expect;
let m = require("../m");

describe("Chia throw test", function(){
  it("should throw ReferenceError", function(){
    expect(m.throwError(101)).to.throw(ReferenceError);
  });
});
{% endhighlight %}

此测试用例无法通过：
{% highlight bash %}
peachcat@peachcat:~/nodejs/mocha $ mocha test/m.js


  Chia throw test
    1) should throw ReferenceError


  0 passing (57ms)
  1 failing

  1) Chia throw test should throw ReferenceError:
     AssertionError: expected [Function] to throw ReferenceError
      at Context.<anonymous> (test/m.js:8:34)
{% endhighlight %}

将throwError 方法包装起来，修改之后的代码如下：
{% highlight javascript %}
'use strict';

let expect = require('chai').expect;
let m = require("../m");

describe("Chia throw test", function(){
  it("should throw ReferenceError", function(){
    let warpper = function(){ m.throwError(101); }
    expect(warpper).to.throw(ReferenceError);
  });
});
{% endhighlight %}

再次跑测试，顺利通过（从上面的代码来看，会抛出异常的方法，是在expect中被调用的。）：
{% highlight javascript %}
peachcat@peachcat:~/nodejs/mocha $ mocha test/m.js


  Chia throw test
      ✓ should throw ReferenceError


        1 passing (51ms)
{% endhighlight %}

**generator方法中抛出异常**：generator方法无法在expect中被执行，即使使用了包装方法也无法调用，因此generator检测抛出异常时，可以使用下面的方式，使用co调用generator方法，在catch中获得异常，并对此异常进行断言。例子如下：
{% highlight javascript %}
'use strict';
let co = require('co');
let expect = require('chai').expect;

function* generatorMethod(a){
  if (a > 10){
    throw new Error("Generator Error");
  } else {
    return a * 10;
  }
}

describe("Generator", function(){
  it('test throw Error', function () {
    return co(function*(){
      let results = yield generatorMethod(11);
    }).catch(function(err){
      expect(err.message).to.equal("Generator Error");
    });
  });
});
{% endhighlight %}

# 五，sinon

sinon可以做许多事情，比如：  stub, mock，可以模拟ajax请求等等。这里我们主要使用了stub功用。下面就stub中的几种使用过的情况做下说明：

1.stub一个方法，下面的代码，将返回一个stub方法替代object中的"method"方法

{% highlight javascript %}
var stub = sinon.stub(object, "method");
{% endhighlight %}

2.stub一个方法，并且在使用指定参数被调用时，返回指定的数据。

{% highlight javascript %}
var stub = sinon.stub(object, "method");
stub.withArgs(42).returns(1);
stub.withArgs(1).throws("TypeError");

object.method(42);  // return 1
object.method(1);   // throws TypeError
{% endhighlight %}

3.使用sandbox隔离stub，nodejs中模块，在全局上是同一个对象，因此对某个模块进行了stub，后面的测试还需要使用此模块时会相互影响，因此可以使用sinon的sandbox功能，将stub进行隔离。例如：

{% highlight javascript %}
describe('get_all_miss_jsons', function(){
  it("#getAllMissJsons", sinon.test(function(){
    //使用this.stub替代全局的sinon.stub
    this.stub(missJsonService, "fetch").returns(someObject);

    //assert
  }));
});
{% endhighlight %}

4.spies、stub和mock的区别

sinon中提供了几种有用的辅助测试功能，spies，stub和mock，下面说明这3种方式的意义和适合场景。

| | 定义 | 适合场景 |
| ------------- |-------------| ----- |
| spies | spy是一个方法，测试时它会记录下每一次此方法被调用时的参数，返回值，或者抛出的异常。spy方法可以一个匿名方法，或者它可以包装一个已存在的方法。 | 在测试callback和了解某个特定的方法在整个测试中是怎么被使用的是非常有用的。spy也可以包装一个已有方法，同样可以统计其调用情况，[例子](http://sinonjs.org/docs/#spies)。见表下方的spy例子。|
| stub | stub是一个预先编写的方法，用于替代某个被测试的方法。stub的使用场景如下 | 1. 单元测试中控制一个方法，强制其走到预置的代码路径；包括强制一个方法抛出异常以测试出错的情况。<br/>2.避免一个方法被直接调用（比如：逻辑太复杂，和本次测试无关的方法；调用耗时太长的方法等）。|
| mock | mock,类似于stub，同样是一个预置的方法，同样用于在测试中替代某个被测试的方法。与stub不同的是，**mock方法在测试中必须被调用，否则测试用例失败**。| 1. mock应该只被用于有单元测试的方法上。如果你想控制你的单元测试是怎么被使用，并且在方法被真实调用之前，请使用mock。<br/>2. mock有内置的断言，可能让你的测试失败。如果你不用在某个特殊调用上使用断言，就没必要使用mock。另外，一个独单的测试中，不要使用超过1个以上的mock。|

一个spy的例子：
{% highlight javascript %}
"test should call subscribers on publish": function () {
    var callback = sinon.spy();    //假装一个callback
    PubSub.subscribe("message", callback);

    PubSub.publishSync("message");

    assertTrue(callback.called); //确定此方法做为回调方法被调用了。
}
{% endhighlight %}

# 六，rewire

nodejs模块中，有许多私有方法，即未通过module.exports导出的方法，可以通过rewire这个npm包提供的方法rewire替换require，就有办法可以访问这些私有方法，从而对其做单元测试。例如：

1.在模块m.js中，存在私有方法_add

{% highlight javascript %}
'use strict';

function _add(a, b){
  return a + b;
};

module.exports.add = function(a, b){
  if (a > 100 ){
    a = a * 2;
  }
  return _add(a, b);
};
{% endhighlight %}

2.使用rewire测试此方法：

{% highlight javascript %}
let rewire = require('rewire');
let m      = rewire("../m");
var expect = require('chai').expect;

describe("rewire", function(){
  it("test private method in module `m`" ,function(){
    let _add = m.__get__('_add');
    expect(_add(4, 3)).to.eq(7);
  });
});
{% endhighlight %}

# 七，总结

原来的npm包，功能相对简单一些，没有单元测试，代码就相对比较集中，没有对各种功能进行区分。但我决定要对核心功能写单元测试时，不得不对代码结构进行修改，单元测试的需求，强制写出结构更清晰的代码，功能被拆分得更小，小功能的逻辑相对更内聚。

总结一下添加单元测试之后，得到的好处吧：

1. 代码结构更清晰。
2. 功能拆分得更详细。
3. 每个功能点的代码更内聚，以便可以单独测试
4. 对后面的修改更有保障，在开发中就能发现问题，比如更换了某个功能的实现 ，跑一次测试，如果测试挂掉，说明代码逻辑和之前的逻辑并不等价，新的实现可能有问题。
5. 可以尝试TDD方式开发，先写测试，对要实现的功能会更清晰。
