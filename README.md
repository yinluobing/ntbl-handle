```
^ Sunny
) 18.08.10
? F
```

[TOC]

# Handle.js

Handle.js，一个基于 koa 和 sequelize 的中间库，让你只专注于接口逻辑。

[api 文档](https://yeshimei.github.io/Handle.js/)。

# Usage

`Handle.js` 依赖 `merge` 库，使用前需先安装 `merge`。

```
npm i merge --save
```

```javascript
// 导入 Handle
// 导入 sequelize 模型
import { article } from '../models/db'

// 把 article 传入 Handle，并实例化
const article = new Handle(article)

// 生成一个查询当前模型所有数据的 koa 中间件
const find = article.findAll()

// 绑定到路由
router.get('/article/find', find)
```

# 开始

## 实例的模型方法一览

Handle.js 代理了部分 sequelize 模型方法，以下：

- GET 请求
    - findOne
    - findAll
    - findById
    - findOrCreate
    - findAndCountAll
    - findAndCount
    - findCreateFind
    - count
    - max
    - min
    - sun
- POST 请求
    - create
    - bulkCreate
    - update
    - destroy
    - increment
    - decrement


以上所有模型方法统称为快捷方法，调用后生成一个 async 的 koa 中间件函数，可以直接挂载路由。此外，还提供了一套对应的过程方法，调用后仅返回原生数据，可供进一步处理。以下：

- GET 请求
    - rawFindOne
    - rawFindAll
    - rawFindById
    - rawFindOrCreate
    - rawFindAndCountAll
    - rawFindAndCount
    - rawFindCreateFind
    - rawCount
    - rawMax
    - rawMin
    - rawSun
- POST 请求
    - rawCreate
    - rawBulkCreate
    - rawUpdate
    - rawDestroy
    - rawIncrement
    - rawDecrement

## where 子句简写


1. 字符串语法

```javascript

article.findOne('id')
// 内部将替换成
// {
//   where: { id:  d.id }
// }
//

```

GET 请求下，`d` 为 `ctx.query` 对象。POST 请求下，`d` 为 `ctx.request.body` 对象（默认为 x-www-form-urlencoded 类型）。

2. 多条件

```javascript

article.findOne(['id', 'uid'])
// 内部将替换成
// {
//   where: {
//     id:  d.id,
//     uid: d.uid
//  }
// }
//
```

3. 提供默认值

```javascript
article.findOne(['id', ['uid': 1]])
// 内部将替换成
// {
//   where: {
//     id:  d.id,
//     uid: 1
//  }
// }
//
```

4. 别名

```javascript
article.findOne([['id', '@aid'], ['uid', 1]])
// 内部将替换成
// {
//   where: {
//     id:  d.aid,
//     uid: 1
//  }
// }
//
```



## 更强大的函数式

函数式下支持所有 sequelize 模型选项中合法的选项。（具体请参考 [Model](http://docs.sequelizejs.com/class/lib/model.js~Model.html)）

```javascript
article.findOne(d => ({
  where: {
    id: d.aid,
    uid: 1
  },
}))
```

更复杂的分页查询。

```javascript
article.findAll(d => {
  const count = d.count || 5
  const page = d.page || 0
  return {

    where: {
      uid: d.uid,

    limit: count,
    offset: page * count
  }
}),
```

## 原生数据

`create()` 将自动把 `d` 作为数据增加到数据库中。

```javascript
article.create()
```

但是，对于 `increment()` 你可能只想增加某个字段的值，你可以直接指定原生数据。

```javascript
article.increment('comment_count', 'id')
```

注意，原生数据在内部不会做任何处理。



## 选项对象

请参考 [API 手册](https://yeshimei.github.io/Handle.js/Handle.html)

# 过程

`process` 将一系列操作封装到一起，提供了可以验证、过滤、重组等处理数据的能力，并且非常适合非事务性的多次数据库操作等场景。通常，需要配合过程方法使用。



```javascript
// 查询用户收藏的文章
articleStar.process(async function (d) {

  const {count = 5, page = 0, uid} = d

  const res = await this.rawFindAll({

    include: [
      {
        // 关联查询文章数据
        model: Article,
        // 并且查询文章的用户数据
        include: [User]
      }
    ],
    // 通过 uid 查询
    where: { uid },
    // 分页
    limit: count,
    offset: page * count
  })
  // 过滤数据，仅返回文章数据
  return res && res.map(d => d.article)
})
```

另外，当出现一些重复逻辑时，你也可以很方便的提取出来复用。


# 作用域

`scope` 是复用模型选项对象的最佳选择，它在内部使用了 `merge` 深度混合所有 `scope` 到选项对象中。外在而言，让你的代码更直观、简洁、易懂和良好的复用性，更易于重构和修改。

把上面的分页逻辑封装成为一个 提供参数的 scope。

```javascript
function pagination (defaultCount = 5, defaultPage = 0) => {
  return d => {
    const count = d.count || defaultCount
    const page = d.page || defaultPage
    return {
      limit: count,
      offset: page * count
    }
  }
}
```

然后，通过 `scope()` 方法到处复用。


```javascript
// 查询所有指定 uid 的数据，且每页返回 10 条数据
article.scope(pagination(10)).findAll('uid')
// 使用 pagination 函数提供默认值
article.scope(pagination()).findAll('uid')
// 其实，你可以省略掉 pagination 前面的 ()
article.scope(pagination).findAll('uid')

// 同时，scope 还可以是对象
articleStar.process(async function (d) {
  const include = {
    include: [
      {
        model: Article,
        include: [User]
      }
    ],
  }

  const res = await this
    .scope(pagination, include)
    .rawFindAll('uid')

  return res && res.map(d => d.article)
}),
```

`scope()` 方法合并的选项对象**仅在第一次被使用的方法上有效**。如果，想要让所有当前实例的模型方法都共享某些 scope ，可以通过 `defaultScope()` 添加。注意，每个 `scope` 都必须最终返回一个选项对象。


# 关联查询

`Include` 是 Handle.js 一个静态工具类，用于统一管理整个应用的关联数据。

```javascript
const { Include } = Handle

Include
  // 添加名为 article 的关联，并忽略查询一些属性。
  .add('article', {
    model: Article,
    attributes: {
      exclude: ['content', 'createdAt', 'updateAt']
    },
  })
  // 同时添加一个名为 user 的关联。
  .add('user', {
    model: User,
    attributes: {
      exclude: ['password', 'lock', 'freeze', 'power' ,'createdAt', 'updateAt']
    },
  })
```

然后，通过 `Handle.create()` 方法，可以简单的组合已添加关联的层级关系。

```javascript
articleStar.process(async function (d) {
    const res = await this
      .scope(pagination, Include.create({
        article: ['user']
      }))
      .rawFindAll('uid')
    return res && res.map(d => d.article)
  }),
```

`Handle` 在可高度复用关联数据的同时，又能以最小化、最简单的方式组织和管理。

# 事务

`transaction` 是通过 `process` 简单的对 sequelize 原生事务的封装。在使用上，和 `process` 完全一致。

```javascript
articleStar.transaction(async function (d) {
  /** 事务相关的处理 */
  return /** 返回处理后的数据 */
}),
```

# 钩子

`Handle` 在选项对象里提供了三个全局钩子 `before` 、`after`， `data`。


```javascript
new Handle(model, {
    // before 钩子在数据库操作之前执行
    before (data, ctx, next) {

    }
    // after 钩子在数据库操作之后执行
    after (result, ctx, next) {

    }
    // data 钩子可以在返回数据到前端之前和捕获异常之后做一些处理
    data (err, result, ctx, next) {

    }
})
```

如果你设置了全局钩子，每个快捷方法都会执行这些钩子，而过程方法则会忽略这些钩子，`process` 会在调用回调前执行 `before` 调用回调后执行 `after` 和 `data`。

# Mock 支持

Handle.js 提供了 mock 的接口，但它依赖于 mock 库。这里，我推荐使用 [mockjs](https://github.com/nuysoft/Mock)。

首先，你需要通过选项对象传入 mock 库以开启 `mock()` 方法

```javascript
import Mock from 'mockjs'

// 指定要使用 mockjs
const article = new Handle(Article)
article.options.mock = Mock
```

然后，使用实例的 `mock()` 方法。

```javascript
// 批量向 article 表中插入 20 条数据
article.mock({
 'data|20': [
   {
     title: '@ctitle',
     content: '@cparagraph'
   }
 ]
})
```

`mock()` 内部调用实例的 `bulkCreate()` 方法批量添加数据（注意，因为使用了原生数据入口，所以不会对数据做任何处理）


# Scopes 工具集（测试中）

`handle.js` 内置了一个 `Scopes` 工具集，帮助你快速编写复杂的接口，让你充分利用 `scope` 函数式编程所带来的优良特性。


```js
const Scopes = Handle.Scopes
const {option, where, pagination, fuzzyQuery, it} = Scopes

article
    .scope(
      // 可选字段
      // 通过 id 查询
      // 通过别名的 user_id 查询 uid 字段
      option('id', ['uid', '@user_id']),
      // 并且只返回 normalize = true 的数据
      where([normalize: true]),
      // 开启分页
      pagination,
      // 开启模糊查询
      fuzzyQuery,
    )
    .findAll()

// 另外通过 it 前置测试方法，可以在同一个接口中轻松实现针对不同开关的定制化功能

article
    .scope(
        // user 和 admin 是内部用于判断是管理员还是用户身份的标识
        // 管理员，可以查询任何数据
        it('admin', where('id'))
        // 普通用户只能查询自己的数据
        it('user', where('id', 'uid'))
        // 当 comment = true 时，带上评论数据
        it('comment', () => Inclue.create('comment'))
    )
    .findAll()
```

**where**

> where 子句的快捷写法

```js
/* {
     id: d.id,
     uid: d.user_id
     recomment: true
   }
*/
where('id', ['uid', '@user_id'] /* 别名 */, ['recomment', true] /* 默认值 */)

// 或者，使用对象语法

where({
  id: '@id',
  uid: '@user_id'
  recomment: true,
})
```

**option**

> 可选字段

用法与 **where** 一致。两者都是 **where 子句的简写方法**， 区别是 **option** 方法 指定的字段当调用接口时未指定是不会出现在 where 子句中的，而 **where** 方法是赋值一个未定义。

**fuzzyQuery(field = 'name', alias?)**

> 模糊查询

```
/*
  {
    name: {
      like: '%'' + d.name + '%''
    }
  }
*/
fuzzyQuery('name')

// 或者，起一个别名

/*
  {
    name: {
      like: '%'' + d.username + '%''
    }
  }
*/
fuzzyQuery('name', 'username')
```

**fuzzyQueryLeft(field = 'name', alias?)**

> 左匹配模糊查询

**fuzzyQueryRight(field = 'name', alias?)**

> 右匹配模糊查询

**pagination(page = 0, count = 5)**

> 分页

```
/*
  {
    limit: d.count,
    offset: d.page * d.count
  }
*/

// page 页数
// count 每页的数量
pagination()
```

**order**

> 排序

```
// 按字段的更新日期的降序排序
order([['updateAt', 'DESC']])
```

**includes**

> 添加一个或多个关联（如果存在层级复杂的关联，Include 更合适）

**set**

> 设置或修改 d 对象 (用法和 where 一致)

**del(filed, ...)**

> 删除 d 对象的字段

**merge**

> 深度合并多个对象或函数

**it**

> 前置断言

```js
// 当 field = true 执行 f1，否则 f2
// 相当于一个条件语句
it(field, f1, f2)
// field 可以有多个，每个可以是字符串、数组和函数
// 当 comment 为真时成立（包含隐式转换为）
it('comment')
// more = 2 时成立
it([['more': 2]])
// 当 more > 3 时成立
it(d => d.more > 3)

// 另外，f1 和 f2 参数接受一个函数或一组函数
it('commment', f1, [f2, f3, f4])
```




# 一些差异性的问题

（敬请期待）

# 插件系统

（暂无）


