---
title: 是否需要使用 ESLint jsx-no-bind 规则？
tags:
  - javascript
  - react
  - eslint
categories: Programming
date: 2018-09-14 09:18:52
---


在使用 [ESLint React][1] 插件时，有一条名为 [`jsx-no-bind`][2] 的检测规则，它会禁止我们在 JSX 属性中使用 `.bind` 方法和箭头函数。比如下列代码，ESLint 会提示 `onClick` 属性中的箭头函数不合法：

```javascript
class ListArrow extends React.Component {
  render() {
    return (
      <ul>
        {this.state.items.map(item => (
          <li key={item.id} onClick={() => { alert(item.id) }}>{item.text}</li>
        ))}
      </ul>
    )
  }
}
```

这条规则的引入原因有二。首先，每次执行 `render` 方法时都会生成一个新的匿名函数对象，这样就会对垃圾回收器造成负担；其次，属性中的箭头函数会影响渲染过程：当你使用了 `PureComponent`，或者自己实现了 `shouldComponentUpdate` 方法，使用对象比较的方式来决定是否要重新渲染组件，那么组件属性中的箭头函数就会让该方法永远返回真值，引起不必要的重复渲染。

然而，反对的声音认为这两个原因还不足以要求我们在所有代码中应用该规则，特别是当需要引入更多代码、并牺牲一定可读性的情况下。在 [Airbnb ESLint][3] 预设规则集中，只禁止了 `.bind` 方法的使用，而允许在属性（props）或引用（refs）中使用箭头函数。对此我翻阅了文档，阅读了一些关于这个话题的博客，也认为这条规则有些过于严格。甚至还有博主称该规则是一种过早优化（premature optimization），我们需要先做基准测试，再着手修改代码。下文中，我将简要叙述箭头函数是如何影响渲染过程的，有哪些可行的解决方案，以及它为何不太重要。

<!-- more -->

## 不同类型的 React 组件

通常我们会通过继承 `React.Component` 类并实现 `render` 方法来创建一个 React 组件。另一个内置的组件基类是 `React.PureComponent`，它的区别在于已经为我们实现了 `shouldComponentUpdate` 方法。在普通的 React 组件中，该方法默认返回 `true`，也就是说当属性（props）或状态（state）发生改变时，一定会重新进行渲染。而 `PureComponent` 实现的该方法中，会对新、旧属性和状态的键值做一个等值比较，只有当内容发生改变时才会重新渲染。下面定义的这两个组件产生的效果是一致的：

```javascript
class PureChild extends React.PureComponent {
  render() {
    return (
      <div>{this.props.message}</div>
    )
  }
}

class RegularChildB extends React.Component {
  shouldComponentUpdate(nextProps, nextStates) {
    return this.props.message !== nextProps.message
  }

  render() {
    return (
      <div>{this.props.message}</div>
    )
  }
}
```

当它们的属性发生变化时，会检查 `message` 变量中的值是否和原来相等。属性和状态都是 `object` 类型，React 会遍历所有键值进行 `===` 等值比较。在 JavaScript 中，只有基础类型之间的比较、或同一个对象和自身比较时才能通过。

```javascript
1 === 1
'hello world' === 'hello world'
[] !== []
(() => {}) !== (() => {})
```

显然，箭头函数是无法通过这个等值检查的。如果父组件将箭头函数作为属性传入 `PureComponent`，那么每次渲染都会引发子组件的渲染。相反地，如果我们没有使用 `PureComponent`，或进行类似的等值比较，那么组件一定会进行更新，也就没有应用该规则的必要了。

另一个种较为流行的组件定义方式是“无状态函数式组件（SFC）”。这类组件好比一个数学函数，其渲染结果完全依赖于输入的属性值。不过，它本质上是一个普通的组件，并没有实现 `shouldComponentUpdate` 方法，且组件的定义方式也不允许我们自己来实现。

```javascript
const StatelessChild = (props) => {
  return (
    <div>{props.message}</div>
  )
}
```

## 如何修复 `jsx-no-bind` 错误警告

箭头函数通常会用作事件处理。如果我们直接使用普通的函数或类方法，`this` 关键字将无法正确绑定到当前实例，它的值是 `undefined`。只有使用了 `.bind` 方法或箭头函数时，我们才能在函数体中通过 `this` 来访问到类的其他成员，只是这样就会触发 `jsx-no-bind` 报警。解决方法是在构造函数中对方法进行绑定，或者使用尚在草案阶段的类属性语法，并通过 [Babel][6] 进行转换。更多信息可以查阅 [React 官方文档][5]，以下用 [代码][7] 演示不同的做法。

```javascript
export default class NoArgument extends React.Component {
  constructor() {
    this.handleClickBoundA = this.handleClickUnbound.bind(this)
    this.handleClickBoundC = () => { this.setState() }
  }
  handleClickUnbound() { /* "this" 的值是未定义 */ }
  handleClickBoundB = () => { this.setState() }
  render() {
    return (
      <div>
        Error: jsx-no-bind
        <button onClick={() => { this.setState() }}>ArrowA</button>
        <button onClick={() => { this.handleClickUnbound() }}>ArrowB</button>
        <button onClick={this.handleClickUnbound.bind(this)}>Bind</button>
        No error:
        <button onClick={this.handleClickBoundA}>BoundA</button>
        <button onClick={this.handleClickBoundB}>BoundB</button>
        <button onClick={this.handleClickBoundC}>BoundC</button>
      </div>
    )
  }
}
```

如果事件处理需要用到额外的参数，比如渲染列表时捕捉每一项的点击事件，就不那么容易了。有两种解决方案，一是将列表项作为独立的组件拆分出来，通过组件属性来传递事件处理函数和它的参数，示例如下：

```javascript
class Item extends React.PureComponent {
  handleClick = () => { this.props.onClick(this.props.item.id) }
  render() {
    return (
      <li onClick={this.handleClick}>{this.props.item.text}</li>
    )
  }
}

export default class ListSeparate extends React.Component {
  handleClick = (itemId) => { alert(itemId) }
  render() {
    return (
      <ul>
        {this.props.items.map(item => (
          <Item key={item.id} item={item} onClick={this.handleClick} />
        ))}
      </ul>
    )
  }
}
```

这种方式也称之为关注点分离（separation of concerns），因为 `List` 组件只需负责遍历列表项，而由 `Item` 组件来负责渲染。不过这样一来也会增加许多模板代码，我们需要跟踪多个属性值来确定事件处理过程，因此降低了代码可读性。若直接使用箭头函数，事件处理和组件渲染是在一处的，便于理解，也是 React 社区所推崇的方式。

另一种方式是使用 DOM [`dataset`][8] 属性，也就是将需要传递的参数暂存在 HTML 标签的 `data-*` 属性中， 然后通过 `event` 变量来读取。

```javascript
export default class ListDataset extends React.Component {
  handleClick = (event) => { alert(event.target.dataset.itemId) }
  render() {
    return (
      <ul>
        {this.props.items.map(item => (
          <li key={item.id} data-item-id={item.id} onClick={this.handleClick}>{item.text}</li>
        ))}
      </ul>
    )
  }
}
```

## 虚拟 DOM 与 React 协调

上文说到，箭头函数会引发 `PureComponent` 不必要的渲染，这个结论只正确了一半。React 的渲染过程可以分为几个步骤：首先，调用 `render` 方法，返回一个 React 元素的树形结构；将该结构与内存中的虚拟 DOM 树进行对比，将差异部分应用到浏览器的真实 DOM 树中。这个过程在 React 中称为协调（[reconciliation][4]）。因此，即便 `render` 方法被调用了多次，如果其返回的 React 元素树都是相同的，那么也不会触发真实 DOM 渲染，而这个过程通常会比纯 JavaScript 要来得耗时。这样看来，如果一个组件的确需要频繁变动，那么继承了 `PureComponent` 反而会增加一次比对的消耗，得不偿失。

![shouldComponentUpdate 生命周期方法](/images/jsx-no-bind/should-component-update.png)

[图片来源][9]

此外，在事件绑定属性中使用箭头函数，一般也不会触发真实 DOM 的渲染，原因是 React 的事件监听器是绑定在顶层的 `document` 元素上的，当 `li` 上触发了 `onClick` 事件后，该事件会向上冒泡（bubble up）至顶层元素，由 React 事件管理系统接收和处理。

![顶层事件委托](/images/jsx-no-bind/top-level-delegation.jpg)

[图片来源][10]

## 结论

可以看到，在修复 `jsx-no-bind` 的过程中，我们需要牺牲一定的代码可读性，而获得的性能收益也许是微不足道甚至是相反的。与其猜测箭头函数会引发性能问题，不如先用最自然的方式来编写代码，当遇到真正的性能瓶颈时加以测度，最终找出合适的技术方案。

## 参考资料

* https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/jsx-no-bind.md
* https://cdb.reacttraining.com/react-inline-functions-and-performance-bdff784f5578
* https://maarten.mulders.it/blog/2017/07/no-bind-or-arrow-in-jsx-props-why-how.html
* https://reactjs.org/docs/faq-functions.html#example-passing-params-using-data-attributes


[1]: https://github.com/yannickcr/eslint-plugin-react
[2]: https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/jsx-no-bind.md
[3]: https://github.com/airbnb/javascript/blob/eslint-config-airbnb-v17.1.0/packages/eslint-config-airbnb/rules/react.js#L93
[4]: https://reactjs.org/docs/reconciliation.html
[5]: https://reactjs.org/docs/handling-events.html
[6]: https://babeljs.io/docs/plugins/transform-class-properties/
[7]: https://github.com/jizhang/blog-demo/blob/master/jsx-no-bind/src/components/NoArgument.js
[8]: https://developer.mozilla.org/en/docs/Web/API/HTMLElement/dataset
[9]: https://reactjs.org/docs/optimizing-performance.html#shouldcomponentupdate-in-action
[10]: https://levelup.gitconnected.com/how-exactly-does-react-handles-events-71e8b5e359f2
