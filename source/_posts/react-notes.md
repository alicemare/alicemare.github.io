---
title: react-notes
date: 2021-01-18 17:15:11
tags: JavaScript
---



简单组件

`React`组件使用一个名为`render()`的方法，接收输入的数据并返回需要展示的内容。在示例中这种类似 XML 的写法被称为 JSX。被传入的数据可在组件中通过`this.props`在`render()`访问。

```JavaScript
class HelloMessage extends React.Component {
  render() {
    return (
      <div>
        Hello {this.props.name}
      </div>
    );
  }
}

ReactDOM.render(
  <HelloMessage name="Taylor" />,
  document.getElementById('hello-example')
);
```

除了通过 this.props 访问外部数据以外，组件还可以维护其内部的状态数据（通过 this.state 访问）