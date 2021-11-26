---
title: 常见问题
slug: /issues
order: 1
---

## 推荐配置

如果在接入子应用的时候，出现了拿不到子应用导出的问题的时候。可以先按照以下步骤自查：

1. 检查子应用是否正确 `export` 了 `provider`。
2. 检查子应用是否配置了 `webpack` 的 `output` 配置。
3. 若为 `js` 入口，需要保证子应用的资源被打包成了但 `bundle`，若有部分依赖未被打包成 `bundle` 会导致无法正常加载

```js
// webpack.config.js
{
  output: {
    libraryTarget: 'umd',
    globalObject: 'window',
    jsonpFunction: 'masterWebpackJsonp', // 于 `webpackjsonp` 可能会冲突，所以可以给子应用和主应用配置不同的 `webpackjsonp `函数。webpack 5 保证 package.json 中的 name 各不相同即可
  },
}
```

## 刷新直接返回子应用内容

> 问题原因

- 微前端是一个 SPA 应用，加载子应用是通过 SPA 模式来动态的加载其他子应用内容
- 当访问到主应用的某个路径下激活子应用时是不存在这个路径下的静态资源的，从而 failback 到主应用的内容
- Garfish 在初始化时根据当前路径来确定加载的子应用
- 如果在访问主应用的某个路径时来加载子应用，而这个地址已经存在一个静态资源，浏览器将会直接返回该资源

> 解决方案

- 子应用的资源地址不要和主应用上面激活路径的资源地址一致

## 子应用的接口和资源路径不正确

尽可能将子应用的接口请求和资源路径调整为绝对路径

1. 子应用在独立运行时，使用相对路径的接口，接口请求的路径是，当前页面域名+相对路径
2. 但是在主应用时，子应用使用相对路径的接口，请求的路径按道理来说还是，当前域名+相对路径

当在微前端的场景下如果 Garfish 让子应用走「当前域名+相对路径」会发生更多的异常请求（hmr 热更新、websock、server worker ...），因为子应用的域名并不一定是与主应用一致，因此 Garfish 框架会对相对路径的资源和请求去进行修正，修正的参照物为基础域名为子应用的路径，在本地开发时可能是正常的，但是发到线上出现问题，原因在于发布到线上之后，Goofy web 为了提升子应用资源加载的性能，子应用的入口会走 CDN。因此参照的基础路径就变为了 CDN 前缀。那么此时子应用的相对路径请求就变为了 CDN 前缀。这一块做了很对权衡，因为 hmr、websock、server worker 这些内容可能难以被用户控制，所以默认走的还是修正模式。

## 为什么主应用仅支持 history 模式？

- 目前 Garfish 是通过命名空间去避免应用间的路由发生冲突的。

- 主应用仅支持 `history` 模式的原因在于，`hash` 路由无法作为子应用的基础路由，从而可能导致主应用和子应用发生路由冲突。

## 根路由作为子应用的激活条件？

- 有部分业务想将根路径作为子应用的激活条件，例如 `garfish.bytedance.net` 就触发子应用的渲染，由于目前子应用 **字符串的激活条件为最短匹配原则**，若子应用 `activeWhen: '/'` 表明 `'/xxx'` 都会激活。

- 之所以为最短匹配原则的原因在于，我们需要判断是否某个子应用的子路由被激活，如果可能是某个子应用的子路由，我们则可能激活该应用。

- 之所以有该限制是由于若某个子应用的激活条件为 `/`，则该应用的 `/xx` 都可能为改子应用的子路由，则可能与其他应用产生冲突，造成混乱。

## 子应用拿到 basename 的作用？

为什么推荐子应用拿通过 `provider` 传递过来的 `basename` 作为子应用的 `basename`，有些业务方在实际过程中直接通过约定形式直接在子应用增加 `basename` 已到达隔离的效果，但该使用方式可能导致主应用如果变更 `basename` 可能导致子应用无法一起变更生效。

例如：

1. 当前主应用访问到 `garfish.bytedance.net` 即可访问到该站点的主页，当前 `basename` 为 `/`，子应用 vue，访问路径为 `garfish.bytedance.net/vue`

2. 如果主应用想更改 `basename` 为 `/site`，则主应用的访问路径变为`garfish.bytedance.net/site`，子应用 vue 的访问路径变为 `garfish.bytedance.net/site/vue`

3. 所以推荐子应用直接将 `provider` 中传递的 `basename` 作为自身应用的基础路由，以保证主应用在变更路由之后，子应用的相对路径还是符合整体变化

> 微前端场景下，每个子应用可能都有自己的路由场景，为保证子应用间路由不冲突，Garfish 框架将配置的 `basename` + `子应用的 activeWhen` 匹配的路径作为子应用的基路径。

- 若在 Garfish 上配置 `basename: /demo`，子应用的激活路径为：`/vue2`，则子应用得到的激活路径为：`/demo/vue2`
- 若子应用的激活条件为函数，在每次发生路由变化时会通过校验子应用的激活函数若函数返回 `true` 表明符合当前激活条件将触发路由激活，
- Garfish 会将当前的路径传入激活函数分割以得到子应用的最长激活路径，并将 `basename` + `子应用最长激活路径传` 给子应用参数
- **子应用如果本身具备路由，在微前端的场景下，必须把 basename 作为子应用的基础路径，没有基础路由，子应用的路由可能与主应用和其他应用发生冲突**

## 主子应用样式冲突

### arco-design 多版本样式冲突

1. [Arco-design 全局配置 ConfigProvider](https://arco.design/react/components/config-provider)
2. 给子应用分别设置不同的 `prefixCls` 前缀

### ant-design 样式冲突

1. 配置 `webpack` 配置

```js
module.exports = {
  module: {
    rules: [
      {
        test: /\.less$/i,
        use: [
          { loader: 'style-loader' },
          { loader: 'css-loader' },
          {
            loader: 'less-loader',
            options: {
              modifyVars: {
                '@ant-prefix': 'define-prefix', // 定制自己的前缀
              },
              javascriptEnabled: true,
            },
          },
        ],
      },
    ],
  },
};
```

2. 配置公共前缀：[antdesign-config](https://ant.design/components/config-provider/#API)

```js
import { ConfigProvider } from 'antd';

export default () => (
  <ConfigProvider prefixCls="define-prefix">
    <App />
  </ConfigProvider>
);
```
