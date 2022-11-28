

## AOP常用注解：

- @Before 前置通知：目标方法之前执行
- @After 后置通知：目标方法之后执行（始终执行)
- @AfterReturning 返回后通知：执行方法结束前执行(异常不执行)
- @AfterThrowing 异常通知：出现异常时候执行
- @Around 环绕通知：环绕目标方法执行

## Spring的AOP顺序

### spring4下的aop测试案例

spring4的springboot版本为1.X version

```
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.After;
import org.aspectj.lang.annotation.AfterReturning;
import org.aspectj.lang.annotation.AfterThrowing;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class MyAspect {
    @Before("execution(public int com.lun.interview.service.CalcServiceImpl.*(..))")
    public void beforeNotify() {
        System.out.println("********@Before我是前置通知");
    }

    @After("execution(public int com.lun.interview.service.CalcServiceImpl.*(..))")
    public void afterNotify() {
        System.out.println("********@After我是后置通知");
    }

    @AfterReturning("execution(public int com.lun.interview.service.CalcServiceImpl.*(..))")
    public void afterReturningNotify() {
        System.out.println("********@AfterReturning我是返回后通知");
    }

    @AfterThrowing(" execution(public int com.lun.interview.service.CalcServiceImpl.*(..))")
    public void afterThrowingNotify() {
        System.out.println("********@AfterThrowing我是异常通知");
    }

    @Around(" execution(public int com.lun.interview.service.CalcServiceImpl.*(..))")
    public Object around(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
        Object retvalue = null;
        System.out.println("我是环绕通知之前AAA");
        retvalue = proceedingJoinPoint.proceed();
        System.out.println("我是环绕通知之后BBB");
        return retvalue ;
    }
}
```

```
import org.springframework.stereotype.Service;

@Service
public class CalcServiceImpl implements CalcService {

	@Override
	public int div(int x, int y) {
		int result = x / y;
		System.out.println("===>CalcServiceImpl被调用，计算结果为：" + result);
		return result;
	}
}
```

正常执行输出结果：

```
Spring Verision : 4.3.13.RELEASE, Sring Boot Version : 1.5.9.RELEASE.

我是环绕通知之前AAA
********@Before我是前置通知
===>CalcServiceImpl被调用，计算结果为：5
我是环绕通知之后BBB
********@After我是后置通知
********@AfterReturning我是返回后通知
```

如果接口中抛出异常，输出结果：

```
Spring Verision : 4.3.13.RELEASE, Sring Boot Version : 1.5.9.RELEASE.

我是环绕通知之前AAA
********@Before我是前置通知
********@After我是后置通知
********@AfterThrowing我是异常通知

java.lang.ArithmeticException: / by zero
	at com.lun.interview.service.CalcServiceImpl.div(CalcServiceImpl.java:10)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	...

```

AOP执行顺序：

- 正常情况下：@Before前置通知----->@After后置通知----->@AfterRunning正常返回
- 异常情况下：@Before前置通知----->@After后置通知----->@AfterThrowing方法异常

### spring5下的aop测试

spring5的springboot版本为2.X version

正常执行输出结果:

```
Spring Verision : 5.2.8.RELEASE, Sring Boot Version : 2.3.3.RELEASE.

我是环绕通知之前AAA
********@Before我是前置通知
===>CalcServiceImpl被调用，计算结果为：5
********@AfterReturning我是返回后通知
********@After我是后置通知
我是环绕通知之后BBB
```

异常执行输出结果:

```
Spring Verision : 5.2.8.RELEASE, Sring Boot Version : 2.3.3.RELEASE.

我是环绕通知之前AAA
********@Before我是前置通知
********@AfterThrowing我是异常通知
********@After我是后置通知

java.lang.ArithmeticException: / by zero
	at com.lun.interview.service.CalcServiceImpl.div(CalcServiceImpl.java:10)
	at com.lun.interview.service.CalcServiceImpl$$FastClassBySpringCGLIB$$355acbc4.invoke(<generated>)
	at org.springframework.cglib.proxy.MethodProxy.invoke(MethodProxy.java:218)
	at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.invokeJoinpoint(CglibAopProxy.java:771)

```

## 总结

### Spring4

正常执行输出结果：

```
Spring Verision : 4.3.13.RELEASE, Sring Boot Version : 1.5.9.RELEASE.

我是环绕通知之前AAA
********@Before我是前置通知
===>CalcServiceImpl被调用，计算结果为：5
我是环绕通知之后BBB
********@After我是后置通知
********@AfterReturning我是返回后通知
```

如果接口中抛出异常，输出结果：

```
Spring Verision : 4.3.13.RELEASE, Sring Boot Version : 1.5.9.RELEASE.

我是环绕通知之前AAA
********@Before我是前置通知
********@After我是后置通知
********@AfterThrowing我是异常通知

java.lang.ArithmeticException: / by zero
	at com.lun.interview.service.CalcServiceImpl.div(CalcServiceImpl.java:10)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	...

```

### Spring5

spring5的springboot版本为2.X version

正常执行输出结果:

```
Spring Verision : 5.2.8.RELEASE, Sring Boot Version : 2.3.3.RELEASE.

我是环绕通知之前AAA
********@Before我是前置通知
===>CalcServiceImpl被调用，计算结果为：5
********@AfterReturning我是返回后通知
********@After我是后置通知
我是环绕通知之后BBB
```

异常执行输出结果:

```
Spring Verision : 5.2.8.RELEASE, Sring Boot Version : 2.3.3.RELEASE.

我是环绕通知之前AAA
********@Before我是前置通知
********@AfterThrowing我是异常通知
********@After我是后置通知

java.lang.ArithmeticException: / by zero
	at com.lun.interview.service.CalcServiceImpl.div(CalcServiceImpl.java:10)
	at com.lun.interview.service.CalcServiceImpl$$FastClassBySpringCGLIB$$355acbc4.invoke(<generated>)
	at org.springframework.cglib.proxy.MethodProxy.invoke(MethodProxy.java:218)
	at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.invokeJoinpoint(CglibAopProxy.java:771)

```