---
layout: post
title: 关于单元测试的总结及思考
category: 方法论
tag: [java, 质量保障]
---

### 什么是单元测试

#### 基本概念
[点击查看](https://en.wikipedia.org/wiki/Unit_testing) Wikipedia 词条，要点：

* 软件测试中的一种方法，验证程序运行符合预期。
* 将程序逻辑切分成单元或者模块，按照最小单元进行测试。面向对象设计的语言中，unit 通常是一个 class/method。
* 理想的 unit 具备良好的独立性，依赖以 mock/stub 的方式注入。关于stub和mock的区别，可以查看martinfowler的博客：[Mocks Aren't Stubs](https://martinfowler.com/articles/mocksArentStubs.html)。
* 由开发人员或者白盒测试人员编写。

那么如何验证程序运行是否符合预期呢？以OOP为例，对象主要由两点来描述：

* <em>State</em>，对象的状态，可以理解为某一时刻的属性值。
* <em>Behavior</em>，对象的行为，接收到外部消息后进行状态变更的过程。

Unit testing 可以基于这两点来做验证，即 state assert 和 behavior verification。

#### 单元测试在整个测试流程中的位置
一般测试流程中有如下几个阶段，并按照时间顺序进行：
<p align="center">
<img src="/assets/images/unit-test-2.png" alt="test" style="height: 60px;"/>
</p>
* 单元测试，由开发主导，验证基础单元/模块功能。
* 集成测试，和外部环境联通，验证跨单元的功能。
* 冒烟测试，属于端到端的测试，一般由开发主导，验证程序基本流程没问题后，再交付给测试人员进行后续测试。
* 回归测试，由测试主导，验证新增功能是否影响原有功能。

### 单元测试的价值
很多团队并不怎么看重单元测试，认为价值不大且投入精力多，这点对于具有良好质量意识的个人来说确实如此，但对于一个团队，需要更多的从管理方法上去提高整体质量。从自己写单元测试来看，我感受有以下几点价值：

* 测试驱动开发，而不是前端驱动。对于纯后端开发人员，写完某个功能，不必等待前端开发完一起联调，自己写单元测试效率更高。
* 前人种树后人乘凉，完善的单元测试有利于后面做重构，降低了代码维护成本。
* 代码质量的一个度量，数据统计方便于管理做决策。
* 在著名的"测试金字塔"中，单元测试处于底层，相比上层的测试，单测运行快、成本低，更能指出问题所在并快速bugfix，关于这一点可以延伸阅读下：[Google testing: Just Say No to More End-to-End Tests](https://testing.googleblog.com/2015/04/just-say-no-to-more-end-to-end-tests.html)。
<p align="center">
<img src="/assets/images/unit-test-3.png" alt="testPyramid" style="width: 350px;"/>
</p>
 
### 实施单元测试的难点
* mock数据难，经常把单元测试写成了集成测试。
* 需要与发布、持续集成环境整合才能体现其主动性，需要有专门的测试开发负责推进。
* 并不是开发中必须有的一个流程，没有强制的衡量标准。
* 对于已经发展庞大但缺失单测的项目，由于代码不符合单测工具所约定的规范，再去补充单元测试很难（亲身经历），需要对项目进行拆分及规范化。

### 如何做单元测试

#### 1、制定需要遵守的规范

* 代码布局

```
src/
├── main
│   ├── java
│   └── resources
└── test
	├── java
	└── resources
```
详细请参考 [maven工程标准结构](http://maven.apache.org/guides/introduction/introduction-to-the-standard-directory-layout.html)

* 类、方法命名

```java
// 类命名以Test结尾	
public class TokenServiceTest extends AbstractTestCase {

	@Resource
	private TokenService tokenService;

	// 方法命名以Test结尾
	@Test
	public void insertTest() {
		MockUp<TokenDao> tokenDaoMockUp = new MockUp<TokenDao>() {
			@Mock
			Integer insert(Token token) {
				return 1;
			}
		};
		ReflectionTestUtils.setField(tokenService, "tokenDao", tokenDaoMockUp.getMockInstance());
		tokenService.insert();
	}
}
```

* 代码逻辑分层，良好的分层有助于更加清晰的单元测试编写，请参考上一篇文章 [Web分层设计模型](http://blog.lastww.com/2017/07/03/web-service-layed-design)。分层之后，每一层应该做什么？

```
1) web层
验证 PATH，REQUEST_METHOD 配置是否正确，验证 RESPONSE_CODE是否正常。这一层实际上属于端到端的测试，一般项目中很难测出问题，大部分情况下可以省略。

2) facade层
验证参数校验，数据组装转换是否符合预期。

3) service/component层
验证业务逻辑、流程是否正常，需要mock掉外部服务依赖。

4) cache层
验证缓存配置是否正常，缓存读写命令是否正常，用例需要包含清理数据的逻辑。

5) dao层
验证sql逻辑是否正确，主键是否正常。如果用到测试数据库，测试完后需要回滚数据。
```

* 与发布流程整合

将单元测试集成到发布流程中，只有单测单通才能进行review和提测。

#### 2、选择好用的工具和框架
关于有哪些工具和框架，以及优缺点，这里就不列举了，直接推荐以下两个：

* TestNG: a testing framework inspired from JUnit and NUnit but introducing some new functionalities, 网站：http://testng.org/doc/
* jmockit: includes APIs for mocking, faking, and integration testing, and a code coverage tool. 网站：http://jmockit.org/about.html。API 介绍如下：

<p align="center">
<img src="/assets/images/unit-test-1.png" alt="jmockit" style="width: 600px;"/>
</p>

#### 3、定期评估

* 覆盖率指标需要保持在一个固定值之上，如果下降需要有人跟进。

### 最终，别忘了
单元测试不仅仅是开发的事，需要测试一起配合推动。

### 参考链接

1、[Martinfowler: TestPyramid](https://martinfowler.com/bliki/TestPyramid.html)

2、[Google testing: Just Say No to More End-to-End Tests](https://testing.googleblog.com/2015/04/just-say-no-to-more-end-to-end-tests.html)

3、[What is Unit test, Integration Test, Smoke test, Regression Test?](https://stackoverflow.com/questions/520064/what-is-unit-test-integration-test-smoke-test-regression-test)

4、[Unit Testing Guidelines](http://geosoft.no/development/unittesting.html)

5、[Martinfowler: Mocks Aren't Stubs](https://martinfowler.com/articles/mocksArentStubs.html)
