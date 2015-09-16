> 英文水平有限，只是对q的库做了翻译，加了自己一些解释，见原文 [https://github.com/kriskowal/q/tree/v1/design](https://github.com/kriskowal/q/tree/v1/design "desing");下文中有PS的是我加上的哈~

#Part 1
----
本文试图解释promises是如何工作的，通过逐步构建一个promise库揭示它为什么能以一种特别的方式进行工作，最后向读者展示了它的主要设计思想。

假设你正在写一个不能立即有返回值的function。通常的API是提供一个回调函数将返回的值传入到回调函数中，而不是立即返回一个值。


	var oneOneSecondLater = function (callback) {
	    setTimeout(function () {
	        callback(1);
	    }, 1000);
	};

对于不复杂的问题这是一个简单的解决方案，但是仍然后很大的提升空间。

更一般的做法是提供一个能同时返回值并且能抛出错误的工具。很明显的一些做法是扩展回调函数的逻辑到处理错误的逻辑中。这里演示同时提供回调和错误处理。

	var maybeOneOneSecondLater = function (callback, errback) {
	    setTimeout(function () {
	        if (Math.random() < .5) {
	            callback(1);
	        } else {
	            errback(new Error("Can't provide one."));
	        }
	    }, 1000);
	};

还有其他一些方法，如将出错的变量作为参数提供给回调，或者通过变量位置，或者通过一个有显著差别的标记值。尽管如此，没有一个方法能在模型运行中抛出错误。捕捉错误或者`try/catch`块的作用就是推迟到程序返回到调用位置时，能够显式地捕捉错误。(PS:个人理解该句话的意思，捕捉错误我们尽量在外部捕捉，在callback之外提供一个出错机制，当内部出错了，外部能捕捉到)。如果错误没有被处理，我们需要一些机制保证错误能隐式的传播。


#Part 2 promise
----
考虑一个能替代直接返回值或者直接抛出错误的一般方法，函数直接返回一个对象，该对象等同于最后函数的返回值，有可能成功或者失败。这个对象就是promise，从名字上可以看出，最终这些个promise是要被resolve掉的。我们可以调用promise上的函数来观察它是否完全执行还是拒绝执行。如果promise被拒绝了，且拒绝的行为没有被明显的观察到，那么在promise链上的其他promise也会由于同样的原因而拒绝执行。（PS：个人理解，一旦当前出错了，该错误会一直向下抛出，直到某一个errcall把他接住处理）

在这个迭代的设计过程当中，我们设计一个具有`then`方法的promise模型，通过该方法，我们能注册回调。


	var maybeOneOneSecondLater = function () {
	    var callback;
	    setTimeout(function () {
	        callback(1);
	    }, 1000);
	    return {
	        then: function (_callback) {
	            callback = _callback;
	        }
	    };
	};
	
	maybeOneOneSecondLater().then(callback);


该设计有两个不足：

- `then`方法第一次调用决定之后要使用的回调。如果每个注册的回调都能被通知到，那么这个方案将非常有用
- 在`promise`构建好之后，超过1s才注册回调，那么回调将不会被调用

通用的方案是我们接受一系列的回调函数，允许他们在超时之前或超时之后都能注册。通过设置一个有两个状态的变量实现这个方案。

	var maybeOneOneSecondLater = function () {
	    var pending = [], value;
	    setTimeout(function () {
	        value = 1;
	        for (var i = 0, ii = pending.length; i < ii; i++) {
	            var callback = pending[i];
	            callback(value);
	        }
	        pending = undefined;
	    }, 1000);
	    return {
	        then: function (callback) {
	            if (pending) {
	                pending.push(callback);
	            } else {
	                callback(value);
	            }
	        }
	    };
	};



作为一个功能性质的函数来说，这已经足够了。`deferred ` 是有两个部分的对象：一部分是注册观察者；另外一部分通知观察者，如下

	var defer = function () {
	    var pending = [], value;
	    return {
	        resolve: function (_value) {
	            value = _value;
	            for (var i = 0, ii = pending.length; i < ii; i++) {
	                var callback = pending[i];
	                callback(value);
	            }
	            pending = undefined;
	        },
	        then: function (callback) {
	            if (pending) {
	                pending.push(callback);
	            } else {
	                callback(value);
	            }
	        }
	    }
	};
	
	var oneOneSecondLater = function () {
	    var result = defer();
	    setTimeout(function () {
	        result.resolve(1);
	    }, 1000);
	    return result;
	};
	
	oneOneSecondLater().then(callback);

### `PS`

> 到这里已经具备promise的雏形了
> 
> 1. 可以任意时间添加任意多的回调
> 2. 可以人为决定什么时候resolve，而不是内部setTimeout实现
> 3. 当promise被resolve之后，还可以添加回调，只不过立即就执行了
> ###但是还存在一些问题，
> 1. 添加回调只能通过`defer.then`添加，非链式
> 2. defer.resolve方法中参数value，应用到每一个callback上，实际上还有一种情况callback2，
>  要用callback1的返回值作为自己参数无法满足
> 3. 无法传播出错情况
>  显然作者也考虑到了同样的问题哈哈，先从可以出错这入手



	var defer = function () {
	    var pending = [], value;
	    return {
	        resolve: function (_value) {
	            if (pending) {
	                value = _value;
	                for (var i = 0, ii = pending.length; i < ii; i++) {
	                    var callback = pending[i];
	                    callback(value);
	                }
	                pending = undefined;
	            } else {
	                throw new Error("A promise can only be resolved once.");
	            }
	        },
	        then: function (callback) {
	            if (pending) {
	                pending.push(callback);
	            } else {
	                callback(value);
	            }
	        }
	    }
	};


到这里你可能有一个观点是：要买抛出一个错误，要不忽略所有下游的回调。（。。。。。翻译不出来了，贴上原文）
You can make an argument either for throwing an error or for ignoring all subsequent resolutions.  One use-case is to give the resolver to a bunch of workers and have a race to resolve the promise, where subsequent resolutions
would be ignored.  It's also possible that you do not want the workers to know which won.  Hereafter, all examples will ignore rather than fault on multiple resolution.

综上所述，defer要能同时处理 多种 resolution 和observation

#part 3 进阶

该设计中两个独立的两部分引入了一些变量。第一部分使得分离和组合`deferred`的`resolve`和`promise`变得非常容易。同时也使得我们很容易区分`promise`和其他值。

按照最少授权原则（PS：职责分离原则？)，通过编码分离promise和resolver。如果授权给某人一个promise，这里只允许他增加观察者。如果授权给某人resolver，他应当仅仅能决定什么时候给出解决方案。彼此职责不能混淆。大量实验表明任何任何不可避免的越权行为将变动很难维护。

###`PS`
> 我理解上面这段话就是 ，我们应该明确返回一个promise，这个promise只是用来添加观察者，不是defer本身具有then，而是返回的promise
> 具备这样的能力。上面的这段话实际上揭示了为什么从`defer.then`转变成`defer.promise.then`，`promise`本身会有一些状态

这种方法的不足之处在于，因为不能迅速处理使用过的`promise`，所以给垃圾回收机制造成一定困难。

还有很多方法来区分`promise`和其他值的方法，这里用的是原型继承

	var Promise = function () {
	};
	
	var isPromise = function (value) {
	    return value instanceof Promise;
	};
	
	var defer = function () {
	    var pending = [], value;
	    var promise = new Promise();
	    promise.then = function (callback) {
	        if (pending) {
	            pending.push(callback);
	        } else {
	            callback(value);
	        }
	    };
	    return {
	        resolve: function (_value) {
	            if (pending) {
	                value = _value;
	                for (var i = 0, ii = pending.length; i < ii; i++) {
	                    var callback = pending[i];
	                    callback(value);
	                }
	                pending = undefined;
	            }
	        },
	        promise: promise
	    };
	};

使用原型继承的一个缺点是在单一的程序中只能有一个promise的实例。这个很难强制去使用，导致强制依赖的困窘。

另外一个是用填鸭子的方式，用一个已经存在的大家都认可的具名方法来区分promise和其他值。`CommonJS/Promises/A`使用`then`方法来区分自己的promise和其他值。这个方法有个不足之处，恰巧当前传入的对象有`then`方法时，将很难区分上述情况。但是，那都不是事，这种小概率事件是可控的。


	var isPromise = function (value) {
	    return value && typeof value.then === "function";
	};
	
	var defer = function () {
	    var pending = [], value;
	    return {
	        resolve: function (_value) {
	            if (pending) {
	                value = _value;
	                for (var i = 0, ii = pending.length; i < ii; i++) {
	                    var callback = pending[i];
	                    callback(value);
	                }
	                pending = undefined;
	            }
	        },
	        promise: {
	            then: function (callback) {
	                if (pending) {
	                    pending.push(callback);
	                } else {
	                    callback(value);
	                }
	            }
	        }
	    };
	};




接下来的问题是如何使得组合使用promise变得更加容易，我们要能用旧的`promise`的返回值驱动新的`promise`（PS:上面说的，callback2 要用callback1的返回值如何解决）。假设你能从一个求两数之和的函数得到一个`promise`作为返回值，我们可以创造一个新的`promise`用来求两者之和。考虑下如何用回调实现：

	var twoOneSecondLater = function (callback) {
	    var a, b;
	    var consider = function () {
	        if (a === undefined || b === undefined)
	            return;
	        callback(a + b);
	    };
	    oneOneSecondLater(function (_a) {
	        a = _a;
	        consider();
	    });
	    oneOneSecondLater(function (_b) {
	        b = _b;
	        consider();
	    });
	};
	
	twoOneSecondLater(function (c) {
	    // c === 2
	});

为了使上面的想法得以实现，需要准备几样东东


-  `then` 方法必须返回一个`promise`
-  上步中返回的`promise`必须能使用callback的返回值最终被`resolved`掉
-  `callback`的返回值必须是一个完全执行的值或者一个`promise`

所以将返回值也转化为promise的形式很简单。这种类型的promise能使用之前获取到的value立即调用任何一个观察者；


	var ref = function (value) {
	    return {
	        then: function (callback) {
	            callback(value);
	        }
	    };
	};

### `PS`
> 1. 作者的这个意思就是，要实现callback返回值的传递，如callback1的返回值传给callback2，将callback2的返回值传给callback3
> 在代码里将上一次的value用ref闭包封装起来，下一次只关心传入的回调即可，ref.then上次的value传入本次的callback中。
> 2. 作者下面更进一步，ref.then链式调用，请君细看下面。同时还要处理value是promise的情况，如果本身是promise，则用value的then方法进行转场，接管当前所在promise的then


下面的方法可以不用考虑传入的值本身一个普通值还是promise，直接将其将传入的value转成promise

var ref = function (value) {
    if (value && typeof value.then === "function")
        return value;
    return {
        then: function (callback) {
            callback(value);
        }
    };
};


接下来，为了能使`then`方法也能返回一个promise，我们来改造下`then`方法；我们强制将callback的返回值传入下一个promise并立即返回。

	var ref = function (value) {
	    if (value && typeof value.then === "function")
	        return value;
	    return {
	        then: function (callback) {
	            return ref(callback(value));
	        }
	    };
	};

对于deferred 来说，回调在将来被调用已经足够复杂了。这里我们借助于defer本身，将callback封装。我们将用callback的返回值驱动`then`返回的promise。

更进一步，`resolve`方法应该能处理resolution本身是一个promise的情况，这里可以把value改为一个promise。它有一个then方法，这个方法可能是由`defer`返回的，也可能是`ref`返回的。如果是`ref`类型的，其行为等同于：callback立即被`then(callback)` 执行。如果是一个`defer`返回的，callback被传入到下一个promise中的`then`方法；所以变成了callback可以监听一个新的promise以便能获取完全执行后的value。回调要执行很多次，可以制作一个进度条反应回调的执行情况



	var defer = function () {
	    var pending = [], value;
	    return {
	        resolve: function (_value) {
	            if (pending) {
	                value = ref(_value); // values wrapped in a promise
	                for (var i = 0, ii = pending.length; i < ii; i++) {
	                    var callback = pending[i];
	                    value.then(callback); // then called instead
						//奇妙之处在这里，上述then可能是ref.then可能是defer.promise.then，不管是那个两者都接受一个回调
						//一个是用来处理value 一个用来驱动下一个resolve
	                }
	                pending = undefined;
	            }
	        },
	        promise: {
	            then: function (_callback) {
	                var result = defer();
	                // callback is wrapped so that its return
	                // value is captured and used to resolve the promise
	                // that "then" returns
	                var callback = function (value) {
	                    result.resolve(_callback(value));
	                };
	                if (pending) {
	                    pending.push(callback);
	                } else {
	                    value.then(callback);
	                }
	                return result.promise;
	            }
	        }
	    };
	};

这里用可thenable的promise来分离deferred的‘promise’和‘resolve’部分



#Part 4 错误传播

为实现错误传播，我们再次引入errback。我们需要一种类似于`ref`类型的新型promise，就像promise完全执行时调用callback一样，它会告知errback拒绝执行以及拒绝的原因。

	var reject = function (reason) {
	    return {
	        then: function (callback, errback) {
	            return ref(errback(reason));
	        }
	    };
	};

这是监听有立即返回拒绝值的最简单的办法

	reject("Meh.").then(function (value) {
	    // we never get here
	}, function (reason) {
	    // reason === "Meh."
	});


这里可以利用promise的API重新引入errback

	var maybeOneOneSecondLater = function (callback, errback) {
	    var result = defer();
	    setTimeout(function () {
	        if (Math.random() < .5) {
	            result.resolve(1);
	        } else {
	            result.resolve(reject("Can't provide one."));
	        }
	    }, 1000);
	    return result.promise;
	};

### `PS`
> 1. reject的也用resolve来返回的，其实是不合理的，之前作者也说了职责分明吗
> 2. reject里接受两个参数，但是实际上只用了errback也是不合理的

为了使上面的例子能工作，defer系统需要一个新的管道，使得能同时处理callback和errback。所以pending将被设计为可以同时传入callback和errback作为`then`方法回调的参数



	
	var defer = function () {
	    var pending = [], value;
	    return {
	        resolve: function (_value) {
	            if (pending) {
	                value = ref(_value);
	                for (var i = 0, ii = pending.length; i < ii; i++) {
	                    // apply the pending arguments to "then"
	                    value.then.apply(value, pending[i]);
	                }
	                pending = undefined;
	            }
	        },
	        promise: {
	            then: function (_callback, _errback) {
	                var result = defer();
	                var callback = function (value) {
	                    result.resolve(_callback(value));
	                };
	                var errback = function (reason) {
	                    result.resolve(_errback(reason));
	                };
	                if (pending) {
	                    pending.push([callback, errback]);
	                } else {
	                    value.then(callback, errback);
	                }
	                return result.promise;
	            }
	        }
	    };
	};


但是，这个版本的defer仍旧有些小问题：errback必须在调用then方法时提供，或者调用一个不存在的errback时，会抛出错误。最简单的办法是提供默认的回调，以便能传递拒绝的行为。如果你仅仅对监控错误感兴趣，那么忽略错误的回调也是合理的（PS：个人立即，不捕捉出错，但是出错的时候别中断程序），所以我们提供一个回调一遍程序能完全执行。
	
	var defer = function () {
	    var pending = [], value;
	    return {
	        resolve: function (_value) {
	            if (pending) {
	                value = ref(_value);
	                for (var i = 0, ii = pending.length; i < ii; i++) {
	                    value.then.apply(value, pending[i]);
	                }
	                pending = undefined;
	            }
	        },
	        promise: {
	            then: function (_callback, _errback) {
	                var result = defer();
	                // provide default callbacks and errbacks
	                _callback = _callback || function (value) {
	                    // by default, forward fulfillment
	                    return value;
	                };
	                _errback = _errback || function (reason) {
	                    // by default, forward rejection
	                    return reject(reason);
	                };
	                var callback = function (value) {
	                    result.resolve(_callback(value));
	                };
	                var errback = function (reason) {
	                    result.resolve(_errback(reason));
	                };
	                if (pending) {
	                    pending.push([callback, errback]);
	                } else {
	                    value.then(callback, errback);
	                }
	                return result.promise;
	            }
	        }
	    };
	};


到这里，我们已经实现隐式的错误传播。我们现在能很容易的从并行或者串行的promise中创建新的promise。下面的例子实现了从求和的promise中创建一个新的promise


	promises.reduce(function (accumulating, promise) {
	    return accumulating.then(function (accumulated) {
	        return promise.then(function (value) {
	            return accumulated + value;
	        });
	    });
	}, ref(0)) 
	//将带有值为0的promise传递到reduce里，返回的promise就是accumulating，0的值指向accumulated
	//在内部将值传递给了promise，来说实现求和
	.then(function (sum) {
	    // the sum is here
	});

#Part 5 安全性和稳定性

另外一个可以优化的地方那个是我们要确保在将来调用中能和callback和errback的注册顺序保持一致。这将极大减少异步变成中的流程控制的步骤，考虑下面一个简单的例子:
	
	var blah = function () {
	    var result = foob().then(function () {
	        return barf();
	    });
	    var barf = function () {
	        return 10;
	    };
	    return result;
	};

这个函数可能抛出一个错误或者返回一个立即被执行的promise，其参数是完全执行后返回的10。blah能否正确执行依赖`foob()`是否在同一个事件轮询或将来某个时间点中被resolve了。如果result返回的promise中的回调将来执行，那么将一直是成功的。

###`PS`
> 这段话看的我晕乎乎的，不知道啥意思。大概琢磨是如果blah返回的promise中，如果callback能确保将来的轮询中执行
> 那么回调将一直能成功返回


	var enqueue = function (callback) {
	    //process.nextTick(callback); // NodeJS
	    setTimeout(callback, 1); // Naïve browser solution
	};
	
	var defer = function () {
	    var pending = [], value;
	    return {
	        resolve: function (_value) {
	            if (pending) {
	                value = ref(_value);
	                for (var i = 0, ii = pending.length; i < ii; i++) {
	                    // XXX
	                    enqueue(function () {
	                        value.then.apply(value, pending[i]);
							//这种写法其实是有问题的，i始终返回pending.length
	                    });  
	                }
	                pending = undefined;
	            }
	        },
	        promise: {
	            then: function (_callback, _errback) {
	                var result = defer();
	                _callback = _callback || function (value) {
	                    return value;
	                };
	                _errback = _errback || function (reason) {
	                    return reject(reason);
	                };
	                var callback = function (value) {
	                    result.resolve(_callback(value));
	                };
	                var errback = function (reason) {
	                    result.resolve(_errback(reason));
	                };
	                if (pending) {
	                    pending.push([callback, errback]);
	                } else {
	                    // XXX
	                    enqueue(function () {
	                        value.then(callback, errback);
	                    });
	                }
	                return result.promise;
	            }
	        }
	    };
	};
	
	var ref = function (value) {
	    if (value && value.then)
	        return value;
	    return {
	        then: function (callback) {
	            var result = defer();
	            // XXX
	            enqueue(function () {
	                result.resolve(callback(value));
	            });
	            return result.promise;
	        }
	    };
	};
	
	var reject = function (reason) {
	    return {
	        then: function (callback, errback) {
	            var result = defer();
	            // XXX
	            enqueue(function () {
	                result.resolve(errback(reason));
	            });
	            return result.promise;
	        }
	    };
	};

这里仍旧有一些安全性的问题；假设任何一个具有`then`方法的对象都被当作promise来处理，那么直接调用`then`方法可能有意向不到的‘收获’。

- callback或者errback必须以同样的顺序被调用
- callback或者errback可能会被同时调用
- callback或者errback可能会被调用多次

用`when`方法封装下promise以此阻止错误发生

所以我们封装callback和errback以便任何的异常都能被转移到出错的机制里去



	var when = function (value, _callback, _errback) {
	    var result = defer();
	    var done;
	
	    _callback = _callback || function (value) {
	        return value;
	    };
	    _errback = _errback || function (reason) {
	        return reject(reason);
	    };
	
	    var callback = function (value) {
	        try {
	            return _callback(value);
	        } catch (reason) {
	            return reject(reason);
	        }
	    };
	    var errback = function (reason) {
	        try {
	            return _errback(reason);
	        } catch (reason) {
	            return reject(reason);
	        }
	    };
	
	    enqueue(function () {
	        ref(value).then(function (value) {
	            if (done)
	                return;
	            done = true;
	            result.resolve(ref(value).then(callback, errback));
	        }, function (reason) {
	            if (done)
	                return;
	            done = true;
	            result.resolve(errback(reason));
	        });
	    });
	
	    return result.promise;
	};


到这里，我们的方案能确保不会有哪些突发性的错误，包括哪些非必需的事件流控制，并且也能使callback和errback各自保持独立。




###Part 5 消息传递

现在来看，promise变成一个有then方法的对象。Deferred promises 负责传递消息给他们具体的 promise。完全执行的promise通过将完全执行后的值传递给callback来进行消息传递。拒绝类型promise通过传递给errback拒绝的原因来实现消息相应。

由此一般地我们认为promise应该是能接受任意消息的对象，包括`then/when`消息。对于有大量的非立即返回的监听来说非常有用的,就像是promise可以被看作是另外一个金星或者worker或者网络上的另一台计算机。
如果我们等待一个网络完全执行完返回的消息值，这个往返的过程可能会浪费掉很多时间。这归结于‘非正式’的网络协议，也是SOAP和RPC逐渐衰落的原因。


尽管如此，我们可以在一个远程的promise被resolve之前发送一个消息，远程promise在成功执行之后能立即返回相应消息。考虑一般情况是，一个对象可能被存放在远程服务器上，自身不能通过网络传输。它有一些内在的状态和能力以至于不能被序列化，如访问数据库。假设我们现在能从这个对象获取到一个promise并且可以发送消息。这些消息很可能有诸如`query`之类的方法构成，这些方法能以此发送反馈的promise。

所以我们构建一套新的promise，这套promise基于一些能发送任意消息的方法之上。`CommonJS/Promises/D`中定义`promiseSend`方法。发送一个`when`类型消息等同于调用`then`方法

	promise.then(callback, errback);
	promise.promiseSend("when", callback, errback);

我们必须重新审视我们所有的方法，用`promiseSend`方法代替`then`方法进行构建。但是，我们不会完全放弃`then`方法的形式，我们仍旧能处理`thenable`类型的promise，只是在内部还是调用`promiseSend`而已。


	function Promise() {}
	Promise.prototype.then = function (callback, errback) {
	    return when(this, callback, errback);
	};

如果promise不能识别出消息的类型（例如 when），他肯定返回一个最终将被拒绝的promise

能接受任意的消息以为这可以执行作为远程promise代理的新型promise，，可以把所有消息发送到远程的promise并且本地的promise能否处理远程promise的响应值


对于上述的用例代理层和未被确认的消息来说，创建一种抽象的promise是非常有用的：它只负责将认识的消息传递给能够处理的对象，未确认的消息传给错误的回调来进行处理。


	var makePromise = function (handler, fallback) {
	//是一个抽象方法，生成promise
	    var promise = new Promise();
	    handler = handler || {};
	    fallback = fallback || function (op) {
	        return reject("Can't " + op);
	    };
	    promise.promiseSend = function (op, callback) {
	        var args = Array.prototype.slice.call(arguments, 2);
	        var result;
	        callback = callback || function (value) {return value};
	        if (handler[op]) {
	            result = handler[op].apply(handler, args);
	        } else {
	            result = fallback.apply(handler, [op].concat(args));
	        }
	        return callback(result);
	    };
	    return promise;
	};


期望handler 的每一个方法以及fallback方法能返回一个可以传入callback的值。handler本身不接受任何操作符的名字，但是fallback需要接受操作符以便进行路由操作，同时多余的参数也被传入进去。

对于ref方法，我们仍强制将其返回值转化为promise。我们把`thenables`方法固化到`promiseSend`，在完全执行的返回值上我们提供了一些基本的交互行为，包括属性操作和方法回调
	
	var reject = function (reason) {
	    var forward = function (reason) {
	        return reject(reason);
	    };
	    return makePromise({
	        when: function (errback) {
	            errback = errback || forward;
	            return errback(reason);
	        }
	    }, forward);
	};

这套defer仍有瑕疵。我们取代数组传参给`then`方法，进而使用"promiseSend"、"makePromise" 和"when"来处理callback和errback的默认参数和封装。


var defer = function () {
    var pending = [], value;
    return {
        resolve: function (_value) {
            if (pending) {
                value = ref(_value);
                for (var i = 0, ii = pending.length; i < ii; i++) {
                    enqueue(function () {
                        value.promiseSend.apply(value, pending[i]);
                    });
                }
                pending = undefined;
            }
        },
        promise: {
            promiseSend: function () {
                var args = Array.prototype.slice.call(arguments);
                var result = defer();
                if (pending) {
                    pending.push(args);
                } else {
                    enqueue(function () {
                        value.promiseSend.apply(value, args);
                    });
                }
            }
        }
    };
};

最后一步是在语法上向promise发送消息更加便捷，我们创建了 "get", "put", "post" and "del" 能发送相应的消息并且从返回结果中取得相应promise。他们看起来如此相似

 
	
	var get = function (object, name) {
	    var result = defer();
	    ref(object).promiseSend("get", result.resolve, name);
	    return result.promise;
	};
	
	get({"a": 10}, "a").then(function (ten) {
	    // ten === ten
	});
	

最后一步是将promise提升至‘状态艺术的层面’，所有成功的回调都使用win，所有失败的回调都使用tail，留给读者自己实践



#part 6  Future



Andrew Sutherland did a great exercise in creating a variation of the Q
library that supported annotations so that waterfalls of promise creation,
resolution, and dependencies could be graphically depicited.  Optional
annotations and a debug variation of the Q library would be a logical
next-step.

There remains some question about how to ideally cancel a promise.  At the
moment, a secondary channel would have to be used to send the abort message.
This requires further research.

CommonJS/Promises/A also supports progress notification callbacks.  A
variation of this library that supports implicit composition and propagation
of progress information would be very awesome.

It is a common pattern that remote objects have a fixed set of methods, all
of which return promises.  For those cases, it is a common pattern to create
a local object that proxies for the remote object by forwarding all of its
method calls to the remote object using "post".  The construction of such
proxies could be automated.  Lazy-Arrays are certainly one use-case.
 
