> 英文水平有限，只是对q的库做了翻译，加了自己一些解释，见原文 [https://github.com/kriskowal/q/tree/v1/design](https://github.com/kriskowal/q/tree/v1/design "desing")

#Part 1
本文试图解释promises是如何工作的，通过逐步构建一个promise库揭示它为什么能以一种特别的方式进行工作，最后向读者展示了它的主要设计思想。

假设你正在写一个不能立即有返回值的function。通常的API是提供一个回调函数将返回的值传入到回调函数中，而不是立即返回一个值。


	var oneOneSecondLater = function (callback) {
	    setTimeout(function () {
	        callback(1);
	    }, 1000);
	};

对于不复杂的问题这是一个简单的解决方案，但是仍然后很大的提升空间。

更加一般的做法是
