---
layout: post
title: 如何为react-redux（connect）写typescript类型
category: typescript
---

## 背景
最近搞了一波redux/connect的类型，用ts写真是头大死了，为了追求尽可能完美的类型保护和提示，经过不断研究总算摸到了些门路，大幅度降低了类型的声明成本，先看个例子
```typescript
const ConnectedComponent = connect(
  function mapStateToProps(state) {
    return {
      user: state.user
    }
  }
)(function MyComponent(props) {
  return (
    <div>
      <h1>{props.user.name} {props.user.age}</h1>
      <ul>
        {props.books.map(({ name, price }) => <p>{name} - {price}</p>)}
      </ul>
    </div>
  )
});

// usage
const store = createStore(/*...*/);
function App() {
  return (
    <Provider store={store}>
      <ConnectedComponent />
    </Provider>
  )
}
```

这个情况下几乎没有类型，ts定义无法发挥任何作用，所有的state，props均无法智能提示和校验，同时ConnectedComponent也无法声明props约束，甚至在MyComponent里的props自动推导的类型是never，强行使用就会引发编辑器一片错误提示


## 预期
* 能让MyComponent的props正确推导出类型，能够使用`props.user`和`props.books`
* 约束使用ConnectedComponent的props，如果未提供必要的`books`属性时报错

## 上手
### 重新定义connect
首先去查了一下`@types/redux`中关于`connect`的声明，其内部支持的类型比较复杂，也有很多我们用不上的场景，所以需要一种简化的connect类型

```typescript
// helps.ts
import { connect as originConnect } from 'react-redux';
interface EasyConnect {
}
export const connect: EasyConnect = originConnect;
```

这样我们的connect用的还是原始实现，但是已经偷偷换掉了类型

### 支持mapStateToProps的情况
接下来一步一步来写EasyConnect的类型，需要后面细化的先写个any
1. EasyConnect是个函数，
2. EasyConnect有一个参数(mapStateToProps)，并且该参数类型是个函数
3. EasyConnect的参数(mapStateToProps)总是会返回一个对象

```typescript
interface EasyConnect {
  (mapStateToProps: (state: any) => any): any;
}
```

接下来处理返回值的问题
4. EasyConnect的返回值是个函数（为了方便表达，我们管它叫Connector,并单独写出来）
5. Connector接受一个React组件
6. Connector返回一个React组件

```ts
interface Connector {
  (component: React.ComponentType): React.ComponentType;
}
interface EasyConnect {
  (mapStateToProps: (state: any) => any): Connector;
}
```

接下来就需要引入泛型定义，注意下面泛型声明的位置，是在函数体之前，而不是EasyConnect声明的位置，表示该泛型参数是执行时注入的，而不是在声明时注入

7. 需要有组件自身props约束声明(TOwnProps)
8. 需要有从全局state分离出来的state声明(TInjectProps)
9. 需要有全局state声明(TRootState)

```typescript
interface Connector {
  (component: React.ComponentType): React.ComponentType;
}
interface EasyConnect {
  <TInjectProps, TOwnProps = {}, TRootState = any>(
    mapStateToProps: (state: any) => any
  ): Connector;
}
```

在此基础上按照connect的实现逐个填入正确的位置

10. `mapStateToProps`接受全局state，并返回映射后注入的props

```typescript
interface Connector {
  (component: React.ComponentType): React.ComponentType;
}
interface EasyConnect {
  <TInjectProps, TOwnProps = {}, TRootState = any>(
    mapStateToProps: (state: TRootState) => TInjectProps
  ): Connector;
}
```

随后加入Connector之后的组件props约束，基于ComponentType的泛型参数

11. 被connect的组件，可以使用的props是OwnProps + 由全局转化来的props
12. connect之后的组件的props约束是OwnProps 

```ts
interface Connector<TInjectProps, TOwnProps> {
  (
    component: React.ComponentType<TInjectProps & TOwnProps>
  ): React.ComponentType<TOwnProps>;
}
interface EasyConnect {
  <TInjectProps, TOwnProps = {}, TRootState = any>(
    mapStateToProps: (state: TRootState) => TInjectProps
  ): Connector;
}
```

13. 将Connector的泛型约束传入

```typescript
interface Connector<TInjectProps, TOwnProps> {
  (
    component: React.ComponentType<TInjectProps & TOwnProps>
  ): React.ComponentType<TOwnProps>;
}
interface EasyConnect {
  <TInjectProps, TOwnProps = {}, TRootState = any>(
    mapStateToProps: (state: TRootState) => TInjectProps
  ): Connector<TInjectProps, TOwnProps>;
}
```

这样大体上就完成了，然后看下如何使用这个类型

```typescript
import { connect } from './helps';

type MapStateProps = { 
  user: {
    name: string;
    age: number;
  }
};
type OwnProps = {
  books: {
    name: string;
    price: number;
  }[] 
};
type RootState = {
  user: {
    name: string;
    age: number;
  }
}
const ConnectedComponent = connect<MapStateProps, OwnProps, RootState>(
  function mapStateToProps(state) {
    return {
      user: state.user
    }
  }
)(function MyComponent(props) {
  // 这里所有props的属性都会正确提示出来
  return (
    <div>
      <h1>{props.user.name} {props.user.age}</h1>
      <ul>
        {props.books.map(({ name, price }) => <p>{name} - {price}</p>)}
      </ul>
    </div>
  )
});

// usage
const store = createStore(/*...*/);
function App() {
  // 这里的ConnectedComponent不传books的话，就会有错误提示拉
  return (
    <Provider store={store}>
      <ConnectedComponent />
    </Provider>
  )
}
```
### 支持dispatch
基础思路都差不多，这里就不再逐行分解来写了

```typescript
import { connect as originConnect, DispatchProp } from 'react-redux';

type AnyObject = {[key: string]: any};
interface Connector<TInjectProps, TOwnProps> {
  (
    component: React.ComponentType<TOwnProps & TInjectProps>
  ): React.ComponentType<TOwnProps>;
}
interface EasyConnect {
  <TInjectProps, TDispatchProps = {}, TOwnProps = {}, TRootState = GlobalRootState<AnyObject>>(
    mapStateToProps: (state: TRootState) => TInjectProps, 
    mapDispatchToProps: (dispatch: any) => TDispatchProps
  ): Connector<TInjectProps & TDispatchProps, TOwnProps>;

  <TInjectProps, TOwnProps = {}, TRootState = GlobalRootState<AnyObject>>(
    mapStateToProps: (state: TRootState) => TInjectProps, 
  ): Connector<TInjectProps & DispatchProp, TOwnProps>;
}
```

实际上，在使用中还会面临一些更复杂的情况，例如页面组件中，是需要传入route的props，例如这样，就需要手动混合OwnProps

```typescript
type MapStateProps = { comments: CommentsState } 
type MapDispatchProps = ReturnType<typeof mapDispatchToProps>;
type OwnProps = RouteComponentProps<{ type: 'all' | 'article' }>;

export default connect<MapStateProps, MapDispatchProps, OwnProps>(
  (state) => ({ comments: state.comments }),
  mapDispatchToProps
)(function CommentsPage(props) {
  console.log(props.match.params.type) // all | article
})
```

当然也有一些很简单的场景, 跟不带类型比也没多几行，但同时享受了类型提示

```typescript
export default connect<GlobalRootState['global']>(
  (state) => ({ ...state.global }),
)(function Layout(props) {
  return ...
})
```

或者有redux-thunk中间件的dispatch, 则不能用redux提供的DispatchProps，需要混入thunk的dispatch函数类型等等

对于ts的类型，万变不离其宗，逐步分写的写，不要着急，特别是一些基础语法，需要不断的练习后就会熟悉起来了
