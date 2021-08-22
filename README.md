# Promise的链式调用、异常穿透、中断promise链

## promise.then()的参数

.then是**Promise原型上的一个方法**, 返回一个promise对象, 它可以接受两个回调参数: 1. `onResolved`, 2. `onRejected`, 这两个参数各自有一个参数, 分别是: 执行成功的数据 和 执行失败的数据。 其实我们平时用的最多的就是第一个回调函数, 用它来处理请求成功后的数据。比如:

```js
new Promise((resolve, reject) => {
	resolve('成功了');
}).then((data) => {
	// 这里的data就是 异步请求成功后 resolve接收到的值
	console.log('success: ', data); // output: success: 成功了
})
复制代码
```

但其实, 它还有第二个回调函数, 用来处理失败时的数据:

```js
new Promise((resolve, reject) => {
	reject('success');
}).then(
(data) => {	console.log('success: ', data);},
(err) => { console.log('failed: ', err);}
)
复制代码
```

## promise链式调用的两个注意点:

### 链式调用时, 上一次.then()处理的数据变成了undefined?

我们来看一下这面这个例子:

```js
new Promise((resolve, reject) => {
	resolve('成功了')
	// reject('失败了')
})
	.then(
		(data) => { console.log('onResolved1', data);},
		(error) => { console.log('onRejected1', error);}
	)
	.then(
		(data) => { console.log('onResolved2', data);},
		(error) => { console.log('onRejected2', error);}
	)
复制代码
```

↑ 上面会打印什么结果呢? 会是: 'onResolved1', '成功了' 'onResolved2', '成功了' 吗? ![在这里插入图片描述](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/86b2f9f5e872427e93c13e2aba97ecbe~tplv-k3u1fbpfcp-watermark.awebp)

会发现在第2个.then()中, 虽然也走了处理成功的回调, 但是数据却丢失了, 变成了undefined. 说明上一个.then()返回的promise对象, 它的数据肯定丢失了, 我们可以再来打印一下这个promise: ![在这里插入图片描述](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/18b791a9edc44851a9f839ea5634a58b~tplv-k3u1fbpfcp-watermark.awebp)

不难看出, 这里的result确实变成了undefined, 以至于链式调用时, 下一个.then()拿不到结果了. 其实`本质原因`是: **如果想链式调用, 那么上一个.then()里处理数据的`回调函数必须要有返回值`(不管是onResolved 还是 onRejected), 否则数据就会丢失, 并且默认执行下一个.then里的onResolved回调!** 我们返回时, 有两种情况(固定值 / promise):

```js
// 直接写.then部分了:
.then(
	(data) => {
		console.log('onResolved1', data)
		// 情况1: 返回固定值
		return 'success'
		// 情况2: 返回promise
		return new Promise((resolve, reject) => {
			resolve(data)
		})
	},
)
.then(
  (data) => { console.log('onResolved2', data)},
  (error) => { console.log('onRejected2', error)}
)
复制代码
```

以上两种情况, 都可以正常执行, 让下一个.then()中的回调函数拿到值: ![在这里插入图片描述](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7e8b51746c574061a64a3e91b4c3ebdf~tplv-k3u1fbpfcp-watermark.awebp)

### 如何根据上一个.then决定执行下一个.then中的回调?

既然.then()里面有两个回调, 我怎么知道链式调用的时候该用处理成功的 还是处理失败的呢?

```js
// 现在执行失败:
const p = new Promise((resolve, reject) => {
  reject('失败了')
}).then(
  (data) => { console.log('onResolved1', data)},
  (error) => {
  	// 场景1: 没有return
    console.log('onRejected1', error)
    // 场景2: return 固定值
    return 'failed'
    // 场景3: return Promise.reject
    return Promise.reject(error)
    // 场景4: return Promise.resolve
    return Promise.resolve(error)
    // 场景5: 抛出错误:
    throw error;
  }
)
p.then(
  (data) => { console.log('onResolved2', data)},
  (error) => { console.log('onRejected2', error)}
)

复制代码
```

会发现:

| 上一个.then的返回值    | 下一个.then的执行结果 |
| ---------------------- | --------------------- |
| return undefined       | 执行onResolved        |
| return 固定值          | 执行onResolved        |
| return Promise.resolve | 执行onResolved        |
| return Promise.reject  | 执行onRejected        |
| throw 数据             | 执行onRejected        |

也就是说 上一个.then中**回调函数的返回值**的不同, 决定了下一个.then会执行不同的回调。(换句话说: `和上一个.then的执行哪个回调无关, 只和其返回值有关`)

## promise异常穿透

当然, 正如开头介绍,then的参数所说, 在.then()中, 我们经常传入onResolved用以处理成功时的数据, 一般不在then里面传入onRejected, 而处理失败的数据一般放在最后的.catch()中:

```js
// 现在执行失败:
new Promise((resolve, reject) => {
  reject('失败了')
}).then(
  (data) => { console.log('onResolved1', data)},
).then(
  (data) => { console.log('onResolved2', data)},
).then(
  (data) => { console.log('onResolved3', data)},
).catch(
  (err) => { console.log('onRejected1', data)},
)
复制代码
```

上述例子中, 一开始 reject就接收了失败的数据, 而数据最终也会通过.catch中的回调函数进行处理, 但是这里我想说明的是, 这个失败的数据是一下子就直接进入了.catch吗? 显然不是的, 既然是链式调用, 那必定也是一层层的传过去了, 但是在.then()中并没有传入处理失败数据的回调, 为什么还会正常往下传递呢? 其实, 当我们在.then()中, 不写第二个参数时, 默认是这样的:

```js
.then(
(data) => { ...处理data },
// 不写第二个参数, 相当于默认传了:
(err) => Promise.reject(err), 
// 或
(err) => { throw err; }
).then()
复制代码
```

这就是为什么链式调用时, 能在最后写.catch() 还能拿到数据的原因了

## 如何中断promise

如果等到最后.catch()处理完, 想结束promise链, 不想再让其链式调用下去了, 可以作如下操作:

```js
.catch((err) => {
  console.log('onRejected', err);
  // 中断promise链:
  return new Promise(() => {})
})
复制代码
```

通过返回一个状态一直为pending的promise即可。