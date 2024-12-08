
在平时使用到一些软件中，比如某宝或者某书，通过记录用户的行为来构建和分析用户的行为数据，同时也能更好优化产品设计和提升用户体验。比如在一个订单系统中，需要确定追踪用户的行为，比如：


* 登录/登出
* 浏览商品
* 加购商品
* 搜索商品关键字
* 下单


上述行为就需要使用到日志系统来存储或者记录数据，Java 有几种日志方案，简单介绍几种实现方案，以及需要注意的点。


# 日志添加需要注意问题


根据业务的不同，需要使用匹配到合适的日志方案。也需要注意几个问题：


* 不能影响原来的业务逻辑。
* 不能报错，即使报错，不能影响原有的业务代码。
* 对于耗时的日志代码，使用异步方法
* 侵入性低，尽量少改动原始代码


# Spring AOP


Spring AOP 通过切面编程实现不修改原有代码，而动态添加功能的能力。这种方式有以下几个好处：


* 侵入性低
* 可重用性强


本文使用基于注解的 AOP,首先定义一个切面注解 AopTest:



```
/**
 * 切面注解
 */
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface AopTest {
}

```

再创建通知 @Around ：



```
@Around("@annotation(com.test.annotation.AopTest)")
public Object annotationTest(ProceedingJoinPoint joinPoint) throws Throwable {
    log.info("执行前");
    Object result = joinPoint.proceed(); // 执行目标方法
    Object[] args = joinPoint.getArgs();
    log.info("执行后");
    return result;
}

```

一般都会在`执行后`添加日志即可，想要在那个方法或者接口加日志，只需要在方法上添加注解即可，比如在接口添加注解：



```
@GetMapping
@AopTest
public String first(String param) {
    log.info("执行first方法");
    return "result " + param;
}

```

请求接口后，就有如下的输出：



```
执行前
执行first方法
执行后

```

但是切面有一个问题，执行切面报错，方法也无法执行，就需要捕获异常，保证业务代码正常执行,改造一下上面的通知：



```
@Around("@annotation(com.test.annotation.AopTest)")
public Object annotationTest(ProceedingJoinPoint joinPoint) throws Throwable {log.info("执行前");
    log.info("执行前");
    Object result = joinPoint.proceed(); // 执行目标方法
    try {
        Object[] args = joinPoint.getArgs();
        // 添加日志
        log.info("执行后");
        int a = 1/0;
    } catch (Exception e) {
        log.error(e.getMessage());
    }
    return result;
}

```

改造后，即使切面报错，也不会影响业务代码的执行了。


AOP 是同步执行的，如果日志添加是一个比较耗时的操作，也会影响接口的响应速度，此时可以使用异步的方式，比如消息队列。


总结一下，Spring AOP 有以下几个优点和缺点。


* 优点：


	+ 侵入性低
	+ 可重用性强


缺点及解决方案：


1、切面报错可能会影响业务代码


* 问题：在切面的异常如果没有正确处理，可能会影响业务代码的正常执行。
* 解决方案：
	+ 捕获异常，确保业务代码正常执行
	+ 一般使用try catch 捕获异常，防止向上传播


2、同步执行会影响接口响应速度


* 问题：如果切面中存在耗时的操作，同步操作会导致接口的响应速度变慢。
* 解决方案：
	+ 通过消息队列将耗时操作异步执行
	+ 使用 @Async 将方法异步执行，将任务从主线程脱离出来，交给其他线程池执行


# 事件监听 \+ 异步


Spring 事件监听机制是一种`发布-订阅`模式，将事件的发布和监听解耦开来。通过这种机制，事件的发布者无需关注监听的逻辑，监听者也无需直接依赖发布者。


Spring 事件监听有三个部分组成：


1. 事件


自定义一个事件，继承 ApplicationEvent：



```
@Getter
@Setter
public class DemoEvent extends ApplicationEvent {

    private String name;

    public DemoEvent(Object source) {
        super(source);
    }
}


```

2. 事件监听


基于 @EventListener 注解监听事件：



```
@Component
@Slf4j
public class DemoEventListener {

  @EventListener
  public void handleEvent(DemoEvent event){
      log.info(event.getName());
      log.info("事件监听");

  }
}


```

3. 事件发布


使用 applicationEventPublisher.publishEvent 方法发布事件，



```

@Autowired
private ApplicationEventPublisher applicationEventPublisher;

@Override
public void publish() {
    // 执行业务代码
    log.info("执行登录，当前时间 {}",LocalTime.now());
    DemoEvent event = new DemoEvent(this);
    event.setName("hello");
    applicationEventPublisher.publishEvent(event);
    log.info("完成登录，当前时间 {}",LocalTime.now());
    log.info("执行 service");
}

```

配置好事件的发布和监听之后，在业务代码添加事件的发布，监听方法内添加日志的记录。


Spring 事件监听虽然解耦了发布和监听，只是解耦逻辑代码，两者还是`同步执行`，并且都是在同一个线程执行的，所以事件监听也无法解决添加日志报错，以及耗时的问题。


1. 验证监听方法是否同步


在事件监听添加延迟：



```
  @EventListener
  public void handleEvent(DemoEvent event){
      try {
          Thread.sleep(2000);
      } catch (InterruptedException e) {
          e.printStackTrace();
      }
      log.info(event.getName());
      log.info("事件监听");

  }

```

控制台输出：



```
执行登录，当前时间 16:52:30.799
hello
事件监听
完成登录，当前时间 16:52:33.799
执行 service

```


> 接口执行了 3 秒多，并且 publish 方法要等待监听方法执行完毕之后才能执行，说明`事件监听是同步的`。


2. 验证监听方法报错是否会影响主流程


在监听方法添加错误代码：



```
@EventListener
public void handleEvent(DemoEvent event){
    int a = 1/0;
    log.info(event.getName());
    log.info("事件监听");

}

```

控制台输出：



```
执行登录，当前时间 17:10:08.396
java.lang.ArithmeticException: / by zero

```


> 监听方法报错，接口也报错，业务代码无法执行，说明`监听方法报错会影响事件发布方法`。


解决报错的问题，使用异常捕获即可。而延迟的问题，就需要使用到`异步`的操作，`异步就是另启一个线程执行监听方法`,异步除了能解决延迟的问题，也顺便解决了报错的问题。


实现异步在监听方法上添加 @Async 异步注解，监听方法添加延迟和错误代码：



```
执行登录，当前时间 17:21:50.971
完成登录，当前时间 17:21:50.974
执行 service

```


> publish 方法既不会延迟，也不会因为监听报错影响执行，异步完美解决耗时和报错的问题


# 消息队列


海量日志场景，消息队列是一个很好的选择，它也是解耦了发布者和订阅者，如果订阅者开启了手动确认，消费者也需要使用 try catch 捕获异常信息，确保消息能正常消费。


# 总结


本文介绍几种日志记录的实现方案：


* `Spring AOP`:
	+ 优点： 侵入性低，代码可重复性强
	+ 问题以及解决方案：
		- 切面中可能会报错，报错会影响业务代码的正常执行，解决方法是使用 try catch 捕获异常。
		- 日志记录会影响业务代码执行效率，可以使用消息队列异步执行日志操作
* `事件监听 + 异步`：
	+ 优点： 解耦业务逻辑和日志记录，提升代码的内聚性。
	+ 缺点以及解决方案：
		- AOP 存在的问题，事件监听同样存在，报错和耗时都会影响业务代码。报错可以使用异常捕获，延迟问题可以使用异步方式解决，而异步另起线程也顺便解决了报错影响业务代码的问题。
* `消息队列`：
	+ 优点： 使用于高并发日记记录场景
	+ 问题： 增加系统的复杂性和稳定性，还需要考虑消息的丢失和重复消费问题。


 本博客参考[FlowerCloud机场订阅官网](https://hanlianfangzhi.com)。转载请注明出处！
