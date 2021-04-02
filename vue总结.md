## 一、VUEX

​	Vuex是专门为vue组件化思想带来的组件间通信问题提供的解决方案，主要解决以下两个问题：

1. 多个视图依赖同一状态
2. 来自不同视图的行为需要变更同一状态

核心：

- state：简单理解为vue维持的全局变量（状态）
- getter：获取state中的状态或方法，可以在取出前对数据进行二次处理
- mutation：改变state中的状态的唯一方法，只能是同步操作
- action：改变state中的状态的方法，action提交的是mutation而不是直接改变状态，与mutation不同的是action可以包含任意异步操作
- module：将复杂应用中的状态分模块保存

### vue单页面应用刷新网页后vuex的state数据丢失的解决方案

产生原因：因为store里的数据是保存在运行内存中的,当页面刷新时，页面会重新加载vue实例，store里面的数据就会被重新赋值。

解决思路：将state里的数据保存一份到本地存储(`localStorage`、`sessionStorage`、`cookie`）中

解决过程：

1. ​	选择合适的客户端存储。使用sessionStorage，原因是vue是单页面应用，操作都是在一个页面跳转路由，还有就是sessionStorage可以保证打开页面时sessionStorage的数据为空，而如果是localStorage则会读取上一次打开页面的数据。
2. ​    保存state里的数据。在每次页面刷新之前先将state数据保存到sessionStorage然后再刷新页面，使用beforeunload事件在页面刷新时触发，放在app.vue入口组件中，保证每次刷新页面都可以触发

```javascript
export default {
	name: 'app',
    created() {
        // 在页面加载时读取sessionStorage里的状态信息
        if (sessionStorage.getItem("store")) {
            this.$store.replace(Object.assign({}, this.$store.state, JSON.parse(sessionStorage.getItem("store"))))
        }
        
        // 在页面刷新时将vuex里的信息保存到sessionStorage里
        window.addEventListener('beforeunload', () => {
            sessionStorage.setItem('store', JSON.stringify(this.$store.state))
        })
    }
}
```

## 二、组件通信方法

### 1、props和$emit

 父子组件传值时，父组件传递的参数，数组和对象，子组件接受之后可以直接进行修改，并且会传递给父组件相应的值也会修改。

如果传递的值是字符串，直接修改会报错。

​	父组件向子组件传值  props：

```vue
// parent.vue 父组件：
<template>
	<div>
        <child :arrs="arrs"></child>
    </div>
</template>
<script>
import child from './components/child'
export default {
    name: "parent",
    components: { child },
    data() {
        return {
        	arrs: [1,2,3,4]
        }
    }
}
</script>

// child.vue 子组件：
<template>
	<div>
        <ul>
            <li v-for="arr in arrs" :key="arr">{{arr}}</li>
    	</ul>
    </div>
</template>
<script>
export default {
    name: "child",
    props: {
        arrs: {
            type: Array,
            required: true
        }
    }
}
</script>
```

​	子组件向父组件传值 $emit：

```vue
// 子组件 child.vue
<template>
	<div>
        <h1 @click="changeText">{{title}}</h1>
    </div>
</template>
<script>
export default {
    name: "child",
    data() {
        return {
            title: "点击给父组件传值"
        }
    },
    methods: {
        changeText() {
            this.$emit('changeText', "我是子组件传给父组件的值")
        }
    }
}
</script>

// 父组件 parent.vue
<template>
	<div>
        <child @changeText="updateTitle"></child>
    </div>
</template>
<script>
import child from './components/child'
export default {
    name: "parent",
    components: { child },
    data() {
        return {
            title: "传的值"
        }
    },
    methods: {
        updateTitle(data) {
            this.title = data
        }
    }
}
</script>
```

### 2、$attrs和$listeners

​	A组件有子组件B，B组件有子组件C，A需要传递数据给C组件

```vue
// A组件：
<template>
	<div>
        <B :messageB="messageB" :messageC="messageC" v-on:getBData="getBData(messageB)"></B>
    </div>
</template>
<script>
import B from "./components/B"
export default {
    name: 'A',
    components: { B },
    data() {
        return {
            messageB: "hello B",
            messageC: "hello C"
        }
    },
    methods: {
        getBData(val) {
            console.log('来自B组件的数据'+val)
        },
        getCData(val) {
            console.log('来自C组件的数据'+val)
        }
    }
}
</script>

// B组件
<template>
	<input type="text" v-model="bMessage" @input="passData(bMessage)">
	<C v-bind="$attrs" v-on="$listener"></C>
</template>
<script>
import C from './components/C'
export default {
    name: 'B',
    components: { C },
    props: {
        messageB: {
            type: String,
            required: true
        }
    }
    data() {
        return {
            bMessage: this.messageB
        }
    },
    methods: {
        passData(val) {
            this.$emit('getBData', val)
        }
    }
}
</script>

// C组件
<template>
	<input type="text" v-model="$attrs.messageC" @input="passCData($attrs.messageC)">
</template>
<script>
export default {
    name: 'C',
    methods: {
        passCData(val) {
            this.$emit('getCData', val)
        }
    }
}
</script>
```

### 3、中央事件总线

​	适用于不是父子关系的组件，使用中央事件总线的方式，新建一个Vue事件bus对象，然后通过bus.$emit触发事件，bus.$on监听触发的事件。

```vue
// main.js
var bus = new Vue()
export default bus

// brother1.vue
<template>
	<div>
        <input type="text" v-model="message" @input="passData(message)">
    </div>
</template>
<script>
import bus from '../../../main.js'
export default {
    name: "brother1",
    data() {
        return {
            message: "hello brother1"
        }
    },
    methods: {
        passData(val) {
            bus.$emit('globalEvent',val)
        }
	}
}
</script>

// brother2.vue
<template>
	<div>
        <p>btother1传递的数据：{{ brotherMessage }}</p>
    </div>
</template>
<script>
import bus from '../../../main.js'
export default {
    name: "brother2",
    data() {
        return {
            brotherMessage: ""
        }
    },
    mounted() {
        bus.$on('globalEvent',(val) => {
            this.brotherMessage = val;
        })
    }
}
</script>
```

### 4、provide和inject

​	父组件通过provide来提供变量，然后在子组件中通过inject来注入变量。不论子组件有多深，只要调用了inject就可以注入provide中的数据。

```vue
// parent.vue
<template>
	<div>
        <grandsun></grandsun>
    </div>
</template>
<script>
import grandsun from './components/components/grandsun'
export default {
    name: "parent",
    components: { grandsun },
    provide: {
        for: "test测试"
    }
}
</script>

// grandsun.vue
<template>
	<div>
        <p>{{ message }}</p>
    </div>
</template>
<script>
export default {
    name: "gransun",
    inject: ['for'],
    data() {
        return {
        	message: this.for
        }
    }
}
</script>
```

### 5、v-model

​	父组件通过v-model传递值给子组件时，会自动传递一个value的prop属性，在子组件中通过this.$emit('input',val)自动修改v-model绑定的值。

```vue
// 父组件
<template>
	<div>
    	<child v-model="message"></child>
    </div>
</template>
<script>
import child from './components/child'
export default {
    name: "parent",
    components: {child},
    data() {
        return {
            message: "hello child"
        }
    }
}
</script>

//子组件
<template>
	<div>
        <input type="text" v-model="childMsg" @change="changeValue">
    </div>
</template>
<script>
export default {
    name: "child",
    props: {
        value: String
    }
    data() {
        return {
            childMsg: this.value
        }
    },
    methods: {
        changeValue() {
            this.$emit('input', this.childMsg)
        }
    }
}
</script>
```

### 6、VueX

​	如果业务逻辑复杂，可以使用vuex将公共数据抽离出来，其他组件可以对公共数据进行读写操作。

## 三、this.$nextTick()方法

​	$nextTick() 方法是在下次DOM更新循环结束之后执行延迟回调。

​	使用：

1. vue生命周期的 created() 钩子函数进行的DOM操作一定要放在vue.nextTick() 的回调函数中，原因是在created钩子函数执行的时候DOM其实并未进行任何渲染，而此时进行DOM操作无异于徒劳，所以此处一定要将DOM操作的代码放进$nextTick() 的回调函数中。mounted钩子函数下操作任何DOM不会出现该问题。
2. 在数据变化后要执行的某个操作，而这个操作需要使用随数据改变而改变的DOM结构的时候，这个操作都应该放进nextTick() 的回调函数中。

## 四、Vue生命周期

### vue2.0

- beforeCreate（创建前）：在实例初始化后，数据观测和事件配置之前被调用，此时组件的选项对象还未创建，el和data并未初始化，因此无法访问methods，data，computed等上的方法和数据。
- created（创建后）：实例已经创建完成后被调用，这一步实例已完成以下配置：数据观测、属性和方法的运算，watch/event事件回调，完成了data数据初始化。然而，挂载阶段还未开始，$el属性目前不可见；可以调用methods中的方法，改变data中的数据，并且修改可以通过vue的响应式绑定体现在页面上、获取computed的计算属性等。
- beforeMount：挂载开始之前被调用，相关的render函数首次被调用（虚拟DOM），实例已完成以下的配置：编译模板，把data里的数据和模板生成html，完成el和data的初始化，注意此时还没有挂载html到页面。
- mounted：挂载完成，模板的html渲染到html页面中，此时可以做一些ajax操作，mounted只会执行一次
- beforeUpdate：在数据更新前被调用，发生在虚拟DOM重新渲染之前，可以在该钩子函数中进一步的更改状态，不会触发附加的重渲染过程。
- updated（更新后）：调用时，组件DOM已经更新，所以可以执行依赖于DOM的操作，大多情况下，应避免在此期间更改状态，因为这可能会导致更新无限循环，该钩子在服务器端渲染期间不被调用。
- beforeDestory：在实例销毁之前调用，实例仍然完全可用，这一步还可以用this来获取实例；一般在这一步做一些重置的操作，如清除组件的计时器和监听的dom事件
- destory：在实例销毁后调用，调用后，所有的事件监听器会被移除，所有的子实例也会被销毁，该钩子在服务器端渲染期间不被调用。

### vue3.0

- beforeCreate和created ->  setup()
- beforeMount  -> onBeforeMount
- mounted -> onMounted
- beforeUpdate -> onBeforeYpdate
- updated -> onUpdated
- beforeDestroy -> onBeforeUnmount
- destroyed -> onUmounted
- errorCaptured -> onErrorCaptured

## 五、Computed属性

​	计算属性，对数据二次加工，实时改变视图层数据

## 五、vue的hash路由和history路由

### 区别

​	hash模式url带着#号，在开发中默认使用这个模式，如果用户考虑url的规范，就需要使用history模式，但是使用history模式有一个问题就是在访问二级页面时做刷新处理，会出现404，这个时候需要和后端人员配合，配置apache或nginx的url重定向，重定向到首页路由。

​	hash：url显示会有‘#’，回车刷新可以加载到hash值对应页面，支持低版本浏览器和IE浏览器。

​	history：url显示无#，回车刷新一般会报404，是H5新推出的API

	### hash模式

​	路由的hash模式利用了window可以监听onhashchange事件，也就是说如果url种的hash值有变化，前端可以做到监听并做一些响应，这样，即使前端并没有发起http请求，也能够找到对应页面的代码进行按需加载。

```js
window.onhashchange = function(event) {
    console.log(event)
}
```

### history模式

​	history模式主要使用H5的pushState() 和 replaceState() 这两个API来实现的，pushState可以改变url地址且不会发送请求，replaceState可以读取历史记录栈，还可以对浏览器记录进行修改。

```javascript
window.history.pushState(stateObject, title, URL)
window.history.replaceState(stateObject, title, URL)
```

### 将默认的hash模式改为history模式

```javascript
export default new VueRouter({
	linkActiveClass: 'mui-active',
	mode: 'history',
    routes: [{path:'/',redirect:{name:'home'}},
            {name:'home',path:'/home',component:homeVue}]
})
```

## 七、vuex数据流

​	单向数据流，指只能从一个方向来修改状态。

​	vuex的单向数据流流程：组件中触发Action，Action中提交Mutations，Mutations修改state。组件根据State或Getters来渲染页面。

## 八、单页面和多页面的区别

​	单页面（SPA）：由一个外壳页面和多个片段页面组成；页面跳转仅片段页面之间的切换，共用外壳页面；局部刷新；url会有#；页面间片段切换快，用户体验好；易实现转场动画；页面间数据传递依赖url，cookie，localstorage等，实现麻烦；搜索引擎优化需要单独方案，不利于SEO；开发成本较高；维护成本较低。

​	多页面（MPA）：由多个完整的html页面组成；页面跳转是由一个完整页面跳到另一个完整页面；整页刷新；页面间切换慢，不流畅，用户体验差；不容易实现转场动画；在同一页面内，页面间传递数据容易；搜索引擎优化直接可以实现；开发成本较低，页面重复代码较多；维护成本复杂。

## 九 如何快速搭建环境并跑起来

- 查看是否有 `dockerfile` ，如果有就跟着 `dockerfile` 跑命令
- 查看是否有 `CI/CD` ，如果有就跟着 `CI/CD` 部署的脚本跑命令

dockerfile：