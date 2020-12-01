# Golang单元测试实践指南

## 基础

### 概念
单元测试是指对软件中最小可测试单元在于程序其他部分相隔离的情况下进行检查和校验。单元测试是白盒测试，通常由开发来完成，测试的对象可以是函数，也可以是类，而测试的目标是来检查程序是否按照预期的逻辑执行。好的单元测试会遵守 AIR 原则，以便让测试用例更加地复合规范。

### AIR原则
AIR，即 Automatic（自动化）、Indepenndent（独立性）、Repleatable（可重复）的简写：
- 自动化：单元测试应该是自动执行，自动校验，自动给出结果，若需要人工检查（如将结果输出到控制台）的单元测试，不是好的单元测试；
- 独立性：单元测试应该是可以独立运行的，测试用例之间无依赖和执行次序，用例内部对外部资源也无依赖；
- 可重复：单元测试应该是可以重复执行的，每次的结果都是稳定可靠的；

我们在对软件进行测试的时候很容易混淆单元测试的概念，如果存在以下问题，那么你写的很可能不是单元测试：

- 是否能一键运行写过的所有测试用例？
- 两周前写的单元测试现在是否能正常运行？
- 自己写的单元测试在同事电脑里是否能正常运行？

你可能会有疑问，如果说不能，那做的都是些什么测试呢？实际上你做的是集成测试。

### 集成测试

集成测试的测试对象是整个系统或者某个功能模块，比如测试用户注册、登录功能是否正常，是一种端到端的测试。如果测试用例使用到真实的系统时间、真实的文件系统、真实的数据库，亦或者是其他真实的外部依赖，那么该测试已经进入到了集成测试的领域。

![](images/unittest.png)

对集成测试以及后续的其他测试，本节不做过多展开，有兴趣的同学可以自行学习。

### 价值意义

可能有同学会问，引入单元测试能解决什么痛点，带来什么价值呢？在笔者看来，单元测试可以带来代码质量提升、低成本确认问题、缩短上线流程等提高人效的价值，毕竟研发事无巨细地检查每一个功能，也不如程序自动完成来得全面和准确。另外也减少低级错误导致测试人员与研发人员之间的信任问题，也可以让测试投出更多宝贵的时间参与到产品讨论和需求评审。

#### 提升代码质量
好的代码，可测性是一个衡量的标准，不好测试的代码，其本身的抽象性、模块性、可维护性都可能存在问题。例如不符合单一职责、接口隔离等设计原则，或者依赖了全局，而可测试的代码，往往其质量相对会高一些。

我们来个反面例子看看不好测的伪代码长啥样：

```golang
type IUserDao interface {
  QueryUserFromDB(name string) User
}

type Service struct {}

func (s *Service) QueryUser(name string) {
  // 创建接口 IUserDao 的实现
  userDao := NewUserDao()
  // 调用方法获得结果并进行后续操作
  userDao.QueryUserFromDB(name)
  ....
}
```

这种依赖关系是强耦合的代码，你在对它进行单元测试的时候会发现，即使你将接口 IUserDao 的模拟对象（mock）创建出来，也没有办法将 mock 对象放入 QueryUser 进行替换。反过来，如果你的单元测试很容易编写，测试性好，也能说明代码质量相对较高。

#### 低成本确认问题
单元测试最大的价值在于它们的反馈的速度，越早发现缺陷修复成本越低，而且业务迭代和代码重构的过程中单元测试可以确保代码的正确性，降低各种修改和重构的风险，也能让后续的集成测试和验收测试更容易通过。


### 入门示例

笔者用个非常简单的示例带大家入门，假设我们存在一个负责计算两数总和的 Add 函数：

```golang
// main.go
package main

func Add (num1, num2 int) int {
  return num1 + num2
} 
```

现在我们给它完善单元测试，先构建参数、然后调用函数、最后断言结果：

```golang
// main_test.go
package main

import (
	"testing"

	"github.com/stretchr/testify/assert"
)

func TestAdd(t *testing.T) {
  // 构建
  mockNum1 := 1
  mockNum2 := 2
  mockExpected := 3

  // 调用
  sum := Add(mockNum1, mockNum2)

  // 断言
	assert.Equal(t, mockExpected , sum)
}
```

我们在终端项目下运行 `go test -v` 就可以看到测试结果：
```text
=== RUN   TestAdd
--- PASS: TestAdd (0.00s)
PASS
ok      unittest 0.013s
```

练习完以上示例你就已经完成入门了，但接下来需要掌握更多的技巧，包括懂得识别依赖、明白如何破除依赖、掌握好用的测试框架再加上适当的编程技巧等，才能写出优秀的贴合实战的单元测试。

## 进阶

### 识别依赖
在实际的项目中，我们的测试单元很可能存在对外部的依赖，而单元测试的 Indepenndent 原则要求测试用例应该要独立运行的，即我们只需要保证被测试的单元内部逻辑正确即可，不需要真正依赖外部资源，因此我们需要做的第一件事情就是识别依赖。

常见的外部依赖项包括：文件I/O操作，如配置文件的读写；网络I/O，如外部HTTP接口调用、RPC调用、对数据库或者消息队列等资源的请求；又或者可以系统时间，项目中没有开发完成的模块等等。

在识别依赖后，就应该考虑怎么破除外部依赖。

### 破除依赖
常用的破除依赖的方法包括：间谍（spy）、存根（stub，也成为打桩）、模拟对象（mock）和伪对象（fake）等：
- spy: 
- stub:
- mock:
- fake:

### 测试框架
掌握基本的测试框架可以帮助我们快速地生成破除依赖的 stub 和 mock 等资源，这里笔者推荐几个好用的测试框架获工具，包括 gomock、monkey 和 stretchr/testify等。

#### gomock
`gomock`是 Golang 官方的维护的测试框架，可以对基于接口的模块提供很好的 mock 支持，通过它的概述我们可以快速了解到它的使用方式：

![](images/gomock.png)
1. 第一步：逻辑代码中定义好你需要接口；
2. 第二步：使用gomock的mockgen工具生成mock文件；
3. 第三步：引入mock文件，对方法进行打桩并断言后续逻辑；

更多详情可以访问它的[使用文档](https://pkg.go.dev/github.com/golang/mock/gomock)和[代码仓库](https://github.com/golang/mock)。

#### monkey
`monkey`框架通过在运行时重写执行文件来实现 stub 操作，该库存很久没有更新了且当前已经处于归档状态，但使用起来真的很方便，更多详情可以参考它的[代码仓库](https://github.com/bouk/monkey)。


#### stretchr/testify

`stretchr/testify`可以提供打印友好、易于阅读的的断言（assert）包，更多详情可以参考它的[代码仓库](https://github.com/stretchr/testify)。

### 编码技巧
最后，如果一段代码我们很难为其编写单元测试，或者需要使用单元测试框架中很高级的特性，那往往意味着代码的设计存在问题。在这里笔记一般建议大家先掌握两个技巧：面向接口而非实现编程以及依赖注入。

#### 面向接口而非实现编程

先谈一下对接口的理解：接口是对行为的一种抽象，相当于一组协议或者契约，表示 has-a 的关系，它是一种自上而下的设计思路，先设计接口，再考虑代码实现，解决的是解耦问题。将接口和实现分离，可以封装不稳定的实现，暴露稳定的接口，上游系统面向接口而非实现编程，这样即使实现发生变化的时候，调用方也基本不需要改动，以此来降低耦合和提高扩展性。

在 Golang 语言中，对于任何数据类型，只要它的方法集合中包含了一个接口的全部方法，那么这个类型就是对这个接口的实现（即鸭子类型）。利用基于接口而非实现编程的设计思想，我们可以通过代理类实现和原始类相同的接口，再将原始类进行替换（静态代理），从而实现 mock 的行为。我们来个简单例子说明：

```golang
// 定义接口
type IUserDao interface {
  QueryUserFromDB(name string) User
}

// 原始实现 UserDao
type UserDao struct{}

func NewUserDao() IUserDao {
  return UserDao{}
}

func (UserDao) QueryUserFromDB(name string) User {
  // do something in user dao
  return User{}
}

// 代理实现 UserDaoMock
type UserDaoMock struct {}

func NewUserDaoMock() IUserDao {
  return UserDaoMock {}
}

func (NewUserDaoMock) QueryUserFromDB(name string) User {
  // do somethong in user dao mock 
  return User{}
}
```

上层在使用时只关心抽象接口 IUserDao，而不是具体实现 UserDao，这样我们的单元测试就可以很轻松地使用 UserDaoMock 对象完成对 UserDao 的 mock 动作，进而开展后续的操作。


#### 依赖注入
依赖注入指的是不通过显式创建的方式（new）在类内部创建依赖类的对象，而是将依赖的对象在外部创建好之后，通过构造函数、函数传参等方式传递给类使用。在没有使用依赖注入框架的情况下，可以这样在 Golang 中实现依赖注入：

```golang
type UserService struct {
  userDao IUserDao
}

func NewUserService (userDao IUserDao) {
  return UserService{ userDao: userDao }
}

func (s *Service) Query(name string) {
  s.userDao.QueryUserFromDB(name)
}

// 调用方
func main () {
  userService := NewUserService(NewUserDao())
}
```

或者你的`NewUserService`方法可以写得更简单一点：
```golang
func NewUserService () {
  return UserService{ userDao: NewUserDao()}
}
```

使用依赖注入而不是显式创建依赖，是编写出可测性良好代码的关键一步，后续的实践过程我们将继续探讨这个过程。





## 实践


## 思考

至此，希望以上知识能帮助你更好地掌握到如何在 Golang 中使用单元测试，最后，笔者也留下几个思考题，欢迎留言评论：

1. 单元测试的粒度到底要做到多细？
2. 单元测试的成本和受益如何平衡？
3. 业界崇尚的TDD是否为最佳实践？