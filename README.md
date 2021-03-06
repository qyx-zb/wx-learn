# wx-learn
微信小程序相关

前言：我从事小程序开发有两年的时间，原生的开发，多个平台，有微信的，百度的，快应用，字节跳动等，两年的时间，也遇到过各种各样的问题，一直想着总结一下，但是总是拖延，作为一名前端开发人员，我们首先要学会如何开发，但是当你具有一定的开发经验之后，不妨深入思考，毕竟如果你还停留在应用开发层面，那你就OUT了，今天带大家探讨下小程序框架本身底层实现的一些技术细节，让我们从小程序的运行机制来深度了解小程序。

关于微信小程序：    
微信小程序底层原理  
开发流程    
流行的框架  
实战经验总结


## 微信小程序底层原理  

我们都知道页面渲染的方式主要有三种:
1. web渲染
2. Native原生渲染
3. web与Native两者掺杂，即Hybrid渲染。

而小程序的呈现形式为第三种。

#### 与h5页面的区别
那么，小程序和普通的 h5 页面到底有什么区别呢？
- 运行环境：小程序基于浏览器内核重构的内置解析器，而 h5 的宿主环境是浏览器。所以小程序中没有 DOM 和 BOM 的相关 API ， jQuery 和一些 NPM 包都不能在小程序中使用；
- 系统权限：小程序能获得更多的系统权限，如网络通信状态、数据缓存能力等；
- 渲染机制：小程序的逻辑层和渲染层是分开的，而 h5 页面 UI 渲染跟 JavaScript 的脚本执行都在一个单线程中，互斥。所以 h5 页面中长时间的脚本运行可能会导致页面失去响应。

其实，小程序开发过程中我们面对的是 iOS 和 Android 微信客户端和辅助开发的小程序开发者工具。根据官方文档，这三大运行环境也是有所区别的：

运行环境|逻辑层|渲染层 
---|---|---
iOS|JavaScriptCore|WKWebView
Android|	X5 JSCore|	X5浏览器
小程序开发者工具|	NWJS|	Chrome WebView

所以微信小程序介于 web 端和原生 App 之间，能够丰富调用功能接口，同时又跨平台。



#### 小程序的架构
##### 双线程模型
小程序的渲染层和逻辑层分别由2个线程管理：

- 渲染层：界面渲染相关的任务全都在 WebView 线程里执行。
- 逻辑层：采用 JsCore 线程运行JS脚本。

 视图层和逻辑层通过系统层的 WeixinJsBridage 进行通信：逻辑层把数据变化通知到视图层，触发视图层页面更新，视图层把触发的事件通知到逻辑层进行业务处理。

一个小程序存在多个界面，所以渲染层存在多个webview线程。逻辑层和渲染层的通信会由Native（微信客户端）做中转，逻辑层发送网络请求也会经由Native转发。

`小程序的UI视图和逻辑处理是用多个webview实现的，逻辑处理的JS代码全部加载到一个Webview里面，称之为AppService，整个小程序只有一个，并且整个生命周期常驻内存，而所有的视图（wxml和wxss）都是单独的Webview来承载，称之为AppView。所以一个小程序打开至少就会有2个webview进程，正式因为每个视图都是一个独立的webview进程，考虑到性能消耗，小程序不允许打开超过5个层级的页面，当然同是也是为了体验更好`


（页面渲染的具体流程是：在渲染层，宿主环境会把 WXML 转化成对应的 JS 对象，在逻辑层发生数据变更的时候，我们需要通过宿主环境提供的 setData 方法把数据从逻辑层传递到渲染层，再经过对比前后差异，把差异应用在原来的Dom树上，渲染出正确的UI界面）

![image](https://raw.githubusercontent.com/qyx-zb/wx-learn/master/8.png)

双线程模型是小程序框架与业界大多数前端 Web 框架不同之处。基于这个模型，可以更好地管控以及提供更安全的环境。缺点是带来了无处不在的异步问题（任何数据传递都是线程间的通信，也就是都会有一定的延时），不过小程序在框架层面已经封装好了异步带来的时序问题。

##### 组件系统
我们知道小程序是有自己的组件的，这些基本组件就是基于 Exparser 框架。

小程序中，所有节点树相关的操作都依赖于 Exparser ，包括 WXML 到页面最终节点树的构建、 CreateSelectorQuery 调用和自定义组件特性等。在WAWebview.js里有个对象叫exparser，它完整的实现小程序里的组件，看具体的实现方式，思路上跟w3c的web components规范神似，但是具体实现上是不一样的，我们使用的所有组件，都会被提前注册好，在Webview里渲染的时候进行替换组装。 
exparser有个核心方法： 
- regiisterBehavior: 注册组件的一些基础行为，供组件继承
- registerElement：注册组件，跟我们交互接口主要是属性和事件

组件触发事件（带上webviewID），调用WeixinJSBridge的接口，publish到native，然后native再分发到AppService层指定webviewID的Page注册事件处理方法。

微信小程序也支持自定义组件，用法和组件间通信类似于 Vue 。

`Exparser是微信小程序的组件组织框架，内置在小程序基础库中，为小程序的各种组件提供基础支持。小程序内所有组件，包括内置组件和自定义组件，都有Exparser组织管理。`


##### 原生组件
在内置组件中，有一些组件并不完全在 Exparser 的渲染体系下，而是由客户端原生参与组件的渲染。比如说 Map 组件,canvas组件，video组件等。它们渲染的层级比在 WebView 层渲染的普通组件要高。

引入原生组件的优点是：
- 绕过 setData、数据通信和重渲染流程，使渲染性能更好。
- 扩展 Web 的能力。比如像输入框组件（input, textarea）有更好地控制键盘的能力。
体验更好，同时也减轻 WebView 的渲染工作。比如像地图组件（map）这类较复杂的组件，其渲染工作不占用 WebView 线程，而交给更高效的客户端原生处理。
 

原生组件的渲染过程：
1. 组件被创建，包括组件属性会依次赋值。
2. 组件被插入到 DOM 树里，浏览器内核会立即计算布局，此时我们可以读取出组件相对页面的位置（x, y坐标）、宽高。
组件通知客户端，客户端在相同的位置上，根据宽高插入一块原生区域，之后客户端就在这块区域渲染界面。
3. 当位置或宽高发生变化时，组件会通知客户端做相应的调整

##### 运行机制
1. **启动**
- 热启动：：假如用户已经打开过某小程序，然后在一定时间内再次打开该小程序，此时无需重新启动，只需将后台态的小程序切换到前台，这个过程就是热启动；
- 冷启动：用户首次打开或小程序被微信主动销毁后再次打开的情况，此时小程序需要重新加载启动，即冷启动。
![image](https://raw.githubusercontent.com/qyx-zb/wx-learn/master/9.png)

2. **销毁**         
只有当小程序进入后台一定时间，或者系统资源占用过高，才会被真正的销毁。

##### 更新机制

开发者在后台发布新版本之后，无法立刻影响到所有现网用户，但最差情况下，也在发布之后 24 小时之内下发新版本信息到用户。

小程序每次冷启动时，都会检查是否有更新版本，如果发现有新版本，将会异步下载新版本的代码包，并同时用客户端本地的包进行启动，即新版本的小程序需要等下一次冷启动才会应用上。

所以如果想让用户使用最新版本的小程序，可以利用 wx.getUpdateManager 做个检查更新的功能：
```
checkNewVersion() {
    const updateManager = wx.getUpdateManager();
    updateManager.onCheckForUpdate((res) => {
      console.log('hasUpdate', res.hasUpdate);
      // 请求完新版本信息的回调
      if (res.hasUpdate) {
        updateManager.onUpdateReady(() => {
          this.setData({
            hasNewVersion: true
          });
        });
      }
    });
  }
```


###### evaluate Javascript

视图层和逻辑层的数据传输，实际上通过两边提供的evaluateJavascript实现。即用户传输的数据，需要将其转换为字符串形式传递，同时把转换后的数据内容拼接成一份JS脚本，在通过JS脚本的形式传递到两边独立环境。

因为evaluateJavascript的执行会受很多方面的影响，数据到达视图层并不是实时的。随意我们的setData函数将数据从逻辑层发送到视图层，是异步的。


#### 模板数据绑定方案
1. 解析语法生成AST
2. 根据AST结果生成DOM
3. 将数据绑定更新至模板

最容易引发性能问题的主要是第三点，而关于数据更新的解决方案，React首先提出了虚拟DOM的设计，而现在也基本被大部分框架吸收，小程序也不例外。

#### 虚拟 DOM 机制 virtual Dom

用JS对象模拟DOM树 -> 比较两个DOM树 -> 比较两个DOM树的差异 -> 把差异应用到真正的DOM树上

1. 在渲染层把WXML转化成对应的JS对象
2. 在逻辑层发生数据变更的时候，通过宿主环境提供的setData方法把数据从逻辑层传递到Native，再转发到渲染层
3. 经过对比前后差异，把差异应用在原来的DOM树上，更新界面

#### 小程序的基础库

小程序的基础库是JavaScript编写的，它可以被注入到渲染层和逻辑层运行。主要用于：

在渲染层，提供各类组件来组件页面的元素

在逻辑层，提供各种API来处理各种元素。

处理数据绑定、组件系统、事件系统、通信系统等一系列框架逻辑

 

小程序的渲染层和逻辑层是两个线程管理，两个线程各自注入了基础库。

小程序的基础库不会打包在小程序的代码中，它会被提前内置在微信客户端。这样可以：

降低业务小程序的代码包大小

可以单独修复基础库中的Bug，无需修改到业务小程序的代码包


#### 双线程的渲染机制
双线程的渲染，其实是结合了前面的一系列机制。
1. 通过模板数据绑定和虚拟DOM机制，小程序提供了带有数据绑定语法的DSL，渲染层来描述页面结构。

```html
<view> {{ message }} </view> 
<view wx:if="{{condition}}"> </view> 
<checkbox checked="{{false}}"> </checkbox>
```

2. 小程序在逻辑层提供了设置页面数据的api
```
this.setData({
    key : value
});
```
3. 逻辑层需要更改页面时，只要把修改后的data通过setData传到渲染层。

传输的数据，会转换为字符串形式传输，故应避免传递大量数据。
4. 渲染层会根据渲染机制重新生成虚拟DOM树，并更新到对应的DOM树上，引起界面变化。

注意：不能同时打开超过5个窗口，打包文件不能大于1M，dom对象不能大于16000个等，这些都是为了保证更好的体验，微信小程序的基础底层架构大概就这么多，大家可以花时间多研究下原理。


## 微信小程序的开发

小程序是基于web规范的，传统的web采用的是html,css,js等进行开发，微信小程序依然是这个套路，只不过微信的官方给它换了个名字WXML,WXSS

#### 小程序的开发前期准备
1. 首先到[小程序注册页](https://mp.weixin.qq.com/wxopen/waregister?action=step1)注册一个账号，获得appid
2. 小程序的[开发文档](https://developers.weixin.qq.com/miniprogram/dev/framework/)，下载小程序的开发者工具

#### 小程序目录结构
![image](https://raw.githubusercontent.com/qyx-zb/wx-learn/master/10.png)

**一个完整的小程序主要由以下几部分组成：**  
- 一个入口文件：app.js
- 一个全局样式：app.wxss
- 一个全局配置：app.json
- 页面：pages下，每个页面再按文件夹划分，每个页面4个文件，页面里面又分：
    - 视图：wxml，wxss
    - 逻辑：js，json（页面配置，不是必须）
    
注：在app.json中可以配置页面，列表的第一个页面便是首页，在pages中也可以引用组建，需要在json文件中配置对应的页面地址,如下图
![image](https://raw.githubusercontent.com/qyx-zb/wx-learn/master/11.png)

#### 小程序的开发调试和打包上线
- 开发者工具有一个调试器功能，使用之后和普通的web页面无异。
![image](https://raw.githubusercontent.com/qyx-zb/wx-learn/master/12.png)
- 开发者工具有一个预览功能，点击可以生成一个二维码，通过微信扫码预览，记得打开调试：
![image](https://raw.githubusercontent.com/qyx-zb/wx-learn/master/13.png)
测试预览的时候，给相关的测试人员添加体验权限，将我们本地的代码上传到管理平台，就可以获得永久的体验码，开发码的有效时间是半个小时。  
- 发布的时候将不检验合法域名去掉，并在服务器中配置域名
![image](https://raw.githubusercontent.com/qyx-zb/wx-learn/master/14.png)
![image](https://raw.githubusercontent.com/qyx-zb/wx-learn/master/15.png)
- 提交审核，审核通过之后一键发布即可
![image](https://raw.githubusercontent.com/qyx-zb/wx-learn/master/16.png)

到这里我们小程序的一个基本开发流程就结束了。



## 微信的登录授权

参考一张微信官方提供的登录流程图辅助理解：
![](https://res.wx.qq.com/wxdoc/dist/assets/img/api-login.2fcc9f35.jpg)
![image.png](https://raw.githubusercontent.com/qyx-zb/wx-learn/master/1.png)

由上图可知,首先通过wx.login获取登录code(登录校验码)，然后请求接口将code发送到开发者服务器，凭借用户的Appid和code从微信服务器获取session_key(本次会话密钥)和openid，在本地服务器根据这两个信息定义用户的登录状态,并且将用户的登录状态返回到界面,这就是一次完整的用户授权过程。

##### 如何触发微信的授权？
有两种方式可以获取用户的基本信息，包括encryptedData和iv这样的敏感信息，一种是微信官方提供的一个API-----wx.getUserInfo，另外一种方式则是通过button组件，通过设置组件的属性`open-type='getUserInfo'`，然后在bindgetuserinfo事件返回值e.detail中取到用户的信息。

**微信登录的流程简单梳理：**
1. 通过wx.login()获取登录凭证code。
2. 将code发送到自己的服务器，服务器将登录凭证发送到微信的服务器上换取openid和session_key。
3. 通过button组件的open-type="getUserInfo", 获取用户信息。
4. 将获取到的用户信息和openid传递到自己的服务器。
5. 利用用户提交的信息在自己的服务器上注册用户账号(等等...)。
6. 将注册之后的信息返回给微信小程序。
7. 将注册信息保存起来以便以后使用。


描述：小程序端登录操作成功后，调用服务端接口生成session_key，服务器存储openId session_key，同时将openid 或自定义id  返回给客户端

请求参数:
code://用来获取sessionkey,进行解密      
返回参数:
openId或自定义id
手机号信息解密接口      
描述: 小程序将获得的加密信息传给服务器，由服务器根据（openid或自定义id）查询sessionkey 后进行解密操作，返回明文信息    
请求参数：  
openId或自定义id    
encryptedData     电话加密信息  
iv                   iv值   
返回参数：
phoneNumber    手机号


### 企业微信接入小程序
1. 后台设置：https://work.weixin.qq.com/help?person_id=1&doc_id=13104

 2. 文档：https://work.weixin.qq.com/api/doc#90000/90136/90289

### 列表上拉分页
注意点：
- scroll-view 
- 节流

先贴一段我项目中的配置代码：
```
<scroll-view scroll-y="{{isScroll}}" lower-threshold='600' bindscrolltolower="{{isLower ? 'bindscrolltolower' : ''}}" bindscroll="scroll">
</scroll-view>
```
这里有很多属性，我先说一下我这里用到的，其他的属性大家可以去看官方文档：

首先官方说`使用竖向滚动时，需要给scroll-view一个固定高度，通过 WXSS 设置 height`，但是我们的列表是不定高的，所以我们没有办法去给出一个定高，解决办法就是给page设置一个高，然后设置scroll-view的高为100%。  
`scroll-y`:允许纵向滚动     
`bindscroll`:滚动时触发，event.detail = {scrollLeft, scrollTop, scrollHeight, scrollWidth, deltaX, deltaY}     
`bindscrolltolower`:滚动到底部/右边时触发        
`lower-threshold`:距底部/右边多远时，触发 scrolltolower 事件     

### 自定义头部导航navigationStyle
效果图：
![image.png](https://raw.githubusercontent.com/qyx-zb/wx-learn/master/2.png)

代码实现：
注意点：
- getCurrentPages（）获取当前页面栈。数组中第一个元素为首页，最后一个元素为当前页面。

我们可以把这个功能写成一个组件：
![image.png](https://raw.githubusercontent.com/qyx-zb/wx-learn/master/3.png)
```
//custom-title.js

/**
 * 自定义导航栏
 * 用法：
 * title：小程序页面标题
 * avatarUrl：用户头像
 * isShowHeadImg：是否展示用户头像
 * backgroundColor：背景颜色
 * 全局提供的值
 * app.globalData.titleHeight,自定义导航栏的高度
 * app.globalData.pageContentHeight,内容区和页面的比例
 * app.globalData.prevPage 上一个页面
**/
const app = getApp();
Component({
    properties: {
    //小程序页面的表头
        title: {
            type: String,
            value: ''
        },
        avatarUrl:{
            type: String,
            value: ''
        },
        isShowHeadImg:{
            type: Boolean,
            value: false
        },
        backgroundColor:{
            type: String,
            value: '#fff'
        },
        isHiddenScroll:{
            type: Boolean,
            value: false
        },
        from:{
            type: String,
            value: ''
        }
    },

    data: {
        statusBarHeight: 0,
        titleBarHeight: 0,
        showIcon:false
    },

    ready: function () {
        let showIcon = getCurrentPages().length > 1 ? true:false;
        this.setData({showIcon});
        if (app.globalData && app.globalData.statusBarHeight && app.globalData.titleBarHeight) {
            this.setData({
                statusBarHeight: app.globalData.statusBarHeight,
                titleBarHeight: app.globalData.titleBarHeight
            });
        } else {
            let that = this;
            wx.getSystemInfo({
                success: function (res) {
                    if (!app.globalData) {
                        app.globalData = {};
                    }
                    if (res.model.indexOf('iPhone') !== -1) {
                        app.globalData.titleBarHeight = 44;
                    } else {
                        app.globalData.titleBarHeight = 48;
                    }
                    app.globalData.statusBarHeight = res.statusBarHeight;
                    that.setData({
                        statusBarHeight: app.globalData.statusBarHeight,
                        titleBarHeight: app.globalData.titleBarHeight
                    });
                },
                failure() {
                    that.setData({
                        statusBarHeight: 0,
                        titleBarHeight: 0
                    });
                }
            });
        }
    },

    methods: {
        headerBack() {
            let pages = getCurrentPages();
            let prevPage = pages[pages.length - 1].__route__; // 前一个页面
            app.globalData.prevPage = prevPage;
            wx.navigateBack({
                delta: 1,
                fail(e) {
                    wx.switchTab({
                        url: '/pages/index/index'
                    });
                }
            });
        },
        getuserinfo(res){
            this.triggerEvent('getuserinfo', res.detail);
        }
    }
});



//custom-title.wxml

<view style="height:{{titleBarHeight}}px;padding-top:{{statusBarHeight}}px;" >
    <view class="header {{from?'border-bottom':''}}" style="height:{{titleBarHeight}}px;padding-top:{{statusBarHeight}}px;background: {{backgroundColor}};">
        <view class="title-bar" bindtap="headerBack" wx:if="{{showIcon&&!isShowHeadImg}}">
            <view class="back">
                <image src="http://c3.xinstatic.com/c/20190506/1127/5ccfa9a005b2a874572.png"></image>
            </view>
        </view>
        <view wx:if="{{isShowHeadImg}}" class="userInfo-avatar">
                <image src="{{avatarUrl?avatarUrl:'http://c4.xinstatic.com/c/20190227/1034/5c75f722444a7504463.png'}}" class="avatar"></image>
                <button class="avatar" open-type="{{avatarUrl?'':'getUserInfo'}}" bindgetuserinfo="{{avatarUrl?'':'getuserinfo'}}"></button>
            </view>
        <view class="header-title">{{title}}</view>
    </view>
</view>
<view class="hiddenScrollBar" wx:if="{{isHiddenScroll}}" style="background: {{backgroundColor}};"></view>



//custom-title.wxss
.border-bottom {
  border-bottom: 1px solid rgba(148, 155, 164, 0.1);
}

.header {
  display: flex;
  align-items: center;
  position: fixed;
  top: 0;
  left: 0;
  width: 750rpx;
  background-color: rgba(255, 255, 255, 0);
  z-index: 99999;
}

.header .back {
  width: 18rpx;
  height: 32rpx;
  display: flex;
  align-items: center;
  /* justify-content: center; */
}

.header .title-bar {
  display: flex;
  align-items: center;
  justify-content: flex-start;
  width: 60rpx;
  height: 60rpx;
  padding-left: 30rpx;
}

.header .title-bar image {
  display: inline-block;
  width: 18rpx;
  height: 32rpx;
  background: transparent;
  vertical-align: top;
}

.header .header-title {
  position: absolute;
  left: 50%;
  font-size: 34rpx;
  color: #1B1B1B;
  font-weight: bold;
  transform: translateX(-50%);
}

.userInfo-avatar {
  position: relative;
  width: 80rpx;
  height: 80rpx;
  margin-left: 40rpx;
}

.userInfo-avatar .avatar {
  width: 80rpx;
  height: 80rpx;
  border-radius: 50%;
}

.userInfo-avatar button {
  position: absolute;
  top: 0;
  opacity: 0;
}

.hiddenScrollBar {
  position: fixed;
  top: 0;
  right: 5rpx;
  width: 12rpx;
  height: 100%;
  z-index: 999999;
}





//custom-title.json

{
    "component": true
}

```
app.js
```
wx.getSystemInfo({
    success: (res) => {
        if (res.model.indexOf('iPhone') !== -1) {
            this.globalData.titleBarHeight = 44;
        } else {
            this.globalData.titleBarHeight = 48;
        }
        this.globalData.statusBarHeight = res.statusBarHeight;
        this.globalData.screenHeight = res.screenHeight;
        this.globalData.titleHeight = res.statusBarHeight + this.globalData.titleBarHeight;
        this.globalData.pageContentHeight = (1-this.globalData.titleHeight/this.globalData.screenHeight)*100;
    }
});
```
使用的页面设置：
```
//wxml

<custom-title title="{{title}}" backgroundColor="#fcfcfc" isHiddenScroll="true" avatarUrl="{{avatarUrl}}" isShowHeadImg="{{true}}" bind:getuserinfo="getuserinfo"/>
<scroll-view bindscrolltolower = 'scrolltolower' scroll-y class="scroll" style="height:{{pageContentHeight}}%">
</<scroll-view>

//js
const app = getApp();
data:{
    titleHeight: app.globalData.titleHeight,
    pageContentHeight:app.globalData.pageContentHeight,
}

//json 

"navigationStyle":"custom"
```
### canvas，video等原生组件
![image.png](https://raw.githubusercontent.com/qyx-zb/wx-learn/master/4.png)
![image.png](https://raw.githubusercontent.com/qyx-zb/wx-learn/master/5.png)

当时在这里遇到很多问题，等我以后慢慢回想起来补充，这里主要想说一个原生组件的问题。

图一是一个使用canvas组件+30张图片结合“精密”的计算实现的一个VR效果的功能组件，记得封装这个组件的时候差点没把我整疯，哎～～～。图二是一个视频组件，但是想要检测网络状况给用户一个提示，这个功能百度小程序官方有提供，但是微信小程序没有，这是我最写的，这两个组件都带来了一个共同的问题，那就是原生组件的层级太高了，想要在原生组件上放置这些icon等元素，则需要在组件最外层包裹一个<cover-view></cover-view>，再说使用这个组件后遇到的其他问题，那就是当前页面如果有弹层的话会被遮挡，这时候就要想办法隐藏掉这个组件。

### IM&分享功能
通过设置button组件的open-type,IM功能的话需要在公众平台配置接口
### 双边滑动的一个slider组建（代码之后整理补齐）
![image.png](https://raw.githubusercontent.com/qyx-zb/wx-learn/master/6.png)
### 列表的一个删除效果（代码之后整理补齐）
![image.png](https://raw.githubusercontent.com/qyx-zb/wx-learn/master/7.png)




### 记录微信小程序开发中曾遇到的bug，以免以后在出现！

1. 给元素的标签上绑定数据时，名字最好用英文小写，不要用驼峰。因为绑定的数据在事件对象e.currentTarget.dataset里的属性都是小写。eg:
```
//wxml里：
<view data-personName="yangsen" bindtap="showname"></view>

//js里 ： 
showname:function(e){
  console.log(e.currentTarget.dataset)//输出的是{personname:"yangsen"}（这时用e.currentTarget.dataset.personName就拿不到数据了）

}
```
2. 微信小程序的video组件，如果添加了属性page-gesture="true"，在ios中，快进，后退或者暂停视频，页面会在30s内无法正常滑动，Android正常。

3. 瑕疵组件的swiper部分使用if指令会导致组件重新渲染因此‘记忆功能’不能实现，因此show指令，但show指令又会导致页面内报错，这种情况下页面内的显示部分使用三元运算符可以避免报错
```
如：{{featureFlawArr[f_c_index].flawIndex}}写法改为{{featureFlawArr[f_c_index] ? featureFlawArr[f_c_index].flawIndex : ''}}会避免页面报错
```
4. 微信小程序的组件    
scroll-view，在Android和ios中横向滑动的时候，表现不一样，ios上不显示滚动条，Android会显示横向的滚动条。有时候我们不需要去显示横向滚动条，文档上并没有禁止横向滚动条的属性，我们必须通过设置css样式
```
:-webkit-scrollbar 滚动条整体部分。
```
隐藏显示的滚动条：
```
::-webkit-scrollbar
{
    width: 0;
    height: 0;
    color: transparent;
}
```
5. 小程序的scroll-view必须设置为绝对定位才会触发触底事件

6. 微信小程序中遮罩层的滚动穿透问题

- 如果弹出的遮罩层没有滚动事件，就直接在蒙层上加`catchtouchmove="preventdefault"`，小程序1.5.0后可以写上`capture-catch:touchmove="preventdefault"`   这样就可以解决问题了。

 - 如果弹出层有滚动事件，那么在弹出层出现的时候给底部的containerView加上一个class，当弹出层消失的时候移除。
```
<view class=”{{maskShow ? ‘noscroll’:’’}}”></view>

.noscroll{
    position:fixed;
    top:0;
    left:0;
    overflow:hidden;
    width:100%;
    height:100%;
}
```
这样的做法可以解决滚动穿透问题，但是每次点击之后，页面都会滚动到最顶部，所以我们需要动态的给top值，这时候可以利用onPageScroll事件来获取卷上去的top值。
```
//html

<view class="{{maskShow ? 'noscroll' : ''}}" style="top:{{needHeight}}px;background:#FF5925"></view>

//Js部分
data{
    needHeight: 0,
    maskShow:false
}

onPageScroll(e){//获取需要卷上去的top值，但是不可以在这里setData,因为onPageScroll的触发频率非常高，会影响性能，所以需要再一个事件中来保存这个状态，比如点击弹出蒙层的事件showMask

onPageScroll(e){
    this.data.scrollTop = e.scrollTop
 }

showMask(){ //点击弹出按钮，先保存卷上去的值，成功后加上class名，这时候页面状态可以被保存起来，不会回到最顶部
    this.setData({
      needHeight: -this.data.scrollTop
    },()=>{
        setTimeout(()=>{
            this.setData({
                maskShow: true,
            })
        },400)
    })
  },

closeMask() {   //点击关闭蒙层，滚动到固定定位的位值
    this.setData({
        maskShow: false
    })
    
    wx.pageScrollTo({
      scrollTop: -this.data.needHeight,
      duration: 0
    })
},
```
 
7. 微信小程序中关于`image加border-radius`不起作用的问题

在微信小程序中，我们遇到一个显示好友列表信息的时候，头像会先闪烁一下，先变方后变圆，虽然给<image/>的外层嵌套了一个<view/>，并且给外层设置了width:50rpx;height:50rpx;border-radius:50%;但是不起作用的，上网查看了一下，说是微信官方的问题，看了多个其他家的小程序也存在这个问题，为了解决这一问题，尝试了一下背景图的方法，发现效果不错。
```
 <view class="head-photo-box">
    <view class="head-photo" style="height:50rpx;width:50rpx;background:url({{item.avatarurl||'http://c2.xinstatic.com/f3/20180523/1758/5b053b5abb9a9533894.png'}});background-size: 100% 100%;"></view>
    <!-- <image class="head-photo avatarurl-item" src="{{item.avatarurl ? item.avatarurl : 'http://c2.xinstatic.com/f3/20180523/1758/5b053b5abb9a9533894.png'}}"></image> -->
</view>
//css
.head-photo-box{
    width: 50rpx;
    height: 50rpx;
    border-radius: 50%;
    overflow: hidden;
    background: url('http://c2.xinstatic.com/f3/20180523/1758/5b053b5abb9a9533894.png');
    z-index: 3rpx;
    margin-right: 20rpx;
    background-size: 100% 100%
}
```

