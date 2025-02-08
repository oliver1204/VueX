## 前言
目前 `Vue` 官方推荐的是 `Pinia` 状态管理库, `Pinia`的设计理念、使用方式和性能都有所不同。Vuex 适用于 `Vue 2` 和 `Vue 3`，而 `Pinia` 是 `Vue 3` 官方推荐的新一代状态管理库，旨在更简单、灵活和高效。

## 1. 概括

`Vuex`是`Vue`的状态管理库，核心概念包括`state`、`mutations`、`actions`、`getters`，还有模块化的`modules`。

首先，`Vuex`作为一个全局的单例模式，确保整个应用只有一个`store实例`。这个`store`里包含应用的状态，以及修改状态的方法。

`Mutations`是同步的，而`Actions`可以处理异步操作。当组件需要修改状态时，应该通过提交`mutation`，而不是直接修改`state`。因为`Vuex`需要跟踪状态的变化，确保每次变化都是通过`mutation`，这样`devtools`可以记录每次状态变更，方便调试。另外，`actions`的存在是为了处理异步操作，因为`mutation`必须是同步的，否则会导致状态变更难以追踪。

`getters`的实现原理，应该是基于Vue的计算属性，缓存结果，只有当依赖的`state`变化时才会重新计算。这样在组件中访问getters时，可以高效地获取派生状态。

`Vuex`通过`Vue`的插件机制，在install方法中通过`vue.mix()`将`store`实例注入到所有子组件中，这样每个组件都可以通过`this.$store`访问到`store`。这也是为什么需要在`Vue`实例化时传入`store`选项的原因。

## 2.核心模块实现原理

### 2.1  State：响应式状态存储
在 Store 初始化时，Vuex 会创建一个 Vue 实例，将 state 作为其 data, 通过 this.$store.state.xxx 访问时，实际是访问 this._vm._data.$$state。

```js
this._vm = new Vue({
  data: { $$state: state }, // $$state 会被 Vue 转换为响应式
});

```

### 2.2 Mutations：同步状态变更

`Mutations` 唯一允许直接修改 state 的入口。必须是同步函数，确保 DevTools 能准确追踪状态变化。

```js
// 注册
mutations: {
  SET_DATA(state, payload) { ... }
}
// 触发
store.commit('SET_DATA', payload);
```

### 2.3 Actions：处理异步操作
`Actions`可包含异步逻辑（如 API 请求），通过 commit 触发 Mutations。通过返回 Promise 支持异步流程控制。示例：
```js
actions: {
  fetchData({ commit }) {
    api.getData().then(res => commit('SET_DATA', res));
  }
}
```

### 2.4 Getters：计算属性
Getters 基于 Vue 的计算属性（computed），结果会被缓存，仅在依赖的 state 变化时重新计算。

### 2.5  Modules：模块化
客户端使用时通过树形结构组织模块，每个模块可包含独立的 `state/mutations/actions/getters`。通过 `namespaced: true` 启用命名空间，避免命名冲突。vuex 内部Store 初始化时递归合并嵌套模块的配置，最终形成扁平化的全局状态树。

## 3. vuex 和 vue 的关系

在 Vue 2 中，Vuex 会通过 Vue.mixin 来全局混入一个 beforeCreate 钩子函数，根组件通过 options.store 获取 store，而子组件则通过 $parent 链式查找，直到找到根组件的 store。

```js
Vue.mixin({
  beforeCreate() {
    if (this.$options.store) {
      // 根组件
      this.$store = this.$options.store;
    } else {
      // 子组件
      this.$store = this.$parent.$store;
    }
  }
});
```

在 Vue 3, Vuex 4 中，由于 Composition API 的引入和 provide/inject 的底层实现变化, 不再有 $parent 的隐式依赖.在 Vuex 的 install 方法中，会通过 app.provide 全局注入 Store：

```js
// Vuex 4 源码简化版
export class Store {
  install(app, injectKey) {
    // 通过 provide 注入 store 实例
    app.provide(injectKey || storeKey, this);
    // 兼容性：同时挂载到全局属性（如 this.$store）
    app.config.globalProperties.$store = this;
  }
}
```

子组件接收 Store, 子组件通过 inject 直接获取 Store 实例，无需通过层层传递 props：

```js
// 子组件中获取 Store（Composition API）
import { inject } from 'vue';
import { storeKey } from 'vuex';

export default {
  setup() {
    const store = inject(storeKey);
    return { store };
  }
}
```

顺便说一下，vue 的 `provide` 和 `inject` 是父子组件通讯的一种方式，但是它的明显缺点是不支持响应式。

###  4 Vue 的 Mixin

Vue 的 Mixin（混入） 是一种代码复用机制，允许将多个组件的公共逻辑（如数据、方法、生命周期钩子等）抽离成一个独立模块，再混入到组件中。

### 4.1 Mixin 的核心原理
#### 1. 合并策略
当组件和 Mixin 存在同名属性或方法时，Vue 会按特定规则合并：

* 数据对象 (data): 递归合并同名属性，组件数据优先级高于 Mixin。

```js
// Mixin
data() { return { name: 'Mixin', age: 20 }; }

// 组件
data() { return { name: 'Component', gender: 'male' }; }

// 合并结果
{ name: 'Component', age: 20, gender: 'male' }

```

* 生命周期钩子 (如 created): 合并为一个数组，Mixin 的钩子先执行，组件的钩子后执行。

```js
// Mixin
created() { console.log('Mixin created'); }

// 组件
created() { console.log('Component created'); }

// 执行顺序：
// 1. 'Mixin created'
// 2. 'Component created'
```
* 方法/计算属性/侦听器 (methods, computed, watch):  同名时，组件中的方法会覆盖 Mixin 的方法。

```js
// Mixin
methods: { log() { console.log('Mixin'); } }

// 组件
methods: { log() { console.log('Component'); } }

// 调用 this.log() 输出 'Component'
```

#### 2. 全局混入
通过 Vue.mixin() 全局混入的 Mixin 会影响所有后续创建的 Vue 实例（慎用）：

```js
Vue.mixin({
  created() { console.log('Global Mixin'); }
});
```

#### 3. Mixin 的优缺点
* 优点：
    * 代码复用：减少重复代码，提高可维护性。
    * 逻辑解耦：将通用逻辑与组件分离，结构更清晰。

* 缺点：
  * 命名冲突：多个 Mixin 或组件同名属性可能覆盖。
  * 隐式依赖：逻辑来源不透明，调试困难。
  * 维护成本：过度使用 Mixin 会导致代码复杂度上升。
  
### 4.2 替代方案（Vue 3 推荐）
在 Vue 3 中，Composition API 是更推荐的代码复用方式，通过函数封装逻辑，避免 Mixin 的缺点：

```js
/ 使用 Composition API 替代 Mixin
// composables/useLogger.js
export function useLogger(name) {
  const log = (message) => {
    console.log(`[${name}]: ${message}`);
  };
  return { log };
}

// 组件中使用
import { useLogger } from './composables/useLogger';
export default {
  name: 'MyComponent',
  setup() {
    const { log } = useLogger('MyComponent');
    log('Hello from Composition API!'); // 输出 [MyComponent]: Hello from Composition API!
    return { log };
  }
};
```

## 5. 手写 EventBus 类 -- 核心是理解 `发布-订阅模式`。
在 Vue 2 中，由于没有官方推荐的事件总线，开发者常常自己创建一个中央事件总线来实现跨组件通信。Vue 3 虽然推荐使用 provide/inject 或者第三方库.

```js
class EventBus {
  constructor() {
    this.events = {}; // 存储事件及回调 { eventName: [callback1, callback2] }
  }

  // 订阅事件
  on(eventName, callback) {
    if (!this.events[eventName]) {
      this.events[eventName] = [];
    }
    this.events[eventName].push(callback);
  }

  // 触发事件
  emit(eventName, ...args) {
    const callbacks = this.events[eventName];
    if (callbacks) {
      callbacks.forEach(cb => cb(...args));
    }
  }

  // 取消订阅
  off(eventName, callback) {
    const callbacks = this.events[eventName];
    if (callbacks) {
      if (callback) {
        // 移除特定回调
        this.events[eventName] = callbacks.filter(cb => cb !== callback);
      } else {
        // 移除事件的所有回调
        delete this.events[eventName];
      }
    }
  }

  // 一次性事件
  once(eventName, callback) {
    const wrapper = (...args) => {
      callback(...args);
      this.off(eventName, wrapper);
    };
    this.on(eventName, wrapper);
  }
}

// 导出一个单例实例（模仿 Vue 2 中常用的 $bus）
const bus = new EventBus();
export default bus;
```

#### 5.1 使用示例
1. 组件 A（发送事件）

```js
import bus from './eventBus';

export default {
  methods: {
    sendMessage() {
      bus.emit('message', { text: 'Hello from Component A!' });
    }
  }
}
```
2. 组件 B（接收事件）

```js
import bus from './eventBus';

export default {
  created() {
    bus.on('message', this.handleMessage);
  },
  beforeUnmount() {
    bus.off('message', this.handleMessage); // 避免内存泄漏
  },
  methods: {
    handleMessage(payload) {
      console.log('Received:', payload.text); // Received: Hello from Component A!
    }
  }
}
```