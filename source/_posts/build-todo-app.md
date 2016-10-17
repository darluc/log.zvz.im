title: 使用 Redux 秒做 todo-list 应用
date: 2016-10-13 01:39:11
tags: 
- javascript
---

[React](https://facebook.github.io/react) 采用组件和单向数据流的方式完美地解决了用户界面的构建问题，不过它所用于处理状态的工具设计得太简单了，感觉就是故意的；也可能是为了强调 React 只是对应传统的 [MVC](http://blog.codinghorror.com/understanding-model-view-controller/) 架构中 View 的部分。

## 作者认为

没有什么能够阻止我们用 React 来构建越来越大的应用，但是很快我们就会发现，想要保持代码简单，我们到处都需要进行状态的管理。

尽管没有官方的解决方案，但是仍然有不少代码库依照 React 的设计理念解决了状态管理的问题。今天我们就要介绍两个这样的类库，并使用它们构建一个简单的小应用。

## Redux

[Redux](https://redux.js.org/) 是一个极精简的库，它融合了 [Flux](https://redux.js.org/) 和 [Elm](https://redux.js.org/) 的设计理念，为应用状态提供容器功能。我们可以使用 Redux 管理任何应用的状态，使我们遵守以下准则：
<!--more-->

1. 状态（ status ）都保存在同一个存储（ store ）中；
2. 动作（ action ）才能改变状态，状态不能直接被修改；

Redux 存储的核心是一个函数，它将当前应用的状态和动作结合起来，创建应用的新状态。我们将这个函数称为 *reducer*。

我们的 React 组件负责向我们的存储发送动作，相应地存储通知组件何时需要重绘。

## ImmutableJS

由于 Redux 不允许直接对应用的状态做修改，所以使用不可修改数据作为应用状态，可以禁止对状态的修改，就显得非常好用了。

[ImmutableJS](https://facebook.github.io/immutable-js) 为我们提供了许多的不可修改的数据结构，以及对应的修改操作接口，其实现方式参考了 Clojure 和 Scala 中的现实，[有很高的代码效率](https://facebook.github.io/immutable-js)。

## 示例

接下来，我们将使用 React、Redux 和 ImmutableJS 来创建一个简单的待办事项列表，它允许我们添加待办事项，且能够切换事项完成状态。

你可以去 [CodePen](http://codepen.io/) 看 SitePoint（@SitePoint）实现的一个 [React, Redux & Immutable Todo](http://codepen.io/SitePoint/pen/bpxapd/) 应用。

也可以去 [GitHub](https://github.com/sitepoint-editors/immutable-redux-todo) 上下载相应的代码。

## 准备工作

我们从建立项目文件目录开始，然后使用 `npm init` 指令初始化一个 `package.json` 文件。然 后我们开始安装我们需要的依赖库。

```shell
npm install --save react react-dom redux react-redux immutable
npm install --save-dev webpack babel-loader babel-preset-es2015 babel-preset-react
```

我们使用 [JSX](https://facebook.github.io/jsx/) 和 [ES2015](https://facebook.github.io/jsx/) 的语法，所以我们会在 [Webpack](https://webpack.github.io/) 进行模块绑定时，用 [Babel](http://babeljs.io/) 来编译 我们的代码。

首先，我们新建一个 Webpack 的配置文件 `webpack.config.js`。

```javascript
module.exports = {
  entry: './src/app.js',
  output: {
    path: __dirname,
    filename: 'bundle.js'
  },
  module: {
    loaders: [
      {
        test: /\.js$/,
        exclude: /node_modules/,
        loader: 'babel',
        query: { presets: [ 'es2015', 'react' ]}
      }
    ]
  }
};
```

然后我们需要扩充一下我们的 `package.json` 文件，加入一行 npm 脚本用来编译我们的代码。

```javascript
"script": {
  "build": "webpack --debug"
}
```

当我们需要编译代码时，只要执行一次 `npm run build` 命令即可。

## React & 组件

在我们编写组件之前，做一些模拟数据会有很大帮助。它能让我们直观感受到我们的组件将要渲染的数据。

```javascript
const dummyTodos = [
  { id: 0, isDone: true, text: 'make components' },
  { id: 1, isDone: false, text: 'design actions' },
  { id: 2, isDone: false, text: 'implement reducer' },
  { id: 3, isDone: false, text: 'connect components' }
];
```

对于这个应用来说，我们只需要两个 React 组件，`<Todo />` 和 `<TodoList />` 。

```jsx
// src/components.js

import React from 'react';

export function Todo(props) {
  const {todo} = props;
  if(todo.isDone) {
    return <strike>{todo.text}</strike>;
  } else {
    return <span>{todo.text}</span>;
  }
}

export function TodoList(props) {
  const { todos } = props;
  return (
    <div className='todo'>
      <input type='text' placeholder='Add todo' />
      <ul className='todo__list'>
        {
          todos.map(t => (
            <li key={t.id} className='todo__item'>
              <Todo todo={t} />
            </li>
          ))
        }
      </ul>
    </div>
  );
}
```

现在，我们可以在项目目录下创建一个 `index.html` 文件来测试一下这些组件，写入以下代码。（你可以在 [GitHub](thunder://QUFlZDJrOi8vfGZpbGV80sm3uNe319kuUGVyc29uLm9mLkludGVyZXN0LlMwMkUxMS5DaGlfRW5nLldFQi1IUi5BQzMuMTAyNFg1NzYueDI2NC1ZWWVUc8jLyMvTsMrTLm1rdnw1MjE3MDY5NTR8Y2ZjY2ZiMTFiN2IxM2ZmZjZhMjMxZDFjOWFhODE1YWV8aD1iZTI1dGUyZjJndnRtYnF4bmt6Mm1hZ2UzZmIyd212NHwvWlo=) 上找到对应的样式文件）

```html
<!DOCTYPE html>
<html>
  <head>
    <link rel="stylesheet" href="styles.css">
    <title>Immutable Todo</title>
  </head>
  <body>
    <div id="app"></div>
    <script src="bundle.js"></script>
  </body>
</html>
```

还需要一个应用入口文件 `src/app.js` 。

```jsx
// src/app.js

import React from 'react';
import { render } from 'react-dom';
import { TodoList } from './components';

const dummyTodos = [
  { id: 0, isDone: true, text: 'make components' },
  { id: 1, isDone: false, text: 'design actions' },
  { id: 2, isDone: false, text: 'implement reducer' },
  { id: 3, isDone: false, text: 'connect components' }
];

render(
  <TodoList todos={dummyTodos} />,
  document.getElementById('app')
);
```

执行 `npm run build` 编译这些代码，在你的浏览器中打开 `index.html` 文件，确保它能正常工作。

## Redux & ImmutableJS

现在我们已经有了不错的用户界面，可以开始考虑底层的状态问题了。让我们从假数据开始，将其很容易就修改为 ImmutableJS 的集合。

```javascript
import { List, Map } from 'immutable';

const dummyTodos = List([
  Map({ id: 0, isDone: true, text: 'make components' }),
  Map({ id: 1, isDone: false, text: 'design actions' }),
  Map({ id: 2, isDone: false, text: 'implement reducer' }),
  Map({ id: 3, isDone: false, text: 'connect components' })
]);
```

ImmutableJS 的映射表 (maps) 与 Javascript 的对象并不一样，所以我们需要略微调整一下之前所写的组件。 所有直接访问属性 ( 比如 *todo.id* ) 的地方，都需要修改为方法调用 ( *todo.get('id')* ) 。

## 设计动作

我们已经有了应用的界面和数据结构，可以开始考虑哪些动作会使得应用更新。在此例中，我们仅需定义两个动作，一个是新增待办事项，另一个是改变待办事项的完成状态。

我们通过定义一些函数来创建这些动作。

```javascript
// src/actions.js

// succinct hack for generating passable unique ids
const uid = () => Math.random().toString(34).slice(2);

export function addTodo(text) {
  return {
    type: 'ADD_TODO',
    payload: {
      id: uid(),
      isDone: false,
      text: text
    }
  };
}

export function toggleTodo(id) {
  return {
    type: 'TOGGLE_TODO',
    payload: id
  }
}
```

每个动作都会返回一个包含类型 ( type ) 和负载数据 ( payload ) 的 Javascript 对象。类型将会决定当我们处理此动作时如何利用这些负载数据。

## 设计 Reducer

至此，我们已经搞定了应用的状态数据以及用于更新状态的动作，可以开始构建 reducer 了。请注意：reducer 是一个以动作和状态作为参数，并据此计算出新的状态的函数。

以下是我们的 reducer 的初始结构。

```javascript
// src/reducer.js

import { List, Map } from 'immutable';

const init = List([]);

export default function(todos = init, action) {
  switch(action.type) {
    case 'ADD_TODO':
      // ...
    case 'TOGGLE_TODO':
      // ...
    default:
      return todos;
  }
}
```

处理 *ADD_TODO* 动作很简单，我们只需要使用 *.push()* 方法，就可以得到一个新的包含新增待办事项的事项列表。

```javascript
case 'ADD_TODO':
  return todos.push(Map(action.payload));
```

注意我们将 todo 对象转化为了一个不可变的 map 对象后，才将其加入列表中。

相比较而言 *TOGGLE_TODO* 会复杂一些。

```javascript
case 'TOGGLE_TODO':
  return todos.map(t => {
    if(t.get('id') === action.payload) {
      return t.update('isDone', isDone => !isDone);
    } else {
      return t;
    }
  });
```

我们使用了 *.map()* 方法轮询列表中的数据，找出其中 *id* 与动作负载数据相匹配的一条。然后调用 *.update()* 方法，传入一个 key 值和一个函数作为参数。最终返回了一个新的 map 对象，其 key 所对应的值被这个传入初始值的函数返回的结果所取代。

下面的注释代码对你的理解也许有点帮助。

```javascript
const todo = Map({ id: 0, text: 'foo', isDone: false});
todo.update('isDone', isDone => !isDone);
// => { id: 0, text: 'foo', isDone: true}
```

## 将一切联接起来

既然我们已经准备好了 actions 和 reducer ，那么就可以创建一个存储（ store ），并且将其与我们的 React 组件联接起来。

```jsx
// src/app.js

import React from 'react';
import { render } from 'react-dom';
import { createStore } from 'redux';
import { TodoList } from './components';
import reducer from './reducer';

const store = createStore(reducer);

render(
  <TodoList todos={store.getState()} />,
  document.getElementById('app')
);
```

我们要让组件能够感知到存储。我们可以用 [react-redux](https://github.com/reactjs/react-redux) 来简化这个过程。它可以帮我们创建一个与存储关联的容器，并将我们的组件放置其中，这样我们就不用修改之前所写的代码了。

我们将会用一个容器包裹我们的 `<TodoList />` 组件。看起来就像这样。

```javascript
// src/containers.js

import { connect } from 'react-redux';
import * as components from './components';
import { addTodo, toggleTodo } from './actions';

export const TodoList = connect(
  function mapStateToProps(state) {
    // ...
  },
  function mapDispatchToProps(dispatch) {
    // ...
  }
)(components.TodoList);
```

我们使用 [connect](https://github.com/reactjs/react-redux/blob/master/docs/api.md#connectmapstatetoprops-mapdispatchtoprops-mergeprops-options) 方法来创建容器。调用 `connect()` 方法时，我们会传入 `mapStateToProps()`  和 `mapDispatchToProps()` 两个函数作为参数。

`mapStateToProps` 方法使用存储的当前状态作为参数（本例中就是一个 todo 列表），而返回值是一个对象，用来描述状态和包裹组件之间的映射关系。

```javascript
function mapStateToProps(state) {
  return { todos: state };
}
```

下面的代码，表示了一个被包裹的 React 组件。

```jsx
<TodoList todos={state} />
```

我们还要提供一个 `mapDispatchToProps` 函数，它会使用存储的 `dispatch` 方法作为参数，这样就可以利用它来分发由我们的动作生成器所生成的动作。

```javascript
function mapDispatchToProps(dispatch) {
  return {
    addTodo: text => dispatch(addTodo(text)),
    toggleTodo: id => dispatch(toggleTodo(id))
  }
}
```

再看一下代码，被包裹的 React 的组件是如何与这些属性组合的。

```jsx
<TodoList todos={state}
  addTodo={text => dispatch(addTodo(text))}
  toggleTodo={id => dispatch(toggleTodo(id))} />
```

现在我们已经使组件知晓了动作生产器，可以通过事件监听来调用它们了。

```jsx
export function TodoList(props) {
  const { todos, toggleTodo, addTodo } = props;
  
  const onSubmit = (event) => {
    const input = event.target;
    const text = input.value;
    const isEnterKey = (event.which == 13);
    const isLongEnough = text.length > 0;
    
    if(isEnterKey && isLongEnough) {
      input.value = '';
      addTodo(text);
    }
  }
};

const toggleClick = id => event => toggleTodo(id);

return (
  <div className='todo'>
    <input type='text'
      className='todo__entry'
      placeholder='Add todo'
      onKeyDown={onSubmit} />
    <ul className='todo__list'>
      {todos.map(t => {
        <li key={t.get('id')}
          className='todo__item'
          onClick={toggleClick(t.get('id'))}>
          <Todo todo={t.toJS()} />
        </li>
      })}
    </ul>
  </div>
);
```

容器将会自动侦测到存储中的变化，并在被包裹的组件属性发生变化时，重新绘制组件。

最后，我们要使用 `<Provider />` 组件，让容器与存储关联起来。

```jsx
// src/app.js

import React from 'react';
import { render } from 'react-dom';
import { createStore } from 'redux';
import { Provider } from 'react-redux';
import reducer from './reducer';
import { TodoList } from './containers';

const store = createStore(reducer);

render(
  <Provider store={store}>
    <TodoList />
  </Provider>,
  document.getElementById('app')
);
```

## 总结

不可否认 React 加上 Redux 的生态系统会略有些复杂，甚至对于初学者来说有点吓人，不过好处也很明显，几乎所有这些概念都是可转移的（ transferable ）。我们早已受够了从头开始学习 [The Elm 架构](https://github.com/evancz/elm-architecture-tutorial) ，或者从零开始一个像 [Om](https://github.com/omcljs/om) 或 [Re-frame](https://github.com/Day8/re-frame) 之类的 ClojureScript 库，而现在我们完全不必接触到 Redux 的底层架构。此外，我们也只体会到不可变数据能力的一小部分，现在我们有更好的能力开始学习 [Clojure](http://clojure.org/) 或 [Haskell](https://www.haskell.org/) 语言了。

无论你是正在探索 web 应用开发趋势，还是早已成天地在编写 Javascript；行为驱动式架构和不可变数据的相关经验已经成为现今开发者的必备技能了，所以**从现在开始**学习这些技能正是时候！



> 翻译自：[How to Build a Todo App Using React, Redux and Immutable.js](https://www.sitepoint.com/how-to-build-a-todo-app-using-react-redux-and-immutable-js/)