title: 单元测试——Qunit
date: 2016-05-08
category: 技术
tags: [测试]
---

## 为什么需要单元测试
+ 正确性：测试可以验证代码的正确性，在上线前做到心里有底
+ 自动化：当然手工也可以测试，通过console可以打印出内部信息，但是这是一次性的事情，下次测试还需要从头来过，效率不能得到保证。通过编写测试用例，可以做到一次编写，多次运行
+ 解释性：测试用例用于测试接口、模块的重要性，那么在测试用例中就会涉及如何使用这些API。其他开发人员如果要使用这些API，那阅读测试用例是一种很好地途径，有时比文档说明更清晰
+ 驱动开发，指导设计：代码被测试的前提是代码本身的可测试性，那么要保证代码的可测试性，就需要在开发中注意API的设计，TDD将测试前移就是起到这么一个作用
+ 保证重构：互联网行业产品迭代速度很快，迭代后必然存在代码重构的过程，那怎么才能保证重构后代码的质量呢？有测试用例做后盾，就可以大胆的进行重构
<!-- more -->

## 测试工具分类
+ Qunit、jasmine、mocha、jest、intern
各自有各自的优缺点，相对比较流行的有jasmine、mocha和Qunit（待验证）
+ Qunit诞生于Jquery团队，但是并不依赖jquery

## Qunit内置断言
+ ok()
    + 只有一个请求的参数，如果返回值TRUE，这个断言通过，否则就失败。另外它接受1个字符串作为测试测试结果的显示。
+ equal()
    + 常用于简单的使用（==）运算符比较期望参数和真实值，当他们相等的时候，断言通过，否则断言失败。这2个值都会在测试结果中显示，额外的也会显示测试消息。
    + 严格比较（===），使用 strictEqual() 代替
+ deepEqual()
    + 比 equal() 并且有更多选项。代替使用简单的比较运算符(==),使用更加准确的比较运算符（===）,也能操作NaN，dates，正则表达式，数组，方法等类型，而 equal() 只能测试基本对象类型。

## 同步回调测试
+ 调用test() 和 expect() 开始测试，使用这个期望断言数量作为参数
实际例子

    ```
    QUnit.test( "a test", function( assert ) {
        expect( 1 );
        var $body = $( "body" );
        $body.on( "click", function() {
            assert.ok( true, "body was clicked!" );
        });
        $body.trigger( "click" );
    });
    ```
    ## 异步回调测试
    + 当测试代码开始一个timeout 和 interval 或者一个ajax 请求，通过expect、asyncTest 和 start配合使用实现

    ```
    QUnit.asyncTest( "asynchronous test: video ready to play", function( assert ) {
        expect( 1 );
        var $video = $( "video" );

        $video.on( "canplaythrough", function() {
            assert.ok( true, "video has loaded and is ready to play" );
            QUnit.start();
        });
    });
    ```
## 测试用户行为
+ 栗子

    ```
        function KeyLogger( target ) {
            if ( !(this instanceof KeyLogger) ) {
                return new KeyLogger( target );
            }
            this.target = target;
            this.log = [];
            var self = this;

            this.target.off( "keydown" ).on( "keydown", function( event ) {
                self.log.push( event.keyCode );
            });
        }

        QUnit.test( "keylogger api behavior", function( assert ) {
            var event,
            $doc = $( document ),
            keys = KeyLogger( $doc );
            // trigger event
            event = $.Event( "keydown" );
            event.keyCode = 9;
            $doc.trigger( event );
            // verify expected behavior
            assert.equal( keys.log.length, 1, "a key was logged" );
            assert.equal( keys.log[ 0 ], 9, "correct key was logged" );
        });
    ```
## 自定义断言
+ 定义一个可以重复使用的方法。条用Qunit.push 在方法内部然后告诉QUnit断言已经发生。
栗子

    ```
    QUnit.assert.contains = function(needle, haystack, message) {
        var actual = (function() {
            var result = haystack.some(function(val) {
                return val.indexOf(needle) > -1;
            });
            return result ? needle : '';
        }());
        this.push(actual, actual, needle, message);
    };
    QUnit.test("retrieving object keys", function(assert) {
        var arrayKeys = ["apple", "big"];
        assert.contains("apple", arrayKeys, "Array keys");
        assert.contains("bigbang", arrayKeys, "Array keys");
    });
    ```
