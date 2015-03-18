# 截图生成部分

## webdrivercss.js

### 返回一个WebdriverCSS实例，这个实例接受一个Webdriverio的实例和一个配置相关的Object

> [webdrivercss.js:213](https://github.com/webdriverio/webdrivercss/blob/master/lib/webdrivercss.js#L213)

```JavaScript
module.exports.init = function(webdriverInstance, options) {
    return new WebdriverCSS(webdriverInstance, options);
};
```

> [webdrivercss.js:21](https://github.com/webdriverio/webdrivercss/blob/master/lib/webdrivercss.js#L21)

```JavaScript
var WebdriverCSS = function(webdriverInstance, options) {
    options = options || {};

    if(!webdriverInstance) {
        throw new Error('A WebdriverIO instance is needed to initialise WebdriverCSS');
    }
    ...
}
```
### 配置初始化完成，工作流程开始，并添加截图相关Command

注意下面代码中的`this.instance`实际为***Webdriverio***的实例。因此下面代码通过[WebdriverIO中的addCommand方法](https://github.com/webdriverio/webdriverio/blob/master/lib/webdriverio.js#L80)向WebdriverIO的指令集中添加了四条指令：\[***saveViewportScreenshot***, ***saveDocumentScreenshot***, ***webdrivercss***, ***sync***\]和其对应的操作。下面我们先看webdrivercss的工作流程里面都做了什么。

> [webdrivercss.js:79](https://github.com/webdriverio/webdrivercss/blob/master/lib/webdrivercss.js#L79)

```JavaScript
var WebdriverCSS = function(webdriverInstance, options) {
    ...
    this.instance = webdriverInstance;
    ...
    this.instance.addCommand('saveViewportScreenshot', viewportScreenshot.bind(this)); // 这里通过WebdriverIO的API
    this.instance.addCommand('saveDocumentScreenshot', documentScreenshot.bind(this));
    this.instance.addCommand('webdrivercss', workflow.bind(this));
    this.instance.addCommand('sync', syncImages.bind(this));

    return this;
}
```

## workflow.js

### 接受一个`pageName`，配置Object，一个callback。`pageName`用来对对比的page进行命名，配置必须为含有name属性的对象，callback会被送到后续的每一步流程中，后续讲解

> [workflow.js:7 ~ workflow.js:38](https://github.com/webdriverio/webdrivercss/blob/master/lib/workflow.js#L7)

```JavaScript
module.exports = function(pagename, args) {
    /*!
     * make sure that callback contains chainit callback
     */
    var cb = arguments[arguments.length - 1];

    this.needToSync = true;

    /*istanbul ignore next*/
    if (typeof args === 'function') {
        args = {};
    }

    if (!(args instanceof Array)) {
        args = [args];
    }

    /**
     * parameter type check
     */
    /*istanbul ignore next*/
    if (typeof pagename === 'function') {
        throw new Error('A pagename is required');
    }
    /*istanbul ignore next*/
    if (typeof args[0].name !== 'string') {
        throw new Error('You need to specify a name for your visual regression component');
    }

    var queuedShots = JSON.parse(JSON.stringify(args)),
        currentArgs = args.shift();
    ...
}
```

这里要着重讲解一下workflow.js的当前上下文以及他封装的`context`。由于上面贴出过的代码段中有这种写法[`this.instance.addCommand('webdrivercss', workflow.bind(this));`](https://github.com/webdriverio/webdrivercss/blob/master/lib/webdrivercss.js#L81)，因此此时workflow.js的上下文为Webddrivercss的实例对象，包含以下配置的属性及其原型链

> [webdrivercss.js:43 ~ webdrivercss.js:71](https://github.com/webdriverio/webdrivercss/blob/master/lib/webdrivercss.js#L43)

```Javascript
var WebdriverCSS = function(webdriverInstance, options) {
    ...
    this.screenshotRoot = options.screenshotRoot || 'webdrivercss';
    this.failedComparisonsRoot = options.failedComparisonsRoot || (this.screenshotRoot + '/diff');
    this.misMatchTolerance = options.misMatchTolerance || 0.05;
    this.screenWidth = options.screenWidth || [];
    this.warning = [];
    this.resultObject = {};
    this.instance = webdriverInstance;
    this.updateBaseline = (typeof options.updateBaseline === 'boolean') ? options.updateBaseline : false;

    /**
     * sync options
     */
    this.key = options.key;
    this.applitools = {
        apiKey: options.key,
        saveNewTests: true, // currently will always happen.
        saveFailedTests: this.updateBaseline,
        batchId: _generateUUID()
    };
    this.host = 'https://eyessdk.applitools.com';
    this.headers = {
        'Accept': 'application/json',
        'Content-Type': 'application/json'
    };
    this.reqTimeout = 5 * 60 * 1000;
    this.user = options.user;
    this.api  = options.api;
    this.usesApplitools = typeof this.applitools.apiKey === 'string' && !this.api;
    this.saveImages = options.saveImages || !this.usesApplitools;
    ...
}
```

workflow.js中的context设置如下(注意this指的是上面说的Webddrivercss的实例对象)：

> [workflow.js:40 ~ workflow.js:72](https://github.com/webdriverio/webdrivercss/blob/master/lib/workflow.js#L40)

```JavaScript
module.exports = function(pagename, args) {
    ...
    var context = {
        self: this,

        /**
         * default attributes
         */
        misMatchTolerance:      this.misMatchTolerance,
        screenshotRoot:         this.screenshotRoot,
        failedComparisonsRoot:  this.failedComparisonsRoot,

        instance:       this.instance, // 这里需要注意，后面会用到很多
        pagename:       pagename,
        applitools:     {
            apiKey: this.applitools.apiKey,
            appName: pagename,
            saveNewTests: this.applitools.saveNewTests,
            saveFailedTests: this.applitools.saveFailedTests,
            batchId: this.applitools.batchId // Group all sessions for this instance together.
        },
        currentArgs:    currentArgs,
        queuedShots:    queuedShots,
        baselinePath:   this.screenshotRoot + '/' + pagename + '.' + currentArgs.name + '.baseline.png',
        regressionPath: this.screenshotRoot + '/' + pagename + '.' + currentArgs.name + '.regression.png',
        diffPath:       this.failedComparisonsRoot + '/' + pagename + '.' + currentArgs.name + '.diff.png',
        screenshot:     this.screenshotRoot + '/' + pagename + '.png',
        isComparable:   false,
        warnings:       [],
        newScreenSize:  0,
        pageInfo:       null,
        updateBaseline: (typeof currentArgs.updateBaseline === 'boolean') ? currentArgs.updateBaseline : this.updateBaseline,
        screenWidth:    currentArgs.screenWidth || [].concat(this.screenWidth), // create a copy of the origin default screenWidth
        cb:             cb
    };
    ...
}
```

继续读会发现workflow的工作流程：
- startSession.js
- setScreenWidth.js
- makeScreenshot.js
- renameFiles.js
- getPageInfo.js
- cropImage.js
- compareImages.js
- saveImageDiff.js
- asyncCallback.js

注意：源码中将每个过程的工作全部绑定`context`为上下文；由于使用了`async.waterfall()`，因此每一个模块都会接受一个callback参数，在源码中被命名成了`done`(有关这个回调函数具体的使用细则请参考[async.waterfall](https://www.npmjs.com/package/async#waterfall))
 
> [workflow.js:81 ~ workflow.js:128](https://github.com/webdriverio/webdrivercss/blob/master/lib/workflow.js#L81)

```JavaScript
module.exports = function(pagename, args) {
    ...
    async.waterfall([
        /**
         * initialize session
         */
        require('./startSession.js').bind(context),

        /**
         * if multiple screen width are given resize browser dimension
         */
        require('./setScreenWidth.js').bind(context),

        /**
         * make screenshot via [GET] /session/:sessionId/screenshot
         */
        require('./makeScreenshot.js').bind(context),

        /**
         * check if files with id already exists
         */
        require('./renameFiles.js').bind(context),

        /**
         * get page informations
         */
        require('./getPageInfo.js').bind(context),

        /**
         * crop image according to user arguments and its position on screen and save it
         */
        require('./cropImage.js').bind(context),

        /**
         * compare images
         */
        require('./compareImages.js').bind(context),

        /**
         * save image diff
         */
        require('./saveImageDiff.js').bind(context)
    ],
        /**
         * run workflow again or execute callback function
         */
        require('./asyncCallback.js').bind(context)

    );
}
```

由于这部分是截图功能的解读，因此我们直接跳过其他去查看`makeScreenshot.js`做了什么

## makeScreenshot.js

makeScreenshot模块首先会接收Webdrivercss的第二个参数中的元素信息，并不会在截图时考虑这些元素(屏蔽这些元素的比较)

> [makeScreenshot.js:7 ~ makeScreenshot.js:44](https://github.com/webdriverio/webdrivercss/blob/master/lib/makeScreenshot.js#L7)

```JavaScript
module.exports = function(done) { // async.waterfall中每一层级传递的回调函数
    /**
     * take actual screenshot in given screensize just once
     */
    if(this.self.takeScreenshot === false) { // context.self(Webdrivercss实例).takeScreenshot，保证截图一次
        return done();
    }

    this.self.takeScreenshot = false; 

    /**
     * gather all elements to hide
     */
    var hiddenElements = [];
    this.queuedShots.forEach(function(args) { // Webdrivercss中的第二个配置参数如：[{ name: 'header', elem: '#header' }, { name: 'hero', elem: '//*[@id="hero"]/div[2]' }]
        if(typeof args.hide === 'string') {
            hiddenElements.push(args.hide);
        }
        if(args.hide instanceof Array) {
            hiddenElements = hiddenElements.concat(args.hide);
        }
    });

    /**
     * hide elements
     */
    if(hiddenElements.length) { // 将Webdrivercss中的第二个配置参数中标注的元素的visibility设置为hidden。这里是为了忽略这些元素
        this.instance.selectorExecute(hiddenElements, function() {

            for(var i = 0; i < arguments.length; ++i) {
                for(var j = 0; j < arguments[i].length; ++j) {
                    arguments[i][j].style.visibility = 'hidden';
                }
            }

        }, function(){});
    }
    ...
}
```
继续刚才的代码，下面部分的代码首先会调用[Webdriverio.pause()](http://webdriver.io/api/utility/pause.html)方法让浏览器等待100ms，然后在调用由Webdrivercss初始化进去的`saveDocumentScreenshot`指令，随后会将刚才不可见(忽略)的元素重新变的可见

> [makeScreenshot.js:49 ~ makeScreenshot.js:64](https://github.com/webdriverio/webdrivercss/blob/master/lib/makeScreenshot.js#L49)

```JavaScript
module.exports = function(done) {
    ...
    // 这里相当于调用了Webdriverio实例的pause方法
    // this.screenshot === context.screenshot (就是screenshotRoot + '/' + pagename + '.png')
    this.instance.pause(100).saveDocumentScreenshot(this.screenshot, done);

    /**
     * make hidden elements visible again
     */
    if(hiddenElements.length) { // 将刚才忽略的元素的显示属性在设置回来
        this.instance.selectorExecute(hiddenElements, function() {

            for(var i = 0; i < arguments.length; ++i) {
                for(var j = 0; j < arguments[i].length; ++j) {
                    arguments[i][j].style.visibility = 'visible';
                }
            }

        }, function(){});
    }
}
```

下面我们需要了解`saveDocumentScreenshot()`方法到底做了什么

> [documentScreenshot.js:30 ~ documentScreenshot.js:75](https://github.com/webdriverio/webdrivercss/blob/master/lib/documentScreenshot.js#L30)

```JavaScript
module.exports = function documentScreenshot(fileName) { // 接受一个文件名作为参数

    var ErrorHandler = this.instance.ErrorHandler; // 缓存一个Webdriverio的ErrorHandler

    /*!
     * make sure that callback contains chainit callback
     */
    var callback = arguments[arguments.length - 1]; // 取到回调函数done，下方存在：callback === done
    
    /*!
     * parameter check
     */
    if (typeof fileName !== 'string') { // 如果文件名不是String通过done函数回调错误信息，并结束整个waterfall流程
        return callback(new ErrorHandler.CommandError('number or type of arguments don\'t agree with saveScreenshot command'));
    }

    var self = this.instance,
        response = {
            execute: [],
            screenshot: []
        },
        tmpDir = __dirname + '/../.tmp',
        cropImages = [],
        currentXPos = 0,
        currentYPos = 0,
        screenshot = null,
        scrollFn = function(w, h) { // 屏幕滚动函数，就不说其原理了，看注释也知道IE6、7不行
            /**
             * IE8 or older
             */
            if(document.all && !document.addEventListener) {
                /**
                 * this still might not work
                 * seems that IE8 scroll back to 0,0 before taking screenshots
                 */
                document.body.style.marginTop = '-' + h + 'px';
                document.body.style.marginLeft = '-' + w + 'px';
                return;
            }

            document.body.style.webkitTransform = 'translate(-' + w + 'px, -' + h + 'px)';
            document.body.style.mozTransform = 'translate(-' + w + 'px, -' + h + 'px)';
            document.body.style.msTransform = 'translate(-' + w + 'px, -' + h + 'px)';
            document.body.style.oTransform = 'translate(-' + w + 'px, -' + h + 'px)';
            document.body.style.transform = 'translate(-' + w + 'px, -' + h + 'px)';
        };
    ...
}
```

下面部分的代码基本看注释也能理解什么意思，需要注意的是下面代码调用了[Webdriverio.execute](http://webdriver.io/api/protocol/execute.html)，这个函数会在当前浏览器中运行javascript代码并在完成时触发回调函数。此外[async.whilst](https://www.npmjs.com/package/async#whilst)得到功能类似于while循环，直到第一个参数返回false才停止执行第二个参数函数

```JavaScript
module.exports = function documentScreenshot(fileName) {
    ...
    async.waterfall([
        /*!
         * create tmp directory to cache viewport shots
         */
        function(cb) {
            fs.exists(tmpDir, function(exists) {
                return exists ? cb() : fs.mkdir(tmpDir, 0755, cb);
            });
        },

        /*!
         * prepare page scan
         */
        function() {
            var cb = arguments[arguments.length - 1]; // 获取async每一级的callback函数
            // 通过Webdriverio.execute()执行js代码来保证截图时不会出现滚动条，以及全屏滚动
            // 注意cb接受的是execute回调传入的两个参数(err, ret)，err即运行js时的错误，若发生则整个Webdrivercss工作结束
            // ret为一个含有value属性的对象，value的值是所执行js的返回值，下例中的ret即为：
            /*  {
             *      value: {
             *          screenWidth: Math.max(document.documentElement.clientWidth, window.innerWidth || 0),
             *          screenHeight: Math.max(document.documentElement.clientHeight, window.innerHeight || 0),
             *          documentWidth: document.documentElement.scrollWidth,
             *          documentHeight: document.documentElement.scrollHeight
             *      }
             *  }
             */
            self.execute(function() { 
                /**
                 * remove scrollbars
                 */
                document.body.style.overflow = "hidden";

                /**
                 * scroll back to start scanning
                 */
                window.scrollTo(0, 0);

                /**
                 * get viewport width/height and total width/height
                 */
                return {
                    screenWidth: Math.max(document.documentElement.clientWidth, window.innerWidth || 0),
                    screenHeight: Math.max(document.documentElement.clientHeight, window.innerHeight || 0),
                    documentWidth: document.documentElement.scrollWidth,
                    documentHeight: document.documentElement.scrollHeight
                };
            }, cb);
        },

        /*!
         * take viewport shots and cache them into tmp dir
         */
        function(res, cb) {
            // res为上一部分cb传给这里的ret参数，即：
            /*  {
             *      value: {
             *          screenWidth: Math.max(document.documentElement.clientWidth, window.innerWidth || 0),
             *          screenHeight: Math.max(document.documentElement.clientHeight, window.innerHeight || 0),
             *          documentWidth: document.documentElement.scrollWidth,
             *          documentHeight: document.documentElement.scrollHeight
             *      }
             *  }
             */
            // 这里将执行js后的document和screen大小缓存在response.executes数组中
            response.execute.push(res);

            /*!
             * run scan
             */
            async.whilst( // 循环执行直到第一个参数返回false

                /*!
                 * while expression
                 */
                function(callback) { // 循环判断函数：判断当前X是否到达页面最右侧
                    return (currentXPos < (response.execute[0].value.documentWidth / response.execute[0].value.screenWidth));
                },

                /*!
                 * loop function
                 */
                function(finishedScreenshot) { // finishedScreenshot等同于上面的循环判断函数
                    response.screenshot = [];

                    async.waterfall([

                        /*!
                         * take screenshot of viewport
                         */
                        self.screenshot.bind(self), // 给Webdriverio.screenshot()绑定之前的context上下文并执行

                        /*!
                         * cache image into tmp dir
                         */
                        function(res, cb) {
                            // 这里的res等同于这种写法得到得到res：Webdriverio.screenshot(function(res){})，
                            // 因此res为所接图像的base64编码
                            var file = tmpDir + '/' + currentXPos + '-' + currentYPos + '.png';
                            gm(new Buffer(res.value, 'base64')).crop(response.execute[0].value.screenWidth, response.execute[0].value.screenHeight, 0, 0).write(file, cb);
                            response.screenshot.push(res);

                            if (!cropImages[currentXPos]) {
                                cropImages[currentXPos] = [];
                            }

                            cropImages[currentXPos][currentYPos] = file;

                            currentYPos++;
                            if (currentYPos > Math.floor(response.execute[0].value.documentHeight / response.execute[0].value.screenHeight)) {
                                currentYPos = 0;
                                currentXPos++;
                            }
                        },

                        /*!
                         * scroll to next area
                         */
                        function() {
                            self.execute(scrollFn,
                                currentXPos * response.execute[0].value.screenWidth,
                                currentYPos * response.execute[0].value.screenHeight,
                                function(val, res) {
                                    response.execute.push(res);
                                }
                            ).pause(100).call(arguments[arguments.length - 1]);
                        }

                    ], finishedScreenshot);
                },
                cb
            );
        },

        /*!
         * concats all shots
         */
        function(cb) {
            var subImg = 0;

            async.eachSeries(cropImages, function(x, cb) {
                var col = gm(x.shift());
                col.append.apply(col, x);

                if (!screenshot) {
                    screenshot = col;
                    col.write(fileName, cb);
                } else {
                    col.write(tmpDir + '/' + (++subImg) + '.png', function() {
                        gm(fileName).append(tmpDir + '/' + subImg + '.png', true).write(fileName, cb);
                    });
                }
            }, cb);
        },

        /*!
         * crop screenshot regarding page size
         */
        function() {
            gm(fileName).crop(response.execute[0].value.documentWidth, response.execute[0].value.documentHeight, 0, 0).write(fileName, arguments[arguments.length - 1]);
        },

        /*!
         * remove tmp dir
         */
        function() {
            rimraf(tmpDir, arguments[arguments.length - 1]);
        },

        /*!
         * scroll back to start position
         */
        function(cb) {
            self.execute(scrollFn, 0, 0, cb);
        },

        /**
         * enable scrollbars again
         */
        function(res, cb) {
            response.execute.push(res);
            self.execute(function() {
                document.body.style.overflow = "visible";
            }, cb);
        }
    ], function(err) {
        callback(err, null, response);
    });
}
```

至此，关于截图部分的源头已经找到，使用了[Webdriverio.screenshot()](http://webdriver.io/api/protocol/screenshot.html)来截图，那么这个截图函数到底是基于什么原理做的呢？

## Webdriverio.screenshot()

Webdriverio.screenshot()是基于JsonWireProtocol协议(WebDriver Wire Protocol)实现的一个功能

> [webdriverio/lib/protocol/screenshot.js](https://github.com/webdriverio/webdriverio/blob/master/lib/protocol/screenshot.js)

```JavaScript
/**
 *
 * Take a screenshot of the current viewport. To get the screenshot of the whole page
 * use the action command `saveScreenshot`
 *
 * @returns {String} screenshot   The screenshot as a base64 encoded PNG.
 * @callbackParameter error, response
 *
 * @see  https://code.google.com/p/selenium/wiki/JsonWireProtocol#/session/:sessionId/screenshot
 * @type protocol
 *
 */

module.exports = function screenshot () {

    this.requestHandler.create(
        '/session/:sessionId/screenshot',
        arguments[arguments.length - 1]
    );
};
```

因此我们需要去了解JsonWireProtocol协议对于screenshot的定义和实现

> 我们都知道这个`/session/:sessionId/screenshot`这个GET请求是发给selenium服务器的，因此通过反编译***selenium-server-standalone-2.37.0.jar***得到其JAVA源码

> org.openqa.selenium.server.SeleniumServer，处理了有关GET/POST请求部分

> org.openqa.selenium.server.SeleniumDriverResourceHandler，中有关于不同的command获取的处理(/session/:sessionId/command)，其中包含了对screenshot命令的处理

> org.openqa.selenium.server.commands.CaptureScreenshotToStringCommand

```Java
package org.openqa.selenium.server.commands;

import java.awt.Rectangle;
import java.awt.Robot;
import java.awt.Toolkit;
import java.awt.image.BufferedImage;
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.TimeoutException;
import java.util.logging.Level;
import java.util.logging.Logger;
import javax.imageio.ImageIO;
import org.openqa.selenium.internal.Base64Encoder;
import org.openqa.selenium.server.RobotRetriever;

public class CaptureScreenshotToStringCommand
{
  public static final String ID = "captureScreenshotToString";
  private static final Logger log = Logger.getLogger(CaptureScreenshotToStringCommand.class
    .getName());

  public String execute()
  {
    try
    {
      return "OK," + captureAndEncodeSystemScreenshot();
    } catch (Exception e) {
      log.log(Level.SEVERE, "Problem capturing a screenshot to string", e);
      return "ERROR: Problem capturing a screenshot to string: " + e.getMessage();
    }
  }

  public String captureAndEncodeSystemScreenshot()
    throws InterruptedException, ExecutionException, TimeoutException, IOException
  {
    Robot robot = RobotRetriever.getRobot();
    Rectangle captureSize = new Rectangle(Toolkit.getDefaultToolkit().getScreenSize());
    BufferedImage bufferedImage = robot.createScreenCapture(captureSize);
    ByteArrayOutputStream outStream = new ByteArrayOutputStream();
    ImageIO.write(bufferedImage, "png", outStream);

    return new Base64Encoder().encode(outStream.toByteArray()); // 这里返回了Base64编码，印证了之前得到返回值
  }
}
```

> org.openqa.selenium.server.RobotRetriever

```Java
package org.openqa.selenium.server;

import java.awt.Robot;
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.FutureTask;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.TimeoutException;
import java.util.logging.Logger;

public class RobotRetriever
{
  private static final Logger log = Logger.getLogger(RobotRetriever.class.getName());
  private static Robot robot;

  public static synchronized Robot getRobot()
    throws InterruptedException, ExecutionException, TimeoutException
  {
    if (robot != null) {
      return robot;
    }
    FutureTask robotRetriever = new FutureTask(new Retriever(null));
    log.info("Creating Robot");
    Thread retrieverThread = new Thread(robotRetriever, "robotRetriever");
    retrieverThread.start();
    robot = (Robot)robotRetriever.get(10L, TimeUnit.SECONDS);

    return robot;
  }

  private static class Retriever
    implements Callable<Robot>
  {
    public Robot call()
      throws Exception
    {
      return new Robot();
    }
  }
}
```

到此为止我们已经了解到Webdrivercss/Webdriverio的截图原理，最底层是使用***java.awt.Robot.createScreenCapture***截图，根本原理还是对桌面截图！解密完毕，其他待续~