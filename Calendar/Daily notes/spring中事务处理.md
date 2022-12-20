---
title: spring中事务处理
date created: 2022-12-19
date modified: 2022-12-20
---

## 事务提交成功后在执行操作

### 使用事件监听机制中@TransactionalEventListener

#### 使用方法

```java
@Slf4j
@Service
public class HelloServiceImpl implements HelloService {

    @Autowired
    private JdbcTemplate jdbcTemplate;
    @Autowired
    private ApplicationEventPublisher applicationEventPublisher;

    @Transactional
    @Override
    public Object hello(Integer id) {
        // 向数据库插入一条记录
        String sql = "insert into user (id,name,age) values (" + id + ",'fsx',21)";
        jdbcTemplate.update(sql);

        // 发布一个自定义的事件~~~
        applicationEventPublisher.publishEvent(new MyAfterTransactionEvent("我是和事务相关的事件，请事务提交后执行我~~~", id));
        return "service hello";
    }

    @Slf4j
    @Component
    private static class MyTransactionListener {
        @Autowired
        private JdbcTemplate jdbcTemplate;

        @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
        private void onHelloEvent(HelloServiceImpl.MyAfterTransactionEvent event) {
            Object source = event.getSource();
            Integer id = event.getId();

            String query = "select count(1) from user where id = " + id;
            Integer count = jdbcTemplate.queryForObject(query, Integer.class);
            
            // 可以看到 这里的count是1  它肯定是在上面事务提交之后才会执行的
            log.info(source + ":" + count.toString()); //我是和事务相关的事件，请事务提交后执行我~~~:1
        }
    }


    // 定一个事件，继承自ApplicationEvent 
    private static class MyAfterTransactionEvent extends ApplicationEvent {

        private Integer id;

        public MyAfterTransactionEvent(Object source, Integer id) {
            super(source);
            this.id = id;
        }

        public Integer getId() {
            return id;
        }
    }
}
————————————————
版权声明：本文为CSDN博主「YourBatman」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/f641385712/article/details/91897175
```

#### 实现原理

1. 通过TransactionalEventListenerFactory 来，把每个标注有`@TransactionalEventListener`注解的方法最终都包装成一个`ApplicationListenerMethodTransactionalAdapter`，它是一个`ApplicationListener`，最终注册进事件发射器的容器里面

```java
public class TransactionalEventListenerFactory implements EventListenerFactory, Ordered {
	private int order = 50; // 执行时机还是比较早的~~~（默认的工厂是最低优先级）
	
	// 显然这个工厂只会生成标注有此注解的handler~~~
	@Override
	public boolean supportsMethod(Method method) {
		return AnnotatedElementUtils.hasAnnotation(method, TransactionalEventListener.class);
	}

	// 这里使用的是ApplicationListenerMethodTransactionalAdapter，而非ApplicationListenerMethodAdapter
	// 虽然ApplicationListenerMethodTransactionalAdapter是它的子类
	@Override
	public ApplicationListener<?> createApplicationListener(String beanName, Class<?> type, Method method) {
		return new ApplicationListenerMethodTransactionalAdapter(beanName, type, method);
	}
}
————————————————
版权声明：本文为CSDN博主「YourBatman」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/f641385712/article/details/91897175
```

2. 在适配类中，根据当前是否有事务，调用不同实现类

```java
// @since 4.2
class ApplicationListenerMethodTransactionalAdapter extends ApplicationListenerMethodAdapter {

	private final TransactionalEventListener annotation;

	// 构造函数
	public ApplicationListenerMethodTransactionalAdapter(String beanName, Class<?> targetClass, Method method) {
		// 这一步的初始化交给父类，做了很多事情   强烈建议看看上面推荐的事件/监听的博文
		super(beanName, targetClass, method);

		// 自己个性化的：和事务相关
		TransactionalEventListener ann = AnnotatedElementUtils.findMergedAnnotation(method, TransactionalEventListener.class);
		if (ann == null) {
			throw new IllegalStateException("No TransactionalEventListener annotation found on method: " + method);
		}
		this.annotation = ann;
	}

	@Override
	public void onApplicationEvent(ApplicationEvent event) {
		// 若**存在事务**：毫无疑问 就注册一个同步器进去~~
		if (TransactionSynchronizationManager.isSynchronizationActive()) {
			TransactionSynchronization transactionSynchronization = createTransactionSynchronization(event);
			TransactionSynchronizationManager.registerSynchronization(transactionSynchronization);
		}
		// 若fallbackExecution=true，那就是表示即使没有事务  也会执行handler
		else if (this.annotation.fallbackExecution()) {
			if (this.annotation.phase() == TransactionPhase.AFTER_ROLLBACK && logger.isWarnEnabled()) {
				logger.warn("Processing " + event + " as a fallback execution on AFTER_ROLLBACK phase");
			}
			processEvent(event);
		}
		else {
			// No transactional event execution at all
			// 若没有事务，输出一个debug信息，表示这个监听器没有执行~~~~
			if (logger.isDebugEnabled()) {
				logger.debug("No transaction is active - skipping " + event);
			}
		}
	}

	// TransactionSynchronizationEventAdapter是一个内部类，它是一个TransactionSynchronization同步器
	// 此类实现也比较简单，它的order由listener.getOrder();来决定
	private TransactionSynchronization createTransactionSynchronization(ApplicationEvent event) {
		return new TransactionSynchronizationEventAdapter(this, event, this.annotation.phase());
	}


	private static class TransactionSynchronizationEventAdapter extends TransactionSynchronizationAdapter {

		private final ApplicationListenerMethodAdapter listener;
		private final ApplicationEvent event;
		private final TransactionPhase phase;

		public TransactionSynchronizationEventAdapter(ApplicationListenerMethodAdapter listener,
				ApplicationEvent event, TransactionPhase phase) {
			this.listener = listener;
			this.event = event;
			this.phase = phase;
		}

		// 它的order又监听器本身来决定  
		@Override
		public int getOrder() {
			return this.listener.getOrder();
		}

		// 最终都是委托给了listenner来真正的执行处理  来执行最终处理逻辑（也就是解析classes、condtion、执行方法体等等）
		@Override
		public void beforeCommit(boolean readOnly) {
			if (this.phase == TransactionPhase.BEFORE_COMMIT) {
				processEvent();
			}
		}

		// 此处结合status和phase   判断是否应该执行~~~~
		// 此处小技巧：我们发现TransactionPhase.AFTER_COMMIT也是放在了此处执行的，只是它结合了status进行判断而已~~~
		@Override
		public void afterCompletion(int status) {
			if (this.phase == TransactionPhase.AFTER_COMMIT && status == STATUS_COMMITTED) {
				processEvent();
			} else if (this.phase == TransactionPhase.AFTER_ROLLBACK && status == STATUS_ROLLED_BACK) {
				processEvent();
			} else if (this.phase == TransactionPhase.AFTER_COMPLETION) {
				processEvent();
			}
		}

		protected void processEvent() {
			this.listener.processEvent(this.event);
		}
	}
}
————————————————
版权声明：本文为CSDN博主「YourBatman」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/f641385712/article/details/91897175
```

#### 注意点

1. 需要排序时，可以使用 @Order注解
2. EventLister写法，注意不能在代理类下，某些可能会没扫描到的情况。因为获取的是生成类是否满足条件

### 使用 TransactionSynchronizationManager，即可以参考ApplicationListenerMethodTransactionalAdapter 的代码实现

```java
TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronizationAdapter() {
    @Override
    public void afterCommit() {
        System.out.println("事务提交后的操作。。。");
    }
});
```

## 参考文章

+ [Spring事务监听机制---使用@TransactionalEventListener处理数据库事务提交成功后再执行操作（附：Spring4.2新特性讲解）【享学Spring】_YourBatman的博客-CSDN博客_spring 如何感知这个事务已经提交](https://fangshixiang.blog.csdn.net/article/details/91897175?spm=1001.2101.3001.6661.1&utm_medium=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1-91897175-blog-126592411.pc_relevant_multi_platform_whitelistv4&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1-91897175-blog-126592411.pc_relevant_multi_platform_whitelistv4&utm_relevant_index=1)
+ [Spring是如何保证同一事务获取同一个Connection的？使用Spring的事务同步机制解决：数据库刚插入的记录却查询不到的问题【享学Spring】_YourBatman的博客-CSDN博客](https://blog.csdn.net/f641385712/article/details/91538445)
