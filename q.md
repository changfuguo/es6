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


