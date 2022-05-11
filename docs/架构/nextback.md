---
id: nextback
title: nextback
---

## 概要

nextback 是下一代 ACM Homepage 后端。使用 Nest.js 框架（一种 TypeScript 后端框架）编写。

它会处理 GraphQL 请求，Nest.js 会通过给定的 GraphQL 请求智能地调用相关的 GraphQL 处理函数（位于文件 `modules/*/*.resolver.ts` 中），在处理之后返回符合请求的响应。

而 GraphQL 处理函数本身会去调用数据库处理函数（位于文件 `modules/*/*.service.ts`）中。

总体来说，nextback 位于数据库和 GraphQL 之间，充当两者的中间层、并且还需要支持：

- 处理 Cookie。
- 用户权限认证。

## 类型定义

在 GraphQL 方面，我们需要定义数据类型。但是我们知道，如果我们想要获得 TypeScript 支持的话，那么我们需要使用 TypeScript 来定义类型，而本身 GraphQL 也有一个独立的类型定义语言。我们采用 `@nestjs/graphql` 来使用 TypeScript 定义类型，并且 Nest.js 会自动根据我们定义的 TypeScript 类型来生成 GraphQL 类型定义（GraphQL schema），保证两者之间的一致性。

同理，在数据库方面，我们也需要定义类型。我们使用 TypeORM 提供的接口，而非直接操作。

## 代码组织

我们的代码组织分为三个主要的文件夹：

- `modules`，按照模组来进行划分。程序的主要入口，而根模块，是 `./src/app.module.ts`。
- `entitys`，定义数据库类型。使用到了 TypeORM。
- `models`，定义 GraphQL 类型。使用到了 `@nestjs/graphql`。
- `utils`，一些辅助性的函数，比如 JWT 处理、Cookie 处理、bcrypt 处理等。

每一个模组都是一个文件夹，并包含了 `index.ts` 来作为模组的入口。同级的还有 `*.service.ts` 和 `*.resolve.ts`，用于处理数据库中的数据，和 GraphQL。

其中 `*.service.ts` 专注于和数据库打交道，而 `*.resolver.ts` 则需要处理：

- 调用 `*.service.ts` 的接口来返回数据。
- 调用 `utils/` 下的相关函数，来进行权限验证、解析 Cookie 和设置 HTTPOnly Cookie。

### Entitys

我们知道 `src/entitys` 用于定义所有实体。而涉及到实体的操作总结下来就四个：

- 增加。
- 删除。
- 修改。
- 查找。

在 TypeORM 中，如果我们使用到了并不存在的字段，就不能通过类型检查。这种类型检查对于删除、查找和修改都十分完美。但是唯独对于增加操作的支持并不友好：有些字段是必须的，但是即使传入一个空对象，TypeORM 的 `create` 方法也能通过类型检测。因此我们会在 `src/entitys` 中每一个实体对应的类定义两个方法：一个私有构造器方法，返回空对象；一个静态公有异步方法 `build`，接受 `Repository<T>` 和一个对象，用于完成类型检查，和补足剩余字段的作用，并返回合格的对象（由工程师保证）。（比如对于用户而样，我们需要让第一个用户为管理员，我们只需要传递其余的字段即可，`build` 方法会补足上 `isAdmin` 字段）。

这样，我们只要确保了一下两点即可保证类型安全：

- 构造器是私有的和 `build` 方法两者实现没有问题。
- 增加实体对象的时候，我们使用的是 `build` 方法（保证类型安全），而非直接传入一个裸对象。

Q：我们可以修改 `create` 方法的类型声明使得它强制只接受 `build` 返回的类型吗？

A：这个问题我有问，维护者告诉我只能这样。（https://github.com/typeorm/typeorm/issues/8973）

## 命名规范和抽象级别

所有字段均采用驼峰命名法。其中类型用大写字母开头，而其他的用小写类型。

### GraphQL

对于 Query，我们一般会查询单数个对象或者复数个对象。我们将其命名为单纯的名词。其中由于英语单词并不说所有都具有单数复数形式我们：让单数的对象保持单数，让复数的对象保持末尾增加 `List` 后缀的约定。（比如新闻 news 没有复数形式，我们如果想要得到 news 数组的话，则我们需要将其命名为 `newsList`。

比如我们可以用于 Query 的名字有（经过思考，如果需要带上限定属性从参数的话，还是应该加上类似 `ByXxxxx` 的后缀）：

- `newsList`；
- `tagList`；
- `newsListByTag`。

而可以用于 Mutation 的名字有：

- `addTagToNews`；
- `removeTagFromNews`；
- `createUser`。

此外，我们知道在 GraphQL 中我们可以用 `input` 类型来减少参数的个数，但是这并非是银弹，我们也应该避免不必要的 `input` 类型，除了：

- 我们的确可以把它抽象成 **一个** 对象。比如说我们尝试新建一个新闻，那么我们可以把构成新闻的字段们抽象成一个 `input`。

在这里，我们约定新建对象的时候，使用 `input` 这种 GraphQL 类型，且 GraphQL 参数名不应该为 `input`。