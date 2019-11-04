> 前言：上一节最后稍微提到了```Vue```内置组件的相关内容，从这一节开始，将会对某个具体的内置组件进行分析。首先是```keep-alive```，它是我们日常开发中经常使用的组件，我们在不同组件间切换时，经常要求保持组件的状态，以避免重复渲染组件造成的性能损耗，而```keep-alive```经常和上一节介绍的动态组件结合起来使用。由于内容过多，```keep-alive```的源码分析将分为上下两部分，这一节主要围绕```keep-alive```的首次渲染展开。



## 13.1 基本用法
`keep-alive`的使用只需要在动态组件的最外层添加标签即可。

```html
<div id="app">
    <button @click="changeTabs('child1')">child1</button>
    <button @click="changeTabs('child2')">child2</button>
    <keep-alive>
        <component :is="chooseTabs">
        </component>
    </keep-alive>
</div>
```


```js

var child1 = {
    template: '<div><button @click="add">add</button><p>{{num}}</p></div>',
    data() {
        return {
            num: 1
        }
    },
    methods: {
        add() {
            this.num++
        }
    },
}
var child2 = {
    template: '<div>child2</div>'
}
var vm = new Vue({
    el: '#app',
    components: {
        child1,
        child2,
    },
    data() {
        return {
            chooseTabs: 'child1',
        }
    },
    methods: {
        changeTabs(tab) {
            this.chooseTabs = tab;
        }
    }
})
```



简单的结果如下，动态组件在```child1,child2```之间来回切换，当第二次切到```child1```时，```child1```保留着原来的数据状态，```num = 5```。

![](./img/13.1.gif)

## 13.2 从模板编译到生成vnode
按照以往分析的经验，我们会从模板的解析开始说起，第一个疑问便是：内置组件和普通组件在编译过程有区别吗？答案是没有的，不管是内置的还是用户定义组件，本质上组件在模板编译成```render```函数的处理方式是一致的，这里的细节不展开分析，有疑惑的可以参考前几节的原理分析。最终针对```keep-alive```的```render```函数的结果如下：

```js
with(this){···_c('keep-alive',{attrs:{"include":"child2"}},[_c(chooseTabs,{tag:"component"})],1)}
```


有了```render```函数，接下来从子开始到父会执行生成```Vnode```对象的过程，```_c('keep-alive'···)```的处理，会执行```createElement```生成组件```Vnode```,其中由于```keep-alive```是组件，所以会调用```createComponent```函数去创建子组件```Vnode```,```createComponent```之前也有分析过，这个环节和创建普通组件```Vnode```不同之处在于，```keep-alive```的```Vnode```会剔除多余的属性内容，**由于```keep-alive```除了```slot```属性之外，其他属性在组件内部并没有意义，例如```class```样式，```<keep-alive clas="test"></keep-alive>```等，所以在```Vnode```层剔除掉多余的属性是有意义的。而```<keep-alive slot="test">```的写法在2.6以上的版本也已经被废弃。**(其中```abstract```作为抽象组件的标志，以及其作用我们后面会讲到)

```js
// 创建子组件Vnode过程
function createComponent(Ctordata,context,children,tag) {
    // abstract是内置组件(抽象组件)的标志
    if (isTrue(Ctor.options.abstract)) {
        // 只保留slot属性，其他标签属性都被移除，在vnode对象上不再存在
        var slot = data.slot;
        data = {};
        if (slot) {
            data.slot = slot;
        }
    }
}
```

## 13.3 初次渲染

`keep-alive`之所以特别，是因为它不会重复渲染相同的组件，只会利用初次渲染保留的缓存去更新节点。所以为了全面了解它的实现原理，我们需要从```keep-alive```的首次渲染开始说起。


### 13.3.1 流程图
为了理清楚流程，我大致画了一个流程图，流程图大致覆盖了初始渲染```keep-alive```所执行的过程，接下来会照着这个过程进行源码分析。

![](./img/13.2.png)

和渲染普通组件相同的是，```Vue```会拿到前面生成的```Vnode```对象执行真实节点创建的过程，也就是熟悉的```patch```过程,```patch```执行阶段会调用```createElm```创建真实```dom```，在创建节点途中，```keep-alive```的```vnode```对象会被认定是一个组件```Vnode```,因此针对组件```Vnode```又会执行```createComponent```函数，它会对```keep-alive```组件进行初始化和实例化。

```js
function createComponent (vnode, insertedVnodeQueue, parentElm, refElm) {
      var i = vnode.data;
      if (isDef(i)) {
        // isReactivated用来判断组件是否缓存。
        var isReactivated = isDef(vnode.componentInstance) && i.keepAlive;
        if (isDef(i = i.hook) && isDef(i = i.init)) {
            // 执行组件初始化的内部钩子 init
          i(vnode, false /* hydrating */);
        }
        if (isDef(vnode.componentInstance)) {
          // 其中一个作用是保留真实dom到vnode中
          initComponent(vnode, insertedVnodeQueue);
          insert(parentElm, vnode.elm, refElm);
          if (isTrue(isReactivated)) {
            reactivateComponent(vnode, insertedVnodeQueue, parentElm, refElm);
          }
          return true
        }
      }
    }
```
`keep-alive`组件会先调用内部钩子```init```方法进行初始化操作，我们先看看```init```过程做了什么操作。

```js
// 组件内部钩子
var componentVNodeHooks = {
    init: function init (vnode, hydrating) {
      if (
        vnode.componentInstance &&
        !vnode.componentInstance._isDestroyed &&
        vnode.data.keepAlive
      ) {
        // kept-alive components, treat as a patch
        var mountedNode = vnode; // work around flow
        componentVNodeHooks.prepatch(mountedNode, mountedNode);
      } else {
          // 将组件实例赋值给vnode的componentInstance属性
        var child = vnode.componentInstance = createComponentInstanceForVnode(
          vnode,
          activeInstance
        );
        child.$mount(hydrating ? vnode.elm : undefined, hydrating);
      }
    },
    // 后面分析
    prepatch： function() {}
}
```
第一次执行，很明显组件```vnode```没有```componentInstance```属性，```vnode.data.keepAlive```也没有值，所以会**调用```createComponentInstanceForVnode```方法进行组件实例化并将组件实例赋值给```vnode```的```componentInstance```属性，** 最终执行组件实例的```$mount```方法进行实例挂载。

`createComponentInstanceForVnode`就是组件实例化的过程，而组件实例化从系列的第一篇就开始说了，无非就是一系列选项合并，初始化事件，生命周期等初始化操作。

```js
function createComponentInstanceForVnode (vnode, parent) {
    var options = {
      _isComponent: true,
      _parentVnode: vnode,
      parent: parent
    };
    // 内联模板的处理，忽略这部分代码
    ···
    // 执行vue子组件实例化
    return new vnode.componentOptions.Ctor(options)
  }
```

### 13.3.2 内置组件选项
我们在使用组件的时候经常利用对象的形式定义组件选项，包括```data,method,computed```等，并在父组件或根组件中注册。```keep-alive```同样遵循这个道理，内置两字也说明了```keep-alive```是在```Vue```源码中内置好的选项配置，并且也已经注册到全局，这一部分的源码可以参考组态组件小节末尾对内置组件构造器和注册过程的介绍。这一部分我们重点关注一下```keep-alive```的具体选项。

```js
// keepalive组件选项
  var KeepAlive = {
    name: 'keep-alive',
    // 抽象组件的标志
    abstract: true,
    // keep-alive允许使用的props
    props: {
      include: patternTypes,
      exclude: patternTypes,
      max: [String, Number]
    },

    created: function created () {
      // 缓存组件vnode
      this.cache = Object.create(null);
      // 缓存组件名
      this.keys = [];
    },

    destroyed: function destroyed () {
      for (var key in this.cache) {
        pruneCacheEntry(this.cache, key, this.keys);
      }
    },

    mounted: function mounted () {
      var this$1 = this;
      // 动态include和exclude
      // 对include exclue的监听
      this.$watch('include', function (val) {
        pruneCache(this$1, function (name) { return matches(val, name); });
      });
      this.$watch('exclude', function (val) {
        pruneCache(this$1, function (name) { return !matches(val, name); });
      });
    },
    // keep-alive的渲染函数
    render: function render () {
      // 拿到keep-alive下插槽的值
      var slot = this.$slots.default;
      // 第一个vnode节点
      var vnode = getFirstComponentChild(slot);
      // 拿到第一个组件实例
      var componentOptions = vnode && vnode.componentOptions;
      // keep-alive的第一个子组件实例存在
      if (componentOptions) {
        // check pattern
        //拿到第一个vnode节点的name
        var name = getComponentName(componentOptions);
        var ref = this;
        var include = ref.include;
        var exclude = ref.exclude;
        // 通过判断子组件是否满足缓存匹配
        if (
          // not included
          (include && (!name || !matches(include, name))) ||
          // excluded
          (exclude && name && matches(exclude, name))
        ) {
          return vnode
        }

        var ref$1 = this;
        var cache = ref$1.cache;
        var keys = ref$1.keys;
        var key = vnode.key == null
          ? componentOptions.Ctor.cid + (componentOptions.tag ? ("::" + (componentOptions.tag)) : '')
          : vnode.key;
          // 再次命中缓存
        if (cache[key]) {
          vnode.componentInstance = cache[key].componentInstance;
          // make current key freshest
          remove(keys, key);
          keys.push(key);
        } else {
        // 初次渲染时，将vnode缓存
          cache[key] = vnode;
          keys.push(key);
          // prune oldest entry
          if (this.max && keys.length > parseInt(this.max)) {
            pruneCacheEntry(cache, keys[0], keys, this._vnode);
          }
        }
        // 为缓存组件打上标志
        vnode.data.keepAlive = true;
      }
      // 将渲染的vnode返回
      return vnode || (slot && slot[0])
    }
  };
```
`keep-alive`选项跟我们平时写的组件选项还是基本类似的，唯一的不同是```keep-ailve```组件没有用```template```而是使用```render```函数。```keep-alive```本质上只是存缓存和拿缓存的过程，并没有实际的节点渲染，所以使用```render```处理是最优的选择。

### 13.3.3 缓存vnode

还是先回到流程图的分析。上面说到```keep-alive```在执行组件实例化之后会进行组件的挂载。而挂载```$mount```又回到```vm._render(),vm._update()```的过程。由于```keep-alive```拥有```render```函数，所以我们可以直接将焦点放在```render```函数的实现上。

- 首先是获取```keep-alive```下插槽的内容，也就是```keep-alive```需要渲染的子组件,例子中是```chil1 Vnode```对象，源码中对应```getFirstComponentChild```函数。

```js
  function getFirstComponentChild (children) {
    if (Array.isArray(children)) {
      for (var i = 0; i < children.length; i++) {
        var c = children[i];
        // 组件实例存在，则返回，理论上返回第一个组件vnode
        if (isDef(c) && (isDef(c.componentOptions) || isAsyncPlaceholder(c))) {
          return c
        }
      }
    }
  }
```

- 判断组件满足缓存的匹配条件，在```keep-alive```组件的使用过程中，```Vue```源码允许我们是用```include, exclude```来定义匹配条件，```include```规定了只有名称匹配的组件才会被缓存，```exclude```规定了任何名称匹配的组件都不会被缓存。更者，我们可以使用```max```来限制可以缓存多少匹配实例，而为什么要做数量的限制呢？我们后文会提到。

拿到子组件的实例后，我们需要先进行是否满足匹配条件的判断,**其中匹配的规则允许使用数组，字符串，正则的形式。**

```js
var include = ref.include;
var exclude = ref.exclude;
// 通过判断子组件是否满足缓存匹配
if (
    // not included
    (include && (!name || !matches(include, name))) ||
    // excluded
    (exclude && name && matches(exclude, name))
) {
    return vnode
}

// matches
function matches (pattern, name) {
    // 允许使用数组['child1', 'child2']
    if (Array.isArray(pattern)) {
        return pattern.indexOf(name) > -1
    } else if (typeof pattern === 'string') {
        // 允许使用字符串 child1,child2
        return pattern.split(',').indexOf(name) > -1
    } else if (isRegExp(pattern)) {
        // 允许使用正则 /^child{1,2}$/g
        return pattern.test(name)
    }
    /* istanbul ignore next */
    return false
}
```

如果组件不满足缓存的要求，则直接返回组件的```vnode```,不做任何处理,此时组件会进入正常的挂载环节。

3. `render`函数执行的关键一步是缓存```vnode```,由于是第一次执行```render```函数，选项中的```cache```和```keys```数据都没有值，其中```cache```是一个空对象，我们将用它来缓存```{ name: vnode }```枚举，而```keys```我们用来缓存组件名。
**因此我们在第一次渲染```keep-alive```时，会将需要渲染的子组件```vnode```进行缓存。**
```js
    cache[key] = vnode;
    keys.push(key);
```

4. 将已经缓存的```vnode```打上标记, 并将子组件的```Vnode```返回。
```vnode.data.keepAlive = true```


### 13.3.4 真实节点的保存

我们再回到```createComponent```的逻辑，之前提到```createComponent```会先执行```keep-alive```组件的初始化流程，也包括了子组件的挂载。并且我们通过```componentInstance```拿到了```keep-alive```组件的实例，而接下来**重要的一步是将真实的```dom```保存再```vnode```中**。

```js
function createComponent(vnode, insertedVnodeQueue) {
    ···
    if (isDef(vnode.componentInstance)) {
        // 其中一个作用是保留真实dom到vnode中
        initComponent(vnode, insertedVnodeQueue);
        // 将真实节点添加到父节点中
        insert(parentElm, vnode.elm, refElm);
        if (isTrue(isReactivated)) {
            reactivateComponent(vnode, insertedVnodeQueue, parentElm, refElm);
        }
        return true
    }
}
```
`insert`的源码不列举出来，它只是简单的调用操作```dom```的```api```,将子节点插入到父节点中，我们可以重点看看```initComponent```关键步骤的逻辑。

```js
function initComponent() {
    ···
    // vnode保留真实节点
    vnode.elm = vnode.componentInstance.$el;
    ···
}
```

**因此，我们很清晰的回到之前遗留下来的问题，为什么```keep-alive```需要一个```max```来限制缓存组件的数量。原因就是```keep-alive```缓存的组件数据除了包括```vnode```这一描述对象外，还保留着真实的```dom```节点,而我们知道真实节点对象是庞大的，所以大量保留缓存组件是耗费性能的。因此我们需要严格控制缓存的组件数量，而在缓存策略上也需要做优化，这点我们在下一篇文章也继续提到。**

由于```isReactivated```为```false```,```reactivateComponent```函数也不会执行。至此```keep-alive```的初次渲染流程分析完毕。

**如果忽略步骤的分析，只对初次渲染流程做一个总结：内置的```keep-alive```组件，让子组件在第一次渲染的时候将```vnode```和真实的```elm```进行了缓存。**



## 13.4 抽象组件
这一节的最后顺便提一下上文提到的抽象组件的概念。```Vue```提供的内置组件都有一个描述组件类型的选项，这个选项就是```{ astract: true }```,它表明了该组件是抽象组件。什么是抽象组件，为什么要有这一类型的区别呢？我觉得归根究底有两个方面的原因。
1. 抽象组件没有真实的节点，它在组件渲染阶段不会去解析渲染成真实的```dom```节点，而只是作为中间的数据过渡层处理，在```keep-alive```中是对组件缓存的处理。
2. 在我们介绍组件初始化的时候曾经说到父子组件会显式的建立一层关系，这层关系奠定了父子组件之间通信的基础。我们可以再次回顾一下```initLifecycle```的代码。

```js
Vue.prototype._init = function() {
    ···
    var vm = this;
    initLifecycle(vm)
}

function initLifecycle (vm) {
    var options = vm.$options;
    
    var parent = options.parent;
    if (parent && !options.abstract) {
        // 如果有abstract属性，一直往上层寻找，直到不是抽象组件
      while (parent.$options.abstract && parent.$parent) {
        parent = parent.$parent;
      }
      parent.$children.push(vm);
    }
    ···
  }
```
子组件在注册阶段会把父实例挂载到自身选项的```parent```属性上，在```initLifecycle```过程中，会反向拿到```parent```上的父组件```vnode```,并为其```$children```属性添加该子组件```vnode```,如果在反向找父组件的过程中，父组件拥有```abstract```属性，即可判定该组件为抽象组件，此时利用```parent```的链条往上寻找，直到组件不是抽象组件为止。```initLifecycle```的处理，让每个组件都能找到上层的父组件以及下层的子组件，使得组件之间形成一个紧密的关系树。

### 13.5 小结
这一节介绍了```Vue```内置组件中一个最重要的，也是最常用的组件```keep-alive```，在日常开发中，我们经常将```keep-alive```配合动态组件```is```使用，达到切换组件的同时，将旧的组件缓存。最终达到保留初始状态的目的。在第一次组件渲染时，```keep-alive```会将组件```Vnode```以及对应的真实节点进行缓存。而当再次渲染组件时，```keep-alive```是如何利用这些缓存的？源码又对缓存进行了优化？并且```keep-alive```组件的生命周期又包括哪些，这些疑问我们将在下一节一一展开。
