# 一.快速开始

liteflow需要你的项目使用maven
## 1.1 依赖

```xml
<dependency>
	<groupId>com.yomahub</groupId>
    <artifactId>liteflow-core</artifactId>
	<version>${liteFlow.version}</version>
</dependency>
```
最新版本为<font color=red>*2.3.1*</font>，为稳定版本，目前jar包已上传中央仓库，可以直接依赖到

## 1.2 流程配置文件

如果你的项目不依赖spring框架（现在还有不依赖spring的项目吗？没关系，liteflow也为你提供了配置）

如果你的项目使用spring或者springboot，请跳过此章节

```xml
<?xml version="1.0" encoding="UTF-8"?>
<flow>
	<nodes>
		<node id="a" class="com.yomahub.liteflow.test.component.AComponent"/>
		<node id="b" class="com.yomahub.liteflow.test.component.BComponent"/>
		<node id="c" class="com.yomahub.liteflow.test.component.CComponent"/>
		<node id="d" class="com.yomahub.liteflow.test.component.DComponent"/>
		<node id="e" class="com.yomahub.liteflow.test.component.EComponent"/>
	</nodes>
	
	<chain name="demoChain">
		<then value="a,b,c"/> <!-- then表示串行 -->
		<when value="d,e"/> <!-- when表示并行 -->
	</chain>
</flow>
```

component为组件，这里你需要实现这些组件，每个组件继承`NodeComponent`类
```java
public class AComponent extends NodeComponent {

	@Override
	public void process() {
		String str = this.getSlot().getRequestData();
		System.out.println(str);
		System.out.println("Acomponent executed!");
	}
}
```

chain为流程链，每个链上可配置多个组件节点。目前执行的模式分串行和并行2种。  
串行标签为`then`，并行标签为`when`。  
在串行的模式下，以下2种写法是等价的,可以根据业务需要来把不同种类的节点放一行里。  
```xml
<then value="a,b,c,d"/>
```
```xml
<then value="a,b"/>
<then value="c,d"/>
```

## 1.3 执行流程链

(不依赖任何第三方框架的写法)

```java
FlowExecutor executor = new FlowExecutor();
executor.setRulePath(Arrays.asList(new String[]{"/config/flow.xml"}));
executor.init();
Slot slot = executor.execute("demoChain", "arg");
```

如果你的项目使用spring，推荐参考[和Spring进行集成](https://bryan31.gitee.io/liteflow/#/?id=%e4%ba%8c%e5%92%8cspring-boot%e9%9b%86%e6%88%90)

# 二.和Spring Boot集成

## 2.1 依赖

liteFlow提供了liteflow-spring-boot-starter依赖包，提供自动装配功能

```xml
<dependency>
  <groupId>com.yomahub</groupId>
  <artifactId>liteflow-spring-boot-starter</artifactId>
  <version>2.3.1</version>
</dependency>
```

## 2.2 配置

在依赖了以上jar包后。

在application.properties里加上配置地址后，就可以在容器中依赖拿到`FlowExecutor`实例

```properties
liteflow.rule-source=config/flow.xml
```

工程中的liteflow-test演示了如何在springboot下进行快速配置

# 三.和spring进行集成

针对于使用了spring但没有使用springboot的项目

## 3.1 流程配置可以省略的部分

流程配置中的`nodes`节点，可以不用配置了，支持spring的自动扫描方式。你需要在你的spring配置文件中定义
```xml
<context:component-scan base-package="com.yomahub.liteflow.test.component" />
<bean class="ComponentScaner"/>
```

当然，你的组件节点也需要注册进spirng容器
```java
@Component("a")
public class AComponent extends NodeComponent 
	@Override
	public void process() {
		String str = this.getSlot().getRequestData();
		System.out.println(str);
		System.out.println("Acomponent executed!");
	}
}
```

## 3.2 Spring中执行器的配置

```xml
<bean id="flowExecutor" class="FlowExecutor">
	<property name="rulePath">
		<list>
			<value>/config/flow.xml</value>
		</list>
	</property>
</bean>
```
然后你的项目中通过spring拿到执行器进行调用流程。



# 四.和zookeeper进行集成

liteFlow支持把配置放在zk集群中，并支持实时修改流程  
你只需在原来配置流程的地方，把本地xml路径换成zk地址就ok了

## 4.1 Spring配置

```xml
<!-- 这种是zk方式配置 -->
<bean id="flowExecutor" class="FlowExecutor">
	<property name="rulePath">
		<list>
			<value>127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183</value>
		</list>
	</property>
	<!--这个不配置就用默认的/lite-flow/flow节点 -->
	<property name="zkNode" value="/lite-flow/customFlow"/>
</bean>
```

## 4.2 Springboot配置

```properties
liteflow.rule-source=127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183
```



# 五.使用自定义的配置源

## 5.1 创建自定义配置源的类

如果你不想用本地的配置，也不打算使用zk作为配置持久化工具。liteFlow支持自定义的配置源的扩展点。  
在你的项目中创建一个类继承`ClassXmlFlowParser`这个类  
```java
public class TestCustomParser extends ClassXmlFlowParser {

	@Override
	public String parseCustom() {
		System.out.println("进入自定义parser");
		String xmlContent = null;
		//这里需要自己扩展从自定义的地方获取配置
		return xmlContent;
	}
}
```

## 5.2 Spring配置

spring中需要改的地方还是执行器的配置，只需要在配置的路径地方放入自定义类的类路径即可  
```xml
<bean id="flowExecutor" class="FlowExecutor">
	<property name="rulePath">
		<list>
			<value>com.yomahub.liteflow.test.TestCustomParser</value>
		</list>
	</property>
</bean>
```

## 5.3 Springboot配置

```properties
liteflow.rule-source=com.yomahub.liteflow.test.TestCustomParser
```



# 六.架构设计

## 6.1 组件编排式流程引擎架构设计

![architecture_image](media/architecture.jpg)

# 七.接入详细指南

## 7.1 执行器

执行器`FlowExecutor`用来执行一个流程，用法为  
```java
public <T extends Slot> T execute(String chainId,Object param);
```
第一个参数为流程ID，第二个参数为流程入参  
返回为`Slot`接口的子类，以上方法所返回的为默认的实现类`DefaultSlot`  

!> 实际在使用时，并不推荐用默认的`DefaultSlot`，推荐自己新建一个类继承`AbsSlot`类  
这是因为默认Slot实现类里面大多数都存放元数据，给用户扩展的数据存储是一个弱类型的Map  
而用户自定义的Slot可以实现强类型的数据，这样对开发者更加友好

推荐使用带自定义Slot的执行接口：
```java
public <T extends Slot> T execute(String chainId,Object param,Class<? extends Slot> slotClazz);
```

关于`Slot`的说明，请参照[数据槽](https://bryan31.gitee.io/liteflow/#/?id=_72%e6%95%b0%e6%8d%ae%e6%a7%bd)

## 7.2 数据槽

在执行器执行流程时会分配唯一的一个数据槽给这个请求。不同请求的数据槽是完全隔离的。  
数据槽实际上是一个Map，里面存放着liteFlow的元数据  
比如可以通过`getRequestData`获得流程的初始参数，通过`getChainName`获取流程的命名，通过`setInput`,`getInput`,`setOutput`,`getOutput`设置和获取属于某个组件专有的数据对象。当然也提供了最通用的方法`setData`和`getData`用来设置和获取业务的数据。

!> 不过这里还是推荐扩展出自定义的Slot（上一小章阐述了原因），自定义的Slot更加友好。更加贴合业务。

## 7.3 组件节点

组件节点需要继承`NodeComponent`类
需要实现`process`方法  
但是推荐实现`isAccess`方法，表示是否进入该节点，可以用于业务参数的预先判断  

其他几个可以覆盖的方法有：  
方法`isContinueOnError`：表示出错是否继续往下执行下一个组件  
方法`isEnd`：表示是否立即结束整个流程 ，也可以在业务日志里根据业务判断来调用this.setIsEnd(true)来结束整个流程。

在组件节点里，随时可以通过方法`getSlot`获取当前的数据槽，从而可以获取任何数据。

## 7.4 路由节点

在实际业务中，往往要通过动态的业务逻辑判断到底接下去该执行哪一个节点  
```xml
<chain name="chain1">
    <then value="a,c(b|d)"/> <!-- cond是路由节点，根据c里的逻辑决定路由到b节点还是d节点,可以配置多个 -->
    <then value="e,f,g"/>
</chain>
```
利用表达式可以很方便的进行条件的判断  
c节点是用来路由的，被称为条件节点，这种节点需要继承`NodeCondComponent`类  
需要实现方法`processCond`，这个方法需要返回`String`类型，就是具体的结果节点

## 7.5 子流程

liteflow从`2.3.0`开始支持显式子流程，在xml里配置的节点，可以是节点，也可以是流程id。比如，你可以这么配置

```xml
<chain name="chain3">
    <then value="a,c,strategy1,g"/>
</chain>
<chain name="strategy1">
    <then value="m(m1|m2|strategy2)"/>
</chain>
<chain name="strategy2">
    <then value="q,p(p1|p2)"/>
</chain>
```

这样是不是很清晰

liteflow支持无穷的嵌套结构，只要你想的出。可以完成相对复杂的流程。

!> 如果存在相同名字的节点和流程，优先节点。

## 7.6 节点内执行流程

liteflow支持在一个节点里通过代码调用另外一条流程， 这个流程关系在xml中并不会显示。所以这里称之为隐式调用。

隐式调用可以完成更为复杂的子流程，比如循环调用：

```java
@Component("h")
public class HComponent extends NodeComponent {

	@Resource
	private FlowExecutor flowExecutor;
	
	@Override
	public void process() {
		System.out.println("Hcomponent executed!");
    for(int i=0;i<10;i++){
      flowExecutor.invoke("strategy1",3, DefaultSlot.class, this.getSlotIndex());
    }
	}
	
}
```
这段代码演示了在某个业务节点内调用另外一个流程链的方法

## 7.7 步骤打印

liteFlow在执行每一条流程链后会打印步骤，这个步骤是程序实际执行的顺序  
样例如下：

```
a==>c==>m==>q==>p==>p1==>g
```

## 7.8 监控

liteFlow提供了简单的监控，目前只统计一个指标：每个组件的平均耗时  
每5分钟会打印一次，并且是根据耗时时长倒序排的。 

# 八.示例工程

## 8.1 测试工程

在项目内有2个简单的测试工程示例：

* liteflow-test-spring
* liteflow-test-springboot

分别对应了spring和springboot环境下的示例工程，可以运行`Runner`进行启动(会连带启动测试用例)

## 8.2 一个完整的案例

如果你想看一个实际的案例，加深对liteflow的理解。可以查看：

> [完整的带简单业务的案例](https://gitee.com/bryan31/liteflow-example)

这个案例为一个价格计算引擎，其目的是模拟了电商中对订单价格的计算，建议大家去pull下来，仔细阅读下，这个业务用到了liteflow的大部分特性

这个示例工程提供了一个简单的界面，供大家测试之用

![example-web](media/example-web.png)

# 九.未来版本计划

## 2.5版本

* 支持更多的表达式，重写表达式解析器
* 重新设计数据总线，解决数据槽热点问题
* 增加一种驱动模式：消息驱动的模式
* 对spring进行标签级支持
* 对组件侵入更低，支持标注级声明
* 增加监控的数据类型

## 2.6版本
* 提供一个简单的组件注册中心
* 有UI界面来查看监控数据
* 此版本的重点功能：能用UI界面来回放整个执行过程（精确到数据槽里每一个对象）
* 此版本的重点功能：界面式设计规则

## 3.0版本
主要是规则引擎的进化，制定规则文件。完善表达式引擎。

# 十.更新记录

## 1.3.1更新日志

优化大量潜在的问题，此版本为稳定版本，主要更新点如下：
* 增加条件节点功能
* 优化异常捕获的日志打印
* 支持自定义SLOT的特性
* 优化步骤打印，能够支持开闭区间的打印方式
* 增加了内部策略的调用方式
* 增加了追踪ID
* 优化了监控打印

## 2.0.1更新日志
更新点如下：
* 增加对zookeeper的支持
* 增加自定义配置源
* 优化监控的表现

## 2.1.3更新日志

更新点如下：

* 增加组件里结束整个流程的配置
* 修复一些可能导致内存变大的bug

## 2.2.2 更新日志

更新点如下：

* pom规划化，符合中央仓库的要求
* 包路径改成个人域名路径
* 修复当有多个流程配置时，条件节点失效的bug

## 2.3.0 更新日志

更新点如下：

* 重构核心部分代码
* 增加子流程显式调用

# 十一.联系作者

关注公众号回复`liteflow`即可加入讨论群

![offIical-wx](http://yomahub.com/images/offIical-wx.jpg)
