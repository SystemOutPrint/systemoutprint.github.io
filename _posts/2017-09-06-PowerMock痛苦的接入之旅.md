---
layout: post
title:  "PowerMock痛苦的接入之旅"
date:   2017-09-06
categories: JUnit
---

## 0x01 why？
单元测试的好处，此处不再叭叭了。<br><br>
一个项目有一套比较完整的测试用例，此处也不再叭叭了。<br><br>
在做写测试用例的时候，或多或少会遇到引用的某个方法返回值并不是我们预期的，可是为了测试用例的执行，我们需要在不修改代码的基础上让特定方法返回特定值。这就是为什么引入PowerMock和Mockito的原因。

## 0x02 PowerMock&Mockito
__Mockito__ 是一个针对 Java 的单元测试模拟框架，它与 EasyMock 和 jMock 很相似，都是为了简化单元测试过程中测试上下文 ( 或者称之为测试驱动函数以及桩函数 ) 的搭建而开发的工具。在有这些模拟框架之前，为了编写某一个函数的单元测试，程序员必须进行十分繁琐的初始化工作，以保证被测试函数中使用到的环境变量以及其他模块的接口能返回预期的值，有些时候为了单元测试的可行性，甚至需要牺牲被测代码本身的结构。单元测试模拟框架则极大的简化了单元测试的编写过程：在被测试代码需要调用某些接口的时候，直接模拟一个假的接口，并任意指定该接口的行为。这样就可以大大的提高单元测试的效率以及单元测试代码的可读性。<br>
__PowerMock__ 也是一个单元测试模拟框架，它是在其它单元测试模拟框架的基础上做出的扩展。通过提供定制的类加载器以及一些字节码篡改技巧的应用，PowerMock 现了对静态方法、构造方法、私有方法以及 Final 方法的模拟支持，对静态初始化过程的移除等强大的功能。因为 PowerMock 在扩展功能时完全采用和被扩展的框架相同的 API, 熟悉 PowerMock 所支持的模拟框架的开发者会发现 PowerMock 非常容易上手。PowerMock 的目的就是在当前已经被大家所熟悉的接口上通过添加极少的方法和注释来实现额外的功能，目前，PowerMock 仅支持 EasyMock 和 Mockito。

## 0x03 problem
在按照各种教程中的maven坐标，发现运行的时候会报如下异常：

		java.lang.VerifyError: Stack map does not match the one at exception handler 325
Exception Details:
  Location:
	limax/zdb/Zdb.testMeta(Llimax/xmlgen/Zdb;)Llimax/xmlgen/Zdb; @325: astore_3
  Reason:
	Type 'java/lang/Object' (current frame, locals[2]) is not assignable to null (stack map, locals[2])
  Current Frame:
	bci: @323
	flags: { }
	locals: { 'limax/zdb/Zdb', 'limax/xmlgen/Zdb', 'java/lang/Object', top, top, null, null, 'java/lang/Object' }
	stack: { 'java/lang/Throwable' }
  Stackmap Frame:
	bci: @325
	flags: { }
	locals: { 'limax/zdb/Zdb', 'limax/xmlgen/Zdb', null, null, top, 'java/lang/Object', top, 'java/lang/Object', 'java/lang/Object' }
	stack: { 'java/lang/Throwable' }
  Bytecode:
   
	......
   
	at java.lang.Class.getDeclaredConstructors0(Native Method)
	at java.lang.Class.privateGetDeclaredConstructors(Class.java:2671)
	at java.lang.Class.getDeclaredConstructors(Class.java:2020)
	at org.powermock.api.mockito.repackaged.ClassImposterizer.setConstructorsAccessible(ClassImposterizer.java:91)
	at org.powermock.api.mockito.repackaged.ClassImposterizer.imposterise(ClassImposterizer.java:77)
	at org.powermock.api.mockito.internal.mockcreation.DefaultMockCreator.createMethodInvocationControl(DefaultMockCreator.java:121)
	at org.powermock.api.mockito.internal.mockcreation.DefaultMockCreator.createMock(DefaultMockCreator.java:69)
	at org.powermock.api.mockito.internal.mockcreation.DefaultMockCreator.mock(DefaultMockCreator.java:46)
	at org.powermock.api.mockito.PowerMockito.mockStatic(PowerMockito.java:71)
	at game.gameserver.common.maintenance.TestMaintenanceFileListener.prepare(TestMaintenanceFileListener.java:41)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.junit.internal.runners.MethodRoadie.runBefores(MethodRoadie.java:133)
	at org.junit.internal.runners.MethodRoadie.runBeforesThenTestThenAfters(MethodRoadie.java:96)
	at org.powermock.modules.junit4.internal.impl.PowerMockJUnit44RunnerDelegateImpl$PowerMockJUnit44MethodRunner.executeTest(PowerMockJUnit44RunnerDelegateImpl.java:310)
	at org.powermock.modules.junit4.internal.impl.PowerMockJUnit47RunnerDelegateImpl$PowerMockJUnit47MethodRunner.executeTestInSuper(PowerMockJUnit47RunnerDelegateImpl.java:131)
	at org.powermock.modules.junit4.internal.impl.PowerMockJUnit47RunnerDelegateImpl$PowerMockJUnit47MethodRunner.access$100(PowerMockJUnit47RunnerDelegateImpl.java:59)
	at org.powermock.modules.junit4.internal.impl.PowerMockJUnit47RunnerDelegateImpl$PowerMockJUnit47MethodRunner$TestExecutorStatement.evaluate(PowerMockJUnit47RunnerDelegateImpl.java:147)
	at org.powermock.modules.junit4.internal.impl.PowerMockJUnit47RunnerDelegateImpl$PowerMockJUnit47MethodRunner.evaluateStatement(PowerMockJUnit47RunnerDelegateImpl.java:107)
	at org.powermock.modules.junit4.internal.impl.PowerMockJUnit47RunnerDelegateImpl$PowerMockJUnit47MethodRunner.executeTest(PowerMockJUnit47RunnerDelegateImpl.java:82)
	at org.powermock.modules.junit4.internal.impl.PowerMockJUnit44RunnerDelegateImpl$PowerMockJUnit44MethodRunner.runBeforesThenTestThenAfters(PowerMockJUnit44RunnerDelegateImpl.java:298)
	at org.junit.internal.runners.MethodRoadie.runTest(MethodRoadie.java:87)
	at org.junit.internal.runners.MethodRoadie.run(MethodRoadie.java:50)
	at org.powermock.modules.junit4.internal.impl.PowerMockJUnit44RunnerDelegateImpl.invokeTestMethod(PowerMockJUnit44RunnerDelegateImpl.java:218)
	at org.powermock.modules.junit4.internal.impl.PowerMockJUnit44RunnerDelegateImpl.runMethods(PowerMockJUnit44RunnerDelegateImpl.java:160)
	at org.powermock.modules.junit4.internal.impl.PowerMockJUnit44RunnerDelegateImpl$1.run(PowerMockJUnit44RunnerDelegateImpl.java:134)
	at org.junit.internal.runners.ClassRoadie.runUnprotected(ClassRoadie.java:34)
	at org.junit.internal.runners.ClassRoadie.runProtected(ClassRoadie.java:44)
	at org.powermock.modules.junit4.internal.impl.PowerMockJUnit44RunnerDelegateImpl.run(PowerMockJUnit44RunnerDelegateImpl.java:136)
	at org.powermock.modules.junit4.common.internal.impl.JUnit4TestSuiteChunkerImpl.run(JUnit4TestSuiteChunkerImpl.java:121)
	at org.powermock.modules.junit4.common.internal.impl.AbstractCommonPowerMockRunner.run(AbstractCommonPowerMockRunner.java:57)
	at org.powermock.modules.junit4.PowerMockRunner.run(PowerMockRunner.java:59)
	at org.eclipse.jdt.internal.junit4.runner.JUnit4TestReference.run(JUnit4TestReference.java:86)
	at org.eclipse.jdt.internal.junit.runner.TestExecution.run(TestExecution.java:38)
	at org.eclipse.jdt.internal.junit.runner.RemoteTestRunner.runTests(RemoteTestRunner.java:459)
	at org.eclipse.jdt.internal.junit.runner.RemoteTestRunner.runTests(RemoteTestRunner.java:678)
	at org.eclipse.jdt.internal.junit.runner.RemoteTestRunner.run(RemoteTestRunner.java:382)
	at org.eclipse.jdt.internal.junit.runner.RemoteTestRunner.main(RemoteTestRunner.java:192)
	
## 0x04 reason
　　PowerMock中为支持对构造函数的测试，借助于Javassist实现对字节码的操作。但是从Java 6开始引入的Stack Map Frames特性与Javassist不兼容。<br>
　　在Java 6中该Stack Map Frames还是可选的。但是到了Java 7，该Stackmap Frames已经是默认使用的，所以不兼容问题导致了该异常。<br>
　　在java 7中可以使用-XX:-UseSplitVerifier来解决此问题，但是java 8中该选项已经失效。但是可以使用-Xverify:none关闭字节码验证来达到相同的效果。
	
## 0x05 solution
maven依赖配置：
```xml
<dependency>
	<groupId>junit</groupId>
	<artifactId>junit</artifactId>
	<scope>test</scope>
</dependency>
<dependency>
	<groupId>org.powermock</groupId>
	<artifactId>powermock-api-mockito</artifactId>
	<version>1.7.1</version>
</dependency>
<dependency>
	<groupId>org.powermock</groupId>
	<artifactId>powermock-module-junit4</artifactId>
	<version>1.7.1</version>
	<scope>test</scope>
</dependency>
<dependency>
	<groupId>org.mockito</groupId>
	<artifactId>mockito-core</artifactId>
	<version>1.10.19</version>
</dependency>
```

maven插件配置
```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-surefire-plugin</artifactId>
	<version>2.19.1</version>
	<configuration>
		<argLine>-Xverify:none</argLine>
	</configuration>
</plugin>
```

禁用字节码验证即可。


## 0x06 demo
```java
@PowerMockIgnore( {"javax.management.*"}) 
@RunWith(PowerMockRunner.class)
@PrepareForTest({table.Bag.class, XBean.class, Logs.class, Onlines.class, ConfigManager.class})
public class BagAddTest {

	private final static long TEST_ROLEID = 1024L;
	private final static int TEST_ITEM_ID = 10086;
	
	@Before
	public void prepare() {
		PowerMockito.mockStatic(table.Bag.class);   
		PowerMockito.when(table.Bag.update(TEST_ROLEID)).thenReturn(null);
		PowerMockito.when(table.Bag.insert(TEST_ROLEID)).thenReturn(new xbean.Bag());
		
		PowerMockito.suppress(MemberModifier.method(XBean.class, "verifyStandaloneOrLockHeld"));
		PowerMockito.suppress(MemberModifier.method(Logs.class, "logObject"));
		PowerMockito.suppress(MemberModifier.method(Onlines.class, "sendWhileCommit", new Class[]{Long.class, Protocol.class}));
		
		SCommonConfig mockCommonCfg = new SCommonConfig();
		mockCommonCfg.param1 = 10;
		Map<Integer, SCommonConfig> mockCommonCfgs = new HashMap<>();
		mockCommonCfgs.put(1, mockCommonCfg);
		
		SItemServerConfig itemServerCfg = new SItemServerConfig();
		itemServerCfg.isStack = 2;
		Map<Integer, SItemServerConfig> mockItemCfgs = new HashMap<>();
		mockItemCfgs.put(TEST_ITEM_ID, itemServerCfg);
		
		Map<String, Object> beanName2Cfgs = new HashMap<>();
		beanName2Cfgs.put(SCommonConfig.class.getName(), mockCommonCfgs);
		beanName2Cfgs.put(SItemServerConfig.class.getName(), mockItemCfgs);
		
		ConfigManager cfgMgr = ConfigManager.getInstance();
		try {
			MemberModifier.field(ConfigManager.class, "beanName2Cfgs").set(cfgMgr, beanName2Cfgs);
		} catch (IllegalArgumentException | IllegalAccessException e) {
			e.printStackTrace();
		};
	}
	
}

```
* @RunWith指定了Junit的执行类为PowerMockRunner。
* @PrepareForTest指定下面要对哪些类进行Mock。
* 如果对静态方法进行mock，则需要先使用PowerMockito.mockStatic。
* PowerMockito.when(A).thenReturn(a)的意思就是当调用A方法的时候，返回a。
* PowerMockito.suppress(A)代表禁用A方法。