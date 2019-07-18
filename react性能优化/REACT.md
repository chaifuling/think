## react性能优化

其实渲染原理决定着性能优化的方法，只有在了解原理之后，才能完全理解为什么这样做可以优化性能。正所谓：知其然，然后知其所以然。

### react渲染

#### JSX如何生成element

```javascript
return (
    <div className="cn">
         <Header> Hello, This is React </Header>
         <div>Start to learn right now!</div>
         Right Reserve.
    </div>
)
```

首先，它会经过babel编译成React.createElement的表达式

```javascript
return (
    React.createElement(
        'div',
        { className: 'cn' },
        React.createElement(
            Header,
            null,
            'Hello, This is React'
        ),
        React.createElement(
            'div',
            null,
            'Start to learn right now!'
        ),
        'Right Reserve'
    )
)
```

#### React 是如何对比出页面变化最小的部分?

你可能会说 React 用 virtual DOM 表示了页面结构, 每次更新, React 都会re-render出新的 virtual DOM, 再通过 diff 算法对比出前后变化, 最后批量更新. 没错, 很好, 这就是大致过程, 但这里存在着一些隐藏的深层问题值得探讨 :

- React 是如何用 virtual DOM 表示了页面结构, 从而使任何页面变化都能被 diff 出来?
- React 是如何 diff 出页面变化最小的部分?

##### React 如何表示页面结构

```javascript
class C extends React.Component {
    render () {
        return (
            <div className='container'>
                  "dscsdcsd"
                  <i onClick={(e) => console.log(e)}>{this.state.val}</i>
                  <Children val={this.state.val}/>
            </div>
        )
    }
}
// virtual DOM(React element)
{
  $$typeof: Symbol(react.element)
  key: null
  props: {  // props 代表元素上的所有属性, 有children属性, 描述子组件, 同样是元素
    children: [
      ""dscsdcsd"",
	  {$$typeof: Symbol(react.element), type: "i", key: null, ref: null, props: {…}, …},
	  {$$typeof: Symbol(react.element), type: class Children, props: {…}, …}
    ]
    className: 'container'
  }  
  ref: null
  type: "div"
  _owner: ReactCompositeComponentWrapper {...} // class C 实例化后的对象
  _store: {validated: false}
  _self: null
  _source: null
}
```

每个标签, 无论是DOM元素还是自定义组件, 都会有 key、type、props、ref 等属性

- key 代表元素唯一id值, 意味着只要id改变, 就算前后元素种类相同, 元素也肯定不一样了
- type 代表元素种类, 有 function(空的wrapper)、class(自定义类)、string(具体的DOM元素名称)类型, 与key一样, 只要改变, 元素肯定不一样
- props 是元素的属性, 任何写在标签上的属性(如className='container')都会被存在这里, 如果这个元素有子元素(包括文本内容), props就会有children属性, 存储子元素; children属性是递归插入、递归更新的依据

也就是说, 如果元素唯一标识符或者类别或者属性有变化, 那么它们re-render后对应的 key、type 和props里面的属性也会改变, 前后一对比即可找出变化. 综上来看, React 这么表示页面结构确实能够反映前后所有变化

##### 那么 React 是如何 diff 的？

`React`基于两个假设：

- 两个相同的组件产生类似的`DOM结构`，不同组件产生不同`DOM结构`
- 对于同一层次的一组子节点，它们可以通过唯一的id区分

同时，基于第一点假设，我们可以推论出，`Diff`算法只会对同层的节点进行比较。如图，它只会对颜色相同的节点进行比较。

![img](https://pic1.zhimg.com/80/v2-19e7de9d2da5510727f0398c7e92756c_hd.jpg)

也就是说如果父节点不同，`React`将不会在去对比子节点。因为不同的组件`DOM结构`会不相同，所以就没有必要在去对比子节点了。这也提高了对比的效率。直接将算法复杂度（o3）变成了（0）

#### Fiber机制

react最新版已经引入了核心调度算法即fiber机制，为什么需要引入fiber？

- 通过上面我们已经知道，React渲染页面分为两个阶段：
  - 调度阶段（reconciliation）：在这个阶段 React 会更新数据生成新的 Virtual DOM，然后通过Diff算法，快速找出需要更新的元素，放到更新队列中去，得到新的更新队列。
  - 渲染阶段（commit）：这个阶段 React 会遍历更新队列，将其所有的变更一次性更新到DOM上
- React渲染的不可控，虽然react的diff算法很优，但是如果存在一个状态发生变化，涉及到1000个组件的更新，这样有可能主线程被React占着用来调度，这段时间内用户的操作不会得到任何的反馈，只有当 React 中需要同步更新的任务完成后，主线程才被释放。对于这段时间内 React 的调度，我们是无能为力的，那么浏览器失去响应，用户体验非常差

Fiber 的中文解释是`纤程`，是线程的颗粒化的一个概念。也就是说一个线程可以包含多个 Fiber。

简单说就是依赖requestIdleCallback`/`requestAnimationFrame实现调度器

`requestIdleCallback`*会让一个低优先级的任务在空闲期被调用，而*`requestAnimationFrame`*会让一个高优先级的任务在下一个栈帧被调用，从而保证了主线程按照优先级执行Fiber单元。*

```javascript
组件的 props/state 更改(开发者控制) -> shouldComponentUpdate（开发者控制）-> 计算 VDOM 的更新（React 的 diff 算法会计算出最小化的更新）-> 更新真实 DOM (React 控制)
```

**当组件的状态发生变化，React 将会构建新的 virtual DOM，使用 diff 算法把新老的 virtual DOM 进行比较，如果新老 virtual DOM 树不相等则重新渲染，相等则不重新渲染，这个过程需要耗费的时间。**

**react在渲染的过程中避免不了对DOM的 操作，DOM操作是非常耗时的**

**因此要提高组件的性能就应该尽一切可能的减少组件的重新渲染**

### 优化实践

#### props和state

组件的 props 或 state 变化，React 将会尝试渲染

- 状态数据层次不能过深，尽可能扁平化，便于数据对比，数组遍历从而减小带来的性能消耗
- 给map列表项添加key
- 不要随意更改props和state的地址引用
- 尽可能保证eventHandle是同一个引用

```javascript
<div onClick={this.tap.bind(this)} /> 
constructor(props) {
    super(props);
    this.tap= this.tap.bind(this);
}
tap =(value)=> {};
<div onClik={()=>this.tap(value)} />
```

#### 别滥用SFC

SFC组件没有 ref，没有生命周期，React 还可以避免不必要的内存申请及检查，这意味着更高效的渲染，React 会直接调用 createElement 返回 VDOM

父组件想要对SFC 以下行为做控制：

- 何时渲染，因为 SFC 没有生命周期，只要传入新的 props/state 就会 re-render，而且这意味着，一旦SFC  的父组件更新，SFC 就会 re-render
- 渲染什么，本身没法控制，高频的re-render 就会带来 VDOM的 diff，这笔开销累计起来就会很大

对于SFC的使用一定要节制，也就是说在使用SFC之前要思考清楚具体做什么，更新的频率等，否则就会适得其反

#### shouldComponentUpdate

##### Pure Component

如果一个组件只和 props 和 state 有关系，给定相同的 props 和 state 就会渲染出相同的结果，那么这个组件就叫做纯组件，换一句话说纯组件只依赖于组件的 props 和 state，下面的代码表示的就是一个纯组件（Pure Component）。

```javascript
class extends React.PuerComponent {
  render() {
    return (
      <div style={{ width: this.props.width }}>
        {this.state.rows}
      </div>
    );
  }
}
```

##### 手动控制

`shouldComponentUpdate` 这个函数会在组件重新渲染之前调用，函数的返回值确定了组件是否需要重新渲染。函数默认的返回值是 `true`，意思就是只要组件的 props 或者 state 发生了变化，就会重新构建 virtual DOM，然后使用 diff 算法进行比较，再接着根据比较结果决定是否重新渲染整个组件。函数的返回值为 `false` 表示不需要重新渲染。 `shouldComponentUpdate` 在组件的重新渲染的过程中的位置如下图：

![6](http://img1.tbcdn.cn/L1/461/1/354dc8a5e804e06986dae35a92309ec7685499d9)

##### 优化一：Container and Component

我们也可以通过组件的容器来隔离外界的变化。容器就是一个数据层，而组件就是专门负责渲染，不进行任何数据交互，只根据得到的数据渲染相应的组件，下面就是一个容器以及他的组件。

```javascript
class BudgetContainer extends Component {
  constructor(props) {
    super(props);
  }

  shouldComponentUpdate() {
		//	进行相应的更新控制
  }

  computeState() {
    return BudgetStore.getStore()
  }

  render() {
    return <Budget { ...this.state } />
  }
}
```

容器不应该有 props 和 children，这样就能够把容器自己和父组件进行隔离，不会因为外界因素去重新渲染，也没有必要重新渲染。

设想一下，如果设计师觉得这个组件需要移动位置，你不需要做任何的更改只需要把组件放到对应的位置即可，我们可以把它移到任何地方，可以放在不同的应用中，同时也可以应用于测试，我们只需要关心容器的内部的数据的来源即可，在不同的环境中编写不同的容器。

使用return null而不是CSS的display:none来控制节点的显示隐藏。保证同一时间页面的DOM节点尽可能的少

##### 优化二：动静分离

假设我们有一个下面这样的组件：

```javascript
<ScrollTable
	width={300}
	color='blue'
	scrollTop={this.props.offsetTop}
/>
```

这是一个可以滚动的表格，`offsetTop` 代表着可视区距离浏览器的的上边界的距离，随着鼠标的滚动，这个值将会不断的发生变化，导致组件的 props 不断地发生变化，组件也将会不断的重新渲染。如果使用下面的这种写法：

```javascript
<OuterScroll>
	<InnerTable width={300} color='blue'/>
</OuterScroll>

```

因为 `InnerTable` 这个组件的 props 是固定的不会发生变化，在这个组件里面使用 `shouldComponentUpdate` ，返回一直为 `false`， 因此不管组件的父组件也就是 `OuterScroll` 组件的状态是怎么变化，组件 `InnerTable` 都不会重新渲染。也就是子组件隔离了父组件的状态变化。

通过把变化的属性和不变的属性进行分离，减少了重新渲染，获得了性能的提升，同时这样做也能够让组件更容易进行分离，更好的被复用。

##### 优化三：最小化变动

下面来看一个例子，一个 list 有 10000 个未标记的 Item，点击某一 Item 该 Item 就会变为已标记，再点击就会变为未标记。很简单对不对，我们采用 redux + react-redux 来实现。

store 中存储的 state：

```javascript
{
    [{id:0, marked: false}, {id:1, marked: false}, ...]
}
```

我们给每一个item绑定onClick 对应的回调函数

```javascript
function onClickHandle(id) {
    return state.map((item) =>
      action.id === item.id ? {...item, marked: !item.marked } : item
}
```

可以看下，性能如何？

###### 思考一：让未被修改的组件对改变无感知

继续改进，想一下最优解，当我们在更新一个 Item 时，如果其他未被修改的 Item 的 props 和 state 没有任何的改变，那么就完全不会经过他们的生命周期。

所以，这里要将数据和组件重新组合。

- 为了避免父组件的 re-render，我们将每个 Item 和 redux store 直接连接
- 将 store 拆分为 ids 和 items，用 ids 给父组件完成 Item 初始化提供一些必要的信息，用 items 对 Item 进行初始化和更新
- 每次点击的时候 ids 不变，所以父组件不会 re-render，只更新对应的子组件。子组件的 reducer

```javascript
updateState(state, { payload }) {
  return {
    ...state,
  [action.id]: {...item, marked: !item.marked}
  };
}
```

当某个 Item 被 mark 时，虽然返回了一个新的对象，但是 `…` 解构函数是浅拷贝，所以 items 数组中对象的引用不变，只有发生更新的那个对象是新的。这样就做到了只改变需要改变的 Item 的 props。

除了真正要更新的 Item，其他所有 Item 对这次点击都是无感知的，性能好了很多。

###### 思考二：范式化 store

但是 ids 和 items 中的 id 冗余了，如果后面还要再加上添加和删除的功能就要同时修改两个属性，如果 ids 只是用来初始化 App 的话就会一直在 store 中残留，还容易引起误会。所以，还是将 store 变回如下的组织：

```javascript
store:{
    items:[
        {id: 0, marked: false},
        {id: 1, marked: false},
		...
    ]
}
```

其他的处理的方式类似**优化一**，每次进行局部更新

但是要补全 App 的 shouldComponentUpdate，因为虽然是局部更新，但是 reducer 是一个纯函数，纯函数每次不修改原 state，返回一个新 state，所以只要手动控制一个 App 的 shouldComponentUpdate 即可，根据业务需要写即可，这里只是做个演示，就直接返回了 false，相当于 App 只是完成初始化的功能。

```javascript
shouldComponentUpdate() {
  return false
}
```

##### 优化四：Immutable.js

对于复杂的数据的比较是非常耗时的，而且可能无法比较，通过使用 Immutable.js 能够很好地解决这个问题， Immutable.js和shouldComponentUpdate是最佳配合。

Immutable.js 的基本原则是对于不变的对象返回相同的引用，而对于变化的对象，返回新的引用。因此对于状态的比较只需要使用如下代码即可：

```javascript
shouldComponentUpdate() {
	return ref1 !== ref2;
}
```

#### state vs store

不要将所有的状态全部放在 store 中,store 中应该存放异步获取的数据或者多个组件需要访问的数据等等

redux 官方文档中也有写什么数据应该放入 store 中

> - 应用中的其他部分需要用到这部分数据吗？
> - 是否需要根据这部分原始数据创建衍生数据？
> - 这部分相同的数据是否用于驱动多个组件？
> - 你是否需要能够将数据恢复到某个特定的时间点（比如：在时间旅行调试的时候）？
> - 是否需要缓存数据？（比如：直接使用已经存在的数据，而不是重新请求）

store 中不应该保存 UI 的状态(除非符合上面的某一点，比如回退时页面的滚动位置)

state 也应该用最少的数据表示尽可能多的信息，在 render 函数中，根据 state 去衍生其他的信息而不是将这样冗余的信息都存在 state 中

store 和 state 都应该尽可能的做到**熵最小**，具体可以看[redux store取代react state合理吗？](https://www.zhihu.com/question/271693121)

### 参考文章

[React高效渲染策略](https://github.com/fi3ework/blog/issues/15)

[将React应用优化到60fps](https://zhuanlan.zhihu.com/p/24959748?utm_source=qq&utm_medium=social&utm_oi=636656349526757376)

[react渲染列表的性能问题](https://zhuanlan.zhihu.com/p/59597247?utm_source=qq&utm_medium=social&utm_oi=636656349526757376)

