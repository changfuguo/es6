# 基于larkjs的react同构尝试

尝试了一周，参考了大量的项目，自己动手搞了一个伪脚手架，算是对自己这些天的一个总结吧

## 一、 前言 backbone和angular


之前的项目中分别用过backbone和angular；

backbone是接触的第一个mvc的框架，瞬间原来那种无组织无纪律的写法，通过一种规范化的表达得到了改善，目录结构明了；但由于约束相对较少，所以需要自己创建大量model和collection完成业务逻辑，对视图和路由也需要额外处理；其数据和代码的绑定，也只能通过model的set/get方式来触发，如下

```
（"change:sourceProperty, this.recalculateComputedProperty）

```

至于angular，其双向绑定的特性，能解放很大一部分get/set的工作；依赖注入，完成模块化开发和组件的交互；视图可以通过指令（Directive）的方式将**业务逻辑和视图**组装，以达到复用，而backbone是业务逻辑和模板在设计上是有明显的界限；通过controller实现逻辑与视图的交互；

当时用的angular1.x版本，存在的问题是，集成第三方代码时，需要纳入到ng的体系之中，比如涉及到set/set的组件，需要通过Directive二次封装；按需加载需要和第三方库相结合如requirejs等。
不过不得不说，angular应该是一个开创性的框架了，本身支持模块化，双向绑定的特性让我兴奋了几天
。这有一个关于backbone和ng的对比 [backbone-vs-angular](http://www.infoq.com/cn/articles/backbone-vs-angular/)

对于我而言，除了ng本身的双向绑定特性，指令我觉得算是真正的组件，我理解的组件至少是js/html的结合体，而不是将两者割裂。

而对于reactjs，集中在view层，其组件内聚业务逻辑、视图和样式三者，更符合我心中的组件化和模块化开发

###反思：前端如何选型？

洪荒时代，都是代码搬运工。不同代码片充斥在项目的一个个角落。
jquery及其插件生态解决了dom的查找和操作等问题，已经红遍全宇宙。

bootstrap这种以UI为主的框架（其实不太想称之为框架，因为只是解决了UI的东西）又解放了大部分的前端er写页面的工作，于是继续红在前端心中。

相比个人感觉extjs可以可以算作一套完整的框架，有基础的类库，有对应的UI，整个风格统一，代码规范。但是正是由于其太完善，包罗万象，用起来有点沉重，更加适合做MIS相关系统。

选型会考虑如下几方面

#### 1、模块化和组件化的实现程度

侠义的讲我理解模块至少是遵守一种规范，AMD也好CMD也好；因为只有有了模块，才能更好的支持业务的划分，解耦，更好处理依赖，以及在此基础之上的各种构建。对于模块的理解，我认为相似功能的抽象，至少应该内聚业务逻辑和视图两者，reactjs在这方面走的更远，将css也js化了。


#### 2、对应构建工具是否完善

为什么要有构建工具，模块代码被编写之后，在送到浏览器执行之前，需要进一步的合并，压缩，减少体积，减少请求，目的都是为了增加展现速度。

另外一个构建工具可以规范并行开发、调试、测试、部署以及上线的流程，说白了就是让你舒服高效的开发直至上线完成。

我用过的几个构架工具，r.js、grunt、gulp、webpack以及fis，各有优点。


 r.js是对应requirejs的构建工具，requirejs本身带我认识了模块化开发的流程，足够灵活。但是需要配置一大堆的config文件，本身并没有约束模块本身如何去开发。


grunt和gulp同时期的构建工具，相比grunt来说，gulp简单的API、灵活的任务处理方式（各种串行和并行）、以及丰富的插件，不得不说对前端人员来说简直是利器.

webpack是目前最火的单页类应用构建工具，讲究的是入口，以入口为主，将需要的东西统统打包，更适合SPA的。并且gulp中需要插件或者写任务的东西，在webpack里直接配置即可，能完成绝大部分的功能，加之其热更新功能，简直就是神器呀。唯一不足的是国外文档，配置太多，好多时候自己一个一个去试配置

fis是我用过的构建工具了感觉最爽的一个，所有的资源都是模块，打包的规则自由灵活，可以说gulp和webpack实现的fis都可以轻松实现。并且内置不同开发环境下的不同部署方式，参考[fis](fis.baidu.com)   ,  [fis、webpack比较](https://zhuanlan.zhihu.com/p/20933749)，与其说是一个构建工具，不如说是个完善的流程解决方案更合适。

#### 3、生态是否足够完善

这个不说了，既然都选了reactjs，也不用看在git上4w+的start，也不吹嘘它的性能如呵呵，如何可自豪的采用es6，也不说可以在服务端渲染直出，其实我更觉得reactjs实现模块开发的方式：不用分开写js逻辑以及html还有样式了，对前端来说真正的模块是离不开各种div标签和样式的，如何优雅的组织三者到一起，我更喜欢reactjs的方式

至于在ng之间如何取舍，可以参考这个文章[choosing-between-react-vs-angular-2](http://react-etc.net/entry/choosing-between-react-vs-angular-2)

> React is a great technology and as a library it's ideal for introducing into old applications. The limited opinions React has on other technologies has cultivated a rich environment of supporting tools. And that is the problem, it's not very coherent or tangible for beginners.


>


> Angular 2 on the other hand continues on the path of being a framework. That makes it more of a product and easier to get into. The Angular 2 team has already done a lot of decisions for developers and as such is more uniform than anything that's "built with React".

## 二、 如何做

之前组里同学 [妈咪特卖](mami.baidu.com)基于reactjs+node的开发方式，相关文章[百度母婴技术团队—基于Reactjs实现webapp](https://github.com/my-fe/wiki/issues/1)


![技术架构](https://cloud.githubusercontent.com/assets/15227832/10684959/0657af3e-7986-11e5-92b7-c4b9164c0add.png)

后端的技术架构

![后端架构](https://cloud.githubusercontent.com/assets/15227832/10685018/e628db9c-7986-11e5-93bc-2dc1f0b871fd.png)


从上面可以看出在实践reactjs过程当中，结合自身的业务，实现了前后端同构，是一个很好的实践，不过当时没参与，接下来我就梳理下自己的思路，对于react本身我也是刚刚入门，demo是照搬的，主要的技术点如下

### 1、react生态

. redux:单向数据流管理

. react-redux:react和redux结合

. react-router:路由管理

. redux-thunk:可以让你愉快的dispatch，官方解释如下

Redux Thunk middleware allows you to write action creators that return a function instead of an action. The thunk can be used to delay the dispatch of an action, or to dispatch only if a certain condition is met. The inner function receives the store methods dispatch and getState as parameters.

简单解释可以返回一个action处理的函数非单纯的action，延迟dispatch或者有条件的dispatch，大白话就是异步dispatch

.react-dom: 解决服务端直出时模拟render时的上下文出错用的 


上面是react前端涉及的几个组件，有人说store树会随着项目越来越大变的臃肿，有人提出immutable.js扁平化管理，本次暂不做尝试

### 2、node生态

node现成的框架express、KOA已经集成了开发所需要的大部分功能，中间价相对比较完善。和[妈咪特卖](mami.baidu.com)的一样，采用组内larkjs，其在koa的基础上，划分出来了con

controller以及pageservice、dataservice以及daoservice，目录结构清晰，约定的路由及可复用的model层极大方便了开发效率，目前支持es6的generator

![](https://raw.githubusercontent.com/changfuguo/es6/master/2016-06/2016-06-14-reactjs-isomorph-practice/node-art.png)


### 3、服务端渲染

为了更好的seo，进行服务端直出。服务端直出在之前的项目里对css以及其他静态资源的打包是通过gulp实现的，但是为了更好的利用webpack的HMR模块，这里对css走webpack的loader；这样引申出来一个问题，***服务端直出的js由于编译后不再和静态资源处于一个目录，那么client端require的css，则会找不到module***，运行server端的脚本会报如下错误：（当然，你绝对可以通过gulp来打包css及其它静态资源，没有什么不好，仅仅是丢失了HMR特性而已）

![](https://raw.githubusercontent.com/changfuguo/es6/master/2016-06/2016-06-14-reactjs-isomorph-practice/can-not-find-module.png)


查阅了好多资料，找到一个[iamdustan](https://twitter.com/iamdustan/status/577561601353465856),[Backend-Apps-with-Webpack](http://jlongster.com/Backend-Apps-with-Webpack--Part-I)给出的解决办法如下,在webpack的配置中增加如下插件，以忽略样式文件，

```
plugins:[
        	new webpack.IgnorePlugin(/\.(css|scss)$/),
        	new webpack.NormalModuleReplacementPlugin(/\.(css|scss)$/, 'node-noop')
        ],
        
```
但是实验之后无果，依然报错。

最终的解决办法采用这个哥们的办法[isomorphic-react-in-real-life](https://reactjsnews.com/isomorphic-react-in-real-life):在webpack打包的时候传入一些指示是服务端还是浏览器端执行的变量，用require运行时加载，如上图中所示：

```
if(process.env.__BROWSER__) {
	require('./index.scss')
}
```
在client的webpack打包配置如下:

```
new webpack.DefinePlugin({
	            "process.env": {
	            	'NODE_ENV' : JSON.stringify('development'),
	            	'__BROWSER__': JSON.stringify(true)
	            }
	        })
	        
```

而在服务端脚本配置不处理__BROWSER__变量，注意这里**require**是运行时加载的，所以该分之在服务端不会执行，避免了找不到模块的错误；

但是这样一来，你就不能用[css modules](https://github.com/css-modules/css-modules)的一些特性了，至少是不能全盘使用了，具体可参考上述连接

好，服务端问题over了，来看下HMR

### 4、HMR（hot module replacement）

这个特性之前是在研究react-native时发现的，当时就觉得惊呆了，有木有！无丢失状态、无需手动刷新，瞬间觉得效率提高了有没有；查阅资料的时候发现在reactjs有过几种热更新的方式

.webpack-dev-server: 

	参考 [webpack-dev-server](http://webpack.github.io/docs/webpack-dev-server.html),[Webpack-dev-server结合后端服务器的热替换配置](http://www.jianshu.com/p/8adf4c2bfa51) ,需要配合 [react-hot-loader](http://gaearon.github.io/react-hot-loader/getstarted/)实现

 	
 **缺点：这种方式需要开双服务！！！**
 
 
.webpack-hot-middleware + webpack-dev-middleware 

直接集成到node的后台服务，以中间价 形式接入hot的请求，配置如下
			

```
if(__DEV__) {

    let webpackConfig = null;
    webpackConfig = require('./webpack.config.dev')(config);
    const compiler = webpack(webpackConfig)
    app.use(webpackDevMiddleware(compiler, { 
      stats: {
        color: true
      },
      inline:true,
      lazy:false,
      hot: true,
      noInfo: true, 
      publicPath: webpackConfig.output.publicPath,
      headers: {'Access-Control-Allow-Origin': '*'},
      //contentBase: "http://" + config.server_host + ':' + config.server_port,
      quiet: true
    }))
    app.use(webpackHotMiddleware(compiler),{
    	heartbeat: 20 * 1000
    });
} 

```

这种方式可以将HMR以中间件的形式插入到当前dev的服务中去，爽歪歪


### 5、服务端开发

尝鲜作祟，现在的larkjs未能全面支持es6特性，采用babel编译为了能使用到这些新特性和client保持一致；所以服务端的代码也需要编译，这个编译过程放到gulp中编译；由于lark的对mvc代码在加载的时候已经将上下文保存起来，要实现服务端的热更新，只能watch完之后只能重启服务，**这样做导致前端页面的数据状态丢失，需要刷新页面**。


  这里注意的是，larkjs自动加载models和controller中的文件，并用之前的module.exports导出，这里如果用es的export default 形式导出的话，会报找不到模块哟，所以导出的还沿用之前的写法
  
  对于服务端渲染的处理需要注意以下几个方面
  
1) 处理除js以外的静态资源，用__BROWSER__变量解决module报错

2）文件变更之后用gulp的watch监控，将重新打包的代码server.js放到node服务中编译重新加载，这里服务端reactRender的代码：

```
'use strict';

import React from 'react'
import {renderToString} from 'react-dom/server'
import { match, RouterContext } from 'react-router'
import { Provider } from 'react-redux'
import routes from './routes'
import configureStore from './store/configureStore'


const render = function(stateData, ctx) {

	return new Promise((resolve, reject) =>{
		try{
			match({routes, location: ctx.originalUrl }, (err, redirect, props) => {
				const store = configureStore(stateData);
				const state = store.getState();
				let html = '';
				try{
					html += renderToString(
						<Provider store={store} key="provider">
							<RouterContext {...props}/>
						</Provider>
					);
				} catch(err) {
					console.log('render error:' + err)
				}
				console.log(html)
				if (err) {
					reject('server render error');
				} else {
					resolve({html, state});
				}
			});
		}catch(err){
			console.log(err)
		}
	});
	
}

exports.default = global.renderReact = render;

```

将renderReact挂到全局下，当不同路由进来结合react-router进行路由同构

3）当前端业务代码更改后，需要在服务端重新加载；此时之前的代码已经require过，需要在启动当前服务的时候进行监控编译后的server.js，将其从require.cache中删除，重新挂载新的代码，过程描述如下：

gulp.watch监控->文件更改->重新编译server.js->cp代码到当前服务中->node服务监控服务中server.js代码发生更改，删除require.cache中旧的文件->重新require新的server.js

server.js重新编译的代码如下

```
if(__DEV__) {

  	let chokidar = require('chokidar');
  	let watcher = chokidar.watch(['./lib/server.js']);
  	let debugpath = require('path');
		watcher.on('ready', () => {
	  	watcher.on('all', (event, path) =>{
		    console.log("Clearing /server/ module cache from server");
		    console.log(event + ':' + path)
		    var fullPath = debugpath.resolve(__dirname, path);
		    if(require.cache[fullPath] && fullPath.endsWith('lib/server.js')) {
		    	console.log(`exist ${fullPath}`)
		    	delete require.cache[fullPath];
		    	require(fullPath);
		    }
	  	});
	});
}
```

这个做法有一点美中不足,**serve.js重新编译和HMR重新分发应该是有个先后顺序，应该是先将编译后的server.js挂到node中去，再触发HMR的下发，这个我没仔细研究，两者顺序不能保证，一旦server.js先编译，没问题；一旦server.js后挂载，则当HMR下发代码的时候会报错，前后端渲染的HTML不一致***，报如下错误：

	React attempted to reuse markup in a container but the checksum was invalid. This generally means that you are using server rendering and the markup generated on the server was not what the client was expecting. React injected new markup to compensate which works but you have lost many of the benefits of server rendering. Instead, figure out why the markup being generated is different on the client or server:
	
4）服务端es7的async/await不能使用，lark的框架本身支持到yield，用async/await方法babel到es2015之后不兼容，555，所以保留lark本身的function *的写法

### 6、前后端路由同构

前后端路由同构实现起来比较容易，前端配置路由的时候需要配置成和后端路由一样，都是手动配置，写代码的时候注意就行了，这里采用的的是服务端给数据的方式到server.js中定义的reactRender方法，进行服务端渲染，

服务端的路由采取了，自动加载的办法，不过按照larkjs的实现方法有点ugly，larkjs的controller直接在module.exports中定义，参数为koa-router，当前counter.js的路由为文件名为基准，router.get('/getlist',function *(){}),对应的路由为'counter/getlist'

对lark的路由进行了简单的封装，如下的controller控制的路由我/counter/index对应下面ActionIndex函数，只不过为了衔接lark的路由控制，实现起来有点ugly

```
'use strict';

class CounterController extends BaseController{

	constructor(){
		super(true)
	}

	* ActionIndex(ctx) {
		return new Promise( (resolve, reject) =>{
			resolve({counter: 330})
		})
	}

	*Action(ctx) {
		return new Promise( (resolve, reject) =>{
			resolve({counter: 33000})
		})
	}
	
}

module.exports = function (router) {
	new CounterController().route(router)
}

```

这样做有一个缺点，**按照lark的路由定义方式，可以实现'/counter/:name'这种带参数的理由匹配**，因为我为了自动映射后端路由到前端路由，函数命名规则不能有':name'的命名，要是非要这么实现类似正则的路由匹配，也只能用CounterController.prototype['index/:name']的命名方式，个人认为不是号的方法，具体后面再研究，最次的是走配置的方法吧


### 7、开发和上线构建

这里开发环境方便调，在client端需要服务端配合HMR功能，dev环境下把watch到的代码直接生成到可访问的目的地。直出的server.js需要watch并在node端进行热替换。对于服务端的热更新，watch到之后直接重启吧

build的时候直接将代码build到dist里去，并打包。

上线的时候结合build.sh进行解压

代码戳这里[carmaintain](https://github.com/changfuguo/carmaintain)
## 三、后续

总结下出现的几个问题，后续需要继续优化的点

1）server.js需要解决摘除css的静态代码，否则报错

2) 优化，体积过大问题，目前的代码是个示例代码，打包的vendor压缩后200多k，业务代码几十K，继续优化

3）路由同构实现类似正则的路由，目前需要进一步实践，最差的就走配置吧

4）异步加载，也即[code splitting]()

```
require.ensure(["./firstScript.js"], function(require) {
});
```

***最后的一个总结是：不要痴迷于框架或者工具本身，好的框架或者工具对开发效率和简化流程是必要的。但是最核心的东西是你如何连接你擅长的工具到具体的业务中去。通用的东西如dom操作、cookie等通用的业务，更重要的是如何将具体业务以组件形式编写和组织起来，这才是前端的核心工作，工具只是更好的方便的让你写业务代码*
***
## 四、参考

1. [react-redux-starter-kit](https://github.com/davezuko/react-redux-starter-kit)

2. [react-redux-universal-hot-example
express
](https://github.com/erikras/react-redux-universal-hot-example
express)

3. [react-webpack-node](github.com/mxstbr/react-boilerplate)

4. [react-webpack-node](https://github.com/choonkending/react-webpack-node)

5. [为 Koa 框架封装 webpack-dev-middleware 中间件](http://www.tuicool.com/articles/MruEni)


6. [深入理解 react-router 路由系统
](https://zhuanlan.zhihu.com/p/20381597)


7. [mirror-web-isomorphic](https://github.com/CQUPTMirror/mirror-web-isomorphic/)

8. [webpack-your-bags](https://blog.madewithlove.be/post/webpack-your-bags/)

9. [reduce-your-bundle-js-file-size](https://lacke.mn/reduce-your-bundle-js-file-size/)

10. [React.js 2016 最佳实践](http://www.tuicool.com/articles/jiQnuqj)


11. [FIS3 、ROLLUP、webpack比较](https://zhuanlan.zhihu.com/p/20933749)

12. [后端同构三部曲](http://jlongster.com/Backend-Apps-with-Webpack--Part-III)


13. [isomorphic-react-in-real-life](https://reactjsnews.com/isomorphic-react-in-real-life)

14. [react-tutorial-cloning-yelp](https://www.fullstackreact.com/articles/react-tutorial-cloning-yelp/)

