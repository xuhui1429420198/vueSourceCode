# vue 源码

### vue 源码可以分为两部分，页面渲染，实现数据的响应式

+ 为了实现数据的响应式，```Vue``` 劫持了数据的 ```get set value ``` 等属性

### 数据劫持

+ 其中数据劫持 ```Vue 2``` 主要通过 ```defineProperty``` 实现, ```Vue 3 ``` 主要通过 ```Es6 Proxy``` 实现

#### Object.defineProperty

+ ```javascript``` 中*引用类型* 中获取属性会触发 ```get``` 方法，设置属性的值会触发 ```set``` 方法。基于这点我们就能在触发这些操作中对数据进行额外的处理。
~~~javascript
const descriptor = {
	configurable: false, // 属性是否可配置（delete 操作）
	enumerable: false, // 该属性是否可枚举
	get: () => {
		// 获取 target[key] 时调用的方法
	},
	set: val => {
		// target[key] = val 时调用的方法
	}
}

const descriptor2 = {
	// value 属性可以是 对象 数值 函数
	value: (arg) => {
		// 可以劫持原型方法
	}
}

Object.defineProperty(target, key, descriptor)
// descriptor 为一个配置对象
// target 为需要劫持的引用类型数据
// key 为被劫持的某引用类型上的属性（键名）
~~~

+ **注意：** ```value``` 描述符不能与 ```get set``` 同时出现

#### Proxy

+ Es6 中的新API 可以完全替代 defineProperty 的工作，使数据的劫持更加简单
~~~javascript
const handler = {
	get: (target, key) => {
		// 与defineProperty 不同，要被劫持的属性在这里进行设置
	},

	set: (target, key, value) => {
		// 同时 与 defineProperty 存在差异
	}
}

const proxy = new Proxy(target, handler)
// target 为被需要进行数据劫持的引用类型
// handler 为配置对象
~~~

### 发布订阅模式

+ 发布订阅模式也可以叫做观察者模式，他定义对象间的一种一对多的依赖关系，当一个对象的状态发生变化，所有依赖他的对象都将得到通知。在js开发中，一般用事件模型来代替传统的 发布-订阅模式。 **可以实现对象间的解耦**。

+ 谁是发布者，谁是订阅者？

	- 发布者要有一个缓存列表，用于存放回调函数，用于通知订阅者

	- ```vue``` 中 ```Dep``` 类用于实例化一个发布者，而订阅者实际上就是 ```vue``` 的 ```dom``` 渲染方法，由 ```Watcher``` 类实例化生成。

	- ```Vue``` 通过观察者模式实现数据与页面之间的的响应式变化（当数据更新，触发页面的更新)。

#### EventEmitter

+ 设置两个类 ```Dep```、 ```Watcher``` 分别用于生成发布者，与订阅者。

~~~javascript
let id = 0
// 事件存储
// Dep 用于对 Watcher 进行管理
class Dep{
	constructor(){
		this.id = ++id
		this.subs = []
	}

	// 收集订阅者（这里就是指收集 Watcher 实例对象）
	addSub(sub){
		this.subs.push(sub)
	}

	// 通知订阅者 （这里就是指触发 Watcher 实例对象上定义的回调函数）
	notify(){
		let subs = this.subs.slice()

		// 排序
		subs = subs.sort((a, b) => {
			a.id - b.id
		})
		subs.forEach(sub => {
			// Watcher 实例对象上的 update 方法用于执行指定的操作
			sub.update()
		})
	}
}

// 用于实例化生成指定的 回调方法以及想要的操作
class Watcher{
	constructor(vm, expressFn){
		// 实例化时指定想要的操作，即 expressFn 回调函数，以及回调函数所必须的作用域（vm） 等参数
		this.vm = vm
	    this.expressFn = expressFn
	}

	update(){
		// 在特定作用域下执行回调函数
		this.expressFn.call(this.vm)
	}
}

function defineReaction(){
	// 
	const dep = new Dep()

	// 添加事件
	const watcher = new Watcher(
		//...
		vm: null,
		expressFn: () => {
			console.log('数据更新了')
		}
	)

	dep.addSub(watcher)

	dep.notify()
}

defineReaction()
~~~

### 数据的劫持

+ ```Vue``` 实例化时需要传入一个配置对象，该对象中包含页面的挂载的根节点，以及数据对象（```data、 computed、 watch、 methods...```）

+ ```get set ```

~~~javascript
class Vue{
	constructor(options){
		this.$data = options.data

		this.initState()
	}

	initState(){
		// 用于实现 Data 对象的劫持
		for(let key in this.$data){
			Object.defineProperty(this, key, {
				get: ()=>{
					return this.$data[key]
				},
				set: newVal=>{
					this.$data[key] = newVal
				}
			})
		}
	}
}
const options = {
	el: '#app',
	data: {
		msg: 'PB'
	}
}
const vm = new Vue(options)

console.log(vm.msg) 
// 'PB'
~~~

### 依赖收集

+ 依赖收集是指将页面所使用到的数据收集到观察者中，用于实现数据更新==> 视图更新的过程

+ 在什么时机收集依赖？

	- 当数据被使用时才有被观察的必要，即当 获取数据时进行依赖收集比较合理。

	- 因此在进行数据劫持的```get```方法中进行依赖收集

+ **注意：只有引用类型的数据才能被劫持**

+ **由于数组的一些方法不会改变数组，因此不会触发观者者，因此需要通过 ```value``` 进行数组的方法劫持**

~~~javascript
// 需要劫持的方法
const methodsToPatch = [
  'push',
  'pop',
  'shift',
  'unshift',
  'splice',
  'sort',
  'reverse'
]

// 数组原型（用于实现数组方法的默认行为）
const arrayProto = Array.prototype

// 劫持后的方法
const arrayMethods = Object.create(arrayProto)

methodsToPatch.forEach( method => {
	// 用于实现数组方法的默认行为
	let original = arrayProto[method]
	Object.defineProperty(arrayMethods, method, {
		value: (...args) => {
			const result = original.apply(this, args)

			let inserted
			// 如果是插入数组的新值也需要进行依赖收集
		    switch (method) {
		      case 'push':
		      case 'unshift':
		        inserted = args
		        break
		      case 'splice':
		        inserted = args.slice(2)
		        break
		    }
		    if (inserted) {
	          console.log('数组数据插入')
	        }
		    // 数据变化，通知观察者
        	console.log('数组数据变化了')
			// 返回数组方法的默认返回值
			return result
		}
	})
})

// 然后再通过 value 方法拦截数组上
function copyAugment(target, src, keys){
  for (let i = 0, l = keys.length; i < l; i++) {
    const key = keys[i]
    Object.defineProperty(target, key, {
    	value: src[key]
    })
  }
}
const arr = [1]
copyAugment(arr, arrayMethods, methodsToPatch)
arr.push(2)
~~~

+ 依赖收集

~~~javascript
// 定义一个类用于实现依赖收集
class Observe{
	constructor(value){
		this.value = value
		// 挂载一个发布者实例对象，用于通知相应的观察者
		this.dep = new Dep() 
		// 将该对象挂载到被收集的数据对象上，便于扩展操作
		value.__ob__ = this

		// 依赖收集
		if(Array.isArray(value)){
			// 数组要进行遍历
			for(let i = 0; l = value.length; i< l; i++){
				observe(value[i])
			}

			// 进行数组方法的劫持
			copyAugment(value, arrayMethods, methodsToPatch)
		}else{
			this.walk(value)
		}
	}

	walk(value){
		for(let key in value){
			defineReactive(value, key)
		}
	}

}

// 真正的依赖收集方法
function defineReactive(target, key){
	const dep = new Dep()

	const property = Object.getOwnPropertyDescriptor(target, key)

	// 由于数据劫持，因此需要执行我们设定的 target 上的 get set 操作
	const getter = property.get
	const setter = property.set

	Object.defineProperty(target, key, {
		get: ()=>{
			// 在这里进行依赖收集
			dep.addSub()

			return getter.call(this)
		},
		set: newVal=>{
			// 数据更新触发 观察者
			dep.notify()

			return setter.call(this, newVal)
		}
	})
}

// 定义一个方法用于依赖收集
function observe(value){
	let ob

	// 判断数据类型
	if(value === null || typeof obj !== 'object'){
		return 
	}
	// 已经收集过的不在收集
	if(value.__ob__){
		ob = value.__ob__
	}else{
		ob = new Observe(value)
	}
	return ob
}
~~~

### 在什么时机创建订阅者watcher?

+ 当数据被使用的时候，创建订阅者才有意义，而数据被指用的时候就是数据被渲染的时候，因此在数据被渲染到页面之前创建一个订阅者是比较合理的,还有一个原因是，在数据被渲染的时候，我们才能知道，数据更新后应该进行什么样的操作才能刷新成我们想要的页面

	- ```Vue``` 在 ```mountComponent``` 方法中实例化一个 订阅者

~~~javascript
// updateComponent  为页面刷新方法

new Watcher(vm, updateComponent, noop, {
  before () {
    if (vm._isMounted && !vm._isDestroyed) {
      callHook(vm, 'beforeUpdate')
    }
  }
}, true /* isRenderWatcher */)
~~~

## 以下是新的

## 页面的渲染

+ ```Vue``` 通过什么策略将数据渲染到页面上的，

#### $mount

+ 实例化一个 ```Vue``` 实例后立即调用 ```vm.$mount('#app');``` 即可将 组件渲染到页面，这个```$mount``` 方法在 web/runtime/index.js 中添加到 ```Vue.prototype```

~~~javascript
Vue.prototype.$mount = function (
  el,
  hydrating
) {
  el = el && inBrowser ? query(el) : undefined
  return mountComponent(this, el, hydrating)   //组件挂载
}
~~~

+ ```mountComponent``` 这个方法是真正的渲染方法，在数据渲染到页面上时添加订阅者，进行依赖收集
~~~javascript
new Watcher(vm, updateComponent, noop, {
  before () {
    if (vm._isMounted && !vm._isDestroyed) {
      callHook(vm, 'beforeUpdate')
    }
  }
})
~~~

+ ```updateComponent``` 为一个方法，有两个作用：
	
	- 将传入的 ```Vue``` 文件组件转化为虚拟节点 (```_render()```)

	- 将虚拟节点渲染为真实的 ```Dom``` (```_update()```)

~~~javascript
vm._update(vm._render(), hydrating)
~~~

+ ```_render()``` 方法中调用了 ```Vue``` 实例化时传入的 ```render()``` 方法

	- ```vnode = render.call(vm._renderProxy, vm.$createElement)```，<font color="red">即实例化过程中传入的```render``` 方法入参是 ```Vue``` 内部定义的 ```createElement``` 方法，这个方法接受一个组件实例，进行节点的转换</font>

+ ```_update()``` 方法会调用 实例上挂载的 ```vm.$el = vm.__patch__(prevVnode, vnode)``` 进行节点的 ```diff ``` 与渲染

	- *```__patch__```方法在入口文件中挂载，web 渲染就是在 web/runtime/index 文件中挂载```Vue.prototype.__patch__ = inBrowser ? patch : noop```<font color="purple">即 最终 ```_update()``` 会调用 ```Vue``` 定义的 ```patch``` 方法</font>*

#### createElement

+ 这个方法将返回一个**虚拟节点实例对象**，主要有两种途径，直接实例化一个 虚拟节点实例对象；调用内部定义的 ```createComponent```创建一个组件

	- 当创建一个组件时会通过 Ctor = baseCtor.extend(Ctor)

+ <font color="yellow">这个还没学</font>

#### patch

+ ```patch``` 方法通过 ```createPatchFunction``` 获得，这个工厂函数需要传入的参数包括，```Dom``` 节点的操作方法(<font color="red">nodeOps</font>)，```Dom```属性设置方法<font color="red">modules</font>。<font color="red">最终返回一个 patch 方法</font>
	
	- ```Dom``` 节点的操作方法如，节点插入，删除，查找父节点等
	~~~javascript
	document.createComment
	document.createTextNode
	setTextContent
	removeChild
	appendChild
	tagName
	~~~
	
	- ```Dom``` 节点属性的操作包括: <font color="red">类（klass）, attrs, props, style, events, transition</font>

+ patch 方法如下
~~~javascript
  return function patch (oldVnode, vnode, hydrating, removeOnly) {
    if (isUndef(vnode)) {
      if (isDef(oldVnode)) invokeDestroyHook(oldVnode)
      return
    }

    let isInitialPatch = false
    const insertedVnodeQueue = []

    // 没有传入旧节点，只有新节点表示要创建一个新的节点
    if (isUndef(oldVnode)) {
      // 创建一个新的节点如 初始化一个组件
      isInitialPatch = true
      createElm(vnode, insertedVnodeQueue)
    } 

    // 旧节点与新节点都存在，
    // 	1: 旧节点是虚拟节点 并且新旧两个节点是同一个节点，需要对这两个节点进行 diff
    //	2: 旧节点不是虚拟节点，将旧节点转换为虚拟节点，
    else {
      const isRealElement = isDef(oldVnode.nodeType)
      if (!isRealElement && sameVnode(oldVnode, vnode)) {
        // patch existing root node
        patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly)
      } 
      else {
        if (isRealElement) {
        	// 将真实 dom 节点转换为虚拟节点，通过实例化一个虚拟节点，将DOM 节点挂载到 该虚拟节点的 elm 属性上
          oldVnode = emptyNodeAt(oldVnode)
        }

        const oldElm = oldVnode.elm
        const parentElm = nodeOps.parentNode(oldElm)

        // 将新的 虚拟节点 elm 上挂载 Dom 节点
        createElm(
          vnode,
          insertedVnodeQueue,
          oldElm._leaveCb ? null : parentElm,
          nodeOps.nextSibling(oldElm)
        )

        // 递归的更新新节点的父级节点
        if (isDef(vnode.parent)) {
          let ancestor = vnode.parent
          const patchable = isPatchable(vnode)
          while (ancestor) {
            for (let i = 0; i < cbs.destroy.length; ++i) {
            // destroy: [ 'registerRef', 'unbindDirectives' ]
            // 移出 ref ， 移出指令
              cbs.destroy[i](ancestor)
            }
            // dom 节点挂载
            ancestor.elm = vnode.elm
            if (patchable) {
              for (let i = 0; i < cbs.create.length; ++i) {
              	//   create: [
				//     'updateAttrs',
				//     'updateClass',
				//     'updateDOMListeners',
				//     'updateDOMProps',
				//     'updateStyle',
				//     '_enter',
				//     'refCreateFun',
				//     'updateDirectives'
				//   ],
				// 更新节点上的 属性样式等
                cbs.create[i](emptyNode, ancestor)
              }
            } else {
              registerRef(ancestor)
            }
            ancestor = ancestor.parent
          }
        }

        // 删除旧节点
        if (isDef(parentElm)) {
          removeVnodes([oldVnode], 0, 0)
        } else if (isDef(oldVnode.tag)) {
          invokeDestroyHook(oldVnode)
        }
      }
    }

    // 返回虚拟节点转换后的 Dom 节点
    invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch)
    return vnode.elm
  }
~~~

+ *这里需要说明以下 <font color="purple">cbs对象</font>*

	- ```cbs``` 对象是根据 ```createPatchFunction``` 传入的 ```Dom``` 属性操作对象**modules**生成的
	~~~javascript
	const hooks = ['create', 'activate', 'update', 'remove', 'destroy']
	for (i = 0; i < hooks.length; ++i) {
	  cbs[hooks[i]] = []
	  for (j = 0; j < modules.length; ++j) {
	    if (isDef(modules[j][hooks[i]])) {
	      cbs[hooks[i]].push(modules[j][hooks[i]])
	    }
	  }
	}
	// 最终形式如下（有些字段不与vue 一致）
	// {
	//   create: [
	//     'updateAttrs',
	//     'updateClass',
	//     'updateDOMListeners',
	//     'updateDOMProps',
	//     'updateStyle',
	//     '_enter',
	//     'refCreateFun',
	//     'updateDirectives'
	//   ],
	//   activate: [ '_enter' ],
	//   update: [
	//     'updateAttrs',
	//     'updateClass',
	//     'updateDOMListeners',
	//     'updateDOMProps',
	//     'updateStyle',
	//     'refUpdateFun',
	//     'updateDirectives'
	//   ],
	//   remove: [],
	//   destroy: [ 'registerRef', 'unbindDirectives' ]
	// }

	~~~

#### patchVnode

+ 这个方法将对新旧虚拟节点进行 diff ,用于获取dom 变化的具体情况

+ 其中比较关键的部分源码如下，通过 dom 操作（**nodeOps**）更新节点数据
~~~javascript
// 新节点不是 文本类型的节点
if (isUndef(vnode.text)) {
	// 新旧节点都存在子节点，通过 updateChildren 更新两个节点
  if (isDef(oldCh) && isDef(ch)) {
    if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly)
  } 

  else if (isDef(ch)) {
  	// 新节点存在子节点，旧节点不存在子节点，直接将子节点添加到 
    if (isDef(oldVnode.text)) nodeOps.setTextContent(elm, '')

   	// createElm(vnodes[startIdx], insertedVnodeQueue, parentElm, refElm, false, vnodes, startIdx)
    // addVnodes 遍历子节点，根据子虚拟节点生成Dom 节点
    addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue)
  } 
  // 新节点不存在子节点，旧节点存在，直接删除旧节点的子节点
  else if (isDef(oldCh)) {
  	// removeNode ==> nodeOps.removeChild(parent, el)
  	// 递归删除子节点 最终执行 nodeOps.removeChild(parent, el)
    removeVnodes(oldCh, 0, oldCh.length - 1)
  } 

  // 旧节点是文本类型，要删除新节点上的 text 
  else if (isDef(oldVnode.text)) {
    nodeOps.setTextContent(elm, '')
  }
} 
 // 如果是文本类型的节点就更新节点的文本 node.textContent = text
else if (oldVnode.text !== vnode.text) {
  nodeOps.setTextContent(elm, vnode.text)
}
~~~

#### updateChildren

+ 新旧节点的 diff 方法

	- 

+ 讲到这里有必要讲一下 如何根据虚拟节点，将生成的  dom节点 挂载到 该vnode 上，即 **createElm** 方法的运作方式
~~~javascript
function createElm (
    vnode,
    insertedVnodeQueue,
    parentElm,
    refElm,
    nested,
    ownerArray,
    index
  ) {
    if (isDef(vnode.elm) && isDef(ownerArray)) {
    	// 拷贝一下要操作的虚拟节点
      vnode = ownerArray[index] = cloneVNode(vnode)
    }

    vnode.isRootInsert = !nested // for transition enter check

    const data = vnode.data
    const children = vnode.children
    const tag = vnode.tag
    // 不是文本类型的节点通过 Dom 操作去创建dom 节点
    if (isDef(tag)) {
      vnode.elm = vnode.ns
        ? nodeOps.createElementNS(vnode.ns, tag)
        : nodeOps.createElement(tag, vnode)
      setScope(vnode)

      if (__WEEX__) {
      	// ...
      } 
      else {
        // 创建 Vnode 挂载的子节点
        //  nodeOps.appendChild(vnode.elm, nodeOps.createTextNode(String(vnode.text)))
        createChildren(vnode, children, insertedVnodeQueue)
        if (isDef(data)) {
          invokeCreateHooks(vnode, insertedVnodeQueue)
        }
        // nodeOps.appendChild(parent, elm)
        insert(parentElm, vnode.elm, refElm)
      }
    } 

    // 是注释类型的节点
    else if (isTrue(vnode.isComment)) {
      vnode.elm = nodeOps.createComment(vnode.text)
      insert(parentElm, vnode.elm, refElm)
    } 
    // 文本类型的节点
    else {
      vnode.elm = nodeOps.createTextNode(vnode.text)
      insert(parentElm, vnode.elm, refElm)
    }
  }
~~~


 ## 以上 Vue 渲染主流程介绍完毕具体流程见流程图