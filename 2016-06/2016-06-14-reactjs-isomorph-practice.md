# 基于larkjs的react同构实践

尝试了一周，参考了大量的项目，自己动手搞了一个伪脚手架，算是对自己这些天的一个总结吧

## 一、 为何选择React


之前的项目中分别用过backbone和angular；

backbone是接触的第一个mvc的框架，感觉是很容易上手，原来那种无组织无纪律的写法，通过一种规范化的表达得到了改善，目录结构明了；但是写着写着发现要写大量的model和view，用了一段时间就放弃了；

至于angular，其双向绑定的特性，能解放很大一部分get/set的工作，于是满怀信心的决定好好研究一番；

后来发现不是那么回事，你要把之前的很多组件改成指令，以达到复用性和维护性的目的，这个不是一个小的工作量；

最要命的是，首次加载的时候N多个的model/controller，直接白屏好久；再后来引入了requirejs，可以按需加载部分代码；再后来升级版本解决了父子controller以及路由问题；

从使用者的角度来看，backbone和angular还是主要集中在解决js逻辑上，模块化更确切的说是js的模块化，样式和html的照样分开写，分开组织；

在当时的情况下对于开发、打包和部署的总体流程也没有总结过完善的方案，加之后来项目完事就暂告一段落了；

不过不得不说，angular应该是一个开创性的框架了，本身支持模块化，双向绑定的特性让我兴奋了几天


###反思：前端快速开发到底需要什么样的技能，如何选型？

洪荒时代，都是代码搬运工。不同代码片充斥在项目的一个个角落。
jquery及其插件生态决绝了dom的查找和操作等问题，已经红遍全宇宙。

bootstrap这种以UI为主的框架（其实不太想称之为框架，因为只是解决了UI的东西）又解放了大部分的前端er写页面的工作，于是继续红在前端心中。

相比个人感觉extjs可以可以算作一套完整的框架，有基础的类库，有对应的UI，整个风格统一，代码规范。但是正是由于其太完善，包罗万象，用起来有点沉重，更加适合做MIS相关系统。

选型会考虑如下几方面

#### 1、是否支持模块化开发。

侠义的讲我理解至少是遵守一种规范，AMD也好CMD也好；因为只有有了模块，才能更好的支持业务的划分，解耦，更好处理依赖，以及在此基础之上的各种构建。对于模块的理解，我认为相似功能的抽象，可以表现为若干的组件的组合也可以表现为一个代码片段，只要你能以一种明确的方式被其他模块引用即可


#### 2、对应构建工具是否完善

为什么要有构建工具，模块代码被编写之后，在送到浏览器执行之前，需要进一步的合并，压缩，这是常识，减少体积，减少请求，目的都是为了增加展现速度。

另外一个构建工具可以规范并行开发、调试、测试、部署以及上线的流程，说白了就是让你舒服高效的开发直至上线完成。

我用过的几个构架工具，r.js、grunt、gulp、webpack以及fis，各有优点。


 r.js是对应requirejs的构建工具，requirejs本身带我认识了模块化开发的流程，足够灵活。但是需要配置一大堆的config文件，本身并没有约束模块本身如何去开发。


grunt和gulp同时期的构建工具，相比grunt来说，gulp简单的API、灵活的任务处理方式（各种串行和并行）、以及丰富的插件，不得不说对前端人员来说编译利器

webpack最火的工具，讲究的是入口，以入口为主，将需要的东西统统打包，更适合SPA的。并且gulp中需要插件或者写任务的东西，在webpack里直接配置即可，并且能完成绝大部分的功能，并且还有热更新，简直就是神器呀。唯一不足的是国外文档，配置太多，好多时候自己一个一个去试配置

fis是我用过的构建工具了感觉最爽的一个，所有的资源都是模块，打包的规则自由灵活，可以说gulp和webpack实现的fis都可以轻松实现。并且内置不同开发环境下的不同部署方式，参考[fis](fis.baidu.com)   ,  [fis、webpack比较](https://zhuanlan.zhihu.com/p/20933749)

#### 3、生态是否足够完善

这个不说了，既然都选了reactjs，也不用看在git上4w+的start，也不吹嘘它的性能如呵呵，如何可自豪的采用es6，也不说可以在服务端渲染tame出，其实我更觉得reactjs实现模块开发的方式：不用分开写js逻辑以及html还有样式了，对前端来说真正的模块是离不开各种div标签和样式的，如何优雅的组织三者到一起，我更喜欢reactjs的方式

至于在ng之间如何取舍，可以参考这个文章[choosing-between-react-vs-angular-2](http://react-etc.net/entry/choosing-between-react-vs-angular-2)

> React is a great technology and as a library it's ideal for introducing into old applications. The limited opinions React has on other technologies has cultivated a rich environment of supporting tools. And that is the problem, it's not very coherent or tangible for beginners.


>


> Angular 2 on the other hand continues on the path of being a framework. That makes it more of a product and easier to get into. The Angular 2 team has already done a lot of decisions for developers and as such is more uniform than anything that's "built with React".

## 二、 如何做

之前组里同学 [妈咪特卖](mami.baidu.com)基于reactjs+node的开发方式，相关文章[百度母婴技术团队—基于Reactjs实现webapp](https://github.com/my-fe/wiki/issues/1)

我简单的用图画下架构图哈

![技术架构](https://cloud.githubusercontent.com/assets/15227832/10684959/0657af3e-7986-11e5-92b7-c4b9164c0add.png)

后端的技术架构

![后端架构](https://cloud.githubusercontent.com/assets/15227832/10685018/e628db9c-7986-11e5-93bc-2dc1f0b871fd.png)


从上面可以看出在实践reactjs过程当中，结合自身的业务，实现了前后端同构，是一个很好的实践，不过当时没参与，接下来我就梳理下自己的思路，对于react本身我也是刚刚入门，demo是照搬的，主要的技术点如下

1、react生态

. redux:单向数据流管理

. react-redux:react和redux结合

. react-router:路由管理

. redux-thunk:可以让你愉快的dispatch，官方解释如下

Redux Thunk middleware allows you to write action creators that return a function instead of an action. The thunk can be used to delay the dispatch of an action, or to dispatch only if a certain condition is met. The inner function receives the store methods dispatch and getState as parameters.

简单解释可以返回一个action处理的函数非单纯的action，延迟dispatch或者有条件的dispatch，大白话就是异步dispatch

.react-dom: 解决服务端直出时模拟render时的上下文出错用的 


上面是react前端涉及的几个组件，有人说store树会随着项目越来越大变的臃肿，有人提出immutable.js扁平化管理，本次暂不做尝试

2、node生态

node现成的框架express、KOA已经集成了开发所需要的大部分功能，中间价相对比较完善。和[妈咪特卖](mami.baidu.com)的一样，采用组内larkjs，其在koa的基础上，划分出来了con

controller以及pageservice、dataservice以及daoservice，目录结构清晰，约定的路由及可复用的model层极大方便了开发效率，目前支持es6的generator

![](https://raw.githubusercontent.com/changfuguo/es6/master/2016-06/2016-06-14-reactjs-isomorph-practice/node-art.png)


3、服务端渲染

为了更好的seo，进行服务端直出。服务端直出在之前的项目里对css以及其他静态资源的打包是通过gulp实现的，但是为了更好的利用webpack的HMR模块，这里对css走webpack的loader；这样引申出来一个问题，服务端直出的js由于编译后不再和静态资源处于一个目录，那么client端require的css，则会找不到module，运行server端的脚本会报如下错误：（当然，你绝对可以通过gulp来打包css及其它静态资源，没有什么不好，仅仅是丢失了HMR特性而已）

![](https://raw.githubusercontent.com/changfuguo/es6/master/2016-06/2016-06-14-reactjs-isomorph-practice/can-not-find-module.png)


查阅了好多资料，找到一个[iamdustan](https://twitter.com/iamdustan/status/577561601353465856),[Backend-Apps-with-Webpack](http://jlongster.com/Backend-Apps-with-Webpack--Part-I)给出的解决办法如下,在webpack的配置中增加如下插件

```
plugins:[
        	new webpack.IgnorePlugin(/\.(css|scss)$/),
        	new webpack.NormalModuleReplacementPlugin(/\.(css|scss)$/, 'node-noop')
        ],
        
```
但是实验之后无果，依然报错。

最终的解决办法采用这个哥们的办法[isomorphic-react-in-real-life](https://reactjsnews.com/isomorphic-react-in-real-life):在webpack打包的时候传入一些指示是服务端还是浏览器端执行的变量，如上图中所示：

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

而在服务端的配置则不处理__BROWSER__变量，注意这里**require**是运行时加载的，所以该分之在服务端不会执行，避免了找不到模块的错误；

但是这样一来，你就不能用[css modules](https://github.com/css-modules/css-modules)的一些特性了，至少是不能全盘使用了，具体可参考上述连接

好，服务端问题over了，来看下HMR

4、HMR（hot module replacement）

这个特性之前是在研究react-native时发现的，当时就觉得惊呆了，有木有！无丢失状态、无需手动刷新，瞬间觉得效率提高了有没有；查阅资料的时候发现在reactjs有过几种热更新的方式

.webpack-dev-server: 

	参考 [webpack-dev-server](http://webpack.github.io/docs/webpack-dev-server.html),[Webpack-dev-server结合后端服务器的热替换配置](http://www.jianshu.com/p/8adf4c2bfa51) ,需要配合 [react-hot-loader](http://gaearon.github.io/react-hot-loader/getstarted/)实现

 	缺点：这种方式需要开双服务！！！
 
.webpack-hot-middleware + webpack-dev-middleware 

	直接集成到node的后台服务，以中间价 形式接入hot的请求，配置如下
			

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


5，服务端开发

尝鲜作祟，现在的larkjs未能全面支持es6特性，为了能使用到这些新特性和client保持一致；所以服务端的代码也需要编译，这个编译过程放到gulp中编译；由于lark的对mvc代码在加载的时候已经将上下文保存起来，要实现服务端的热更新，只能watch完之后只能重启服务，**这样做导致前端页面的数据状态丢失，需要刷新页面**。


  这里注意的是，larkjs自动加载models和controller中的文件，并用之前的module.exports导出，这里如果用es的export default 形式导出的话，会报找不到模块哟，所以导出的还沿用之前的写法
  
 




