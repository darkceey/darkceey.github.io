---
layout:     post
title:      乐观锁注解和实现
subtitle:   在SpringBoot中自定义乐观锁
date:       2019-04-28
author:     darkceey
header-img: img/about_desk.jpg
catalog: true
tags:
    - 乐观锁
---

>随便点评：岁月悠悠，不经意间，翻看之前的博客，一转眼，一年多就可以飞逝而去，时不我待啊。最近一直忙着项目，抽不开身，平时一些好的想法或思路不了了之，深以为憾，最近基于实际业务场景，自定义实现了
>乐观锁的注解及监听功能，感觉还可以，特记录在此，往能帮到大家。
>以后要开启我的博客记录之旅了~~ 
> 记2019年4月28日23:51:28


# 乐观锁介绍 #
  总是假设最好的情况，每次去拿数据的时候都认为别人不会修改，所以不会上锁，但是在更新的时候会判断一下在此期间别人有没有去更新这个数据，可以使用版本号机制和CAS算法实现。数据版本,为数据增加的一个版本标识。当读取数据时，将版本标识的值一同读出，数据每更新一次，同时对版本标识进行更新。当我们提交更新的时候，判断数据库表对应记录的当前版本信息与第一次取出来的版本标识进行比对，如果数据库表当前版本号与第一次取出来的版本标识值相等，则予以更新，否则认为是过期数据。乐观锁适用于多读的应用类型，这样可以提高吞吐量，像数据库提供的类似于write_condition机制，其实都是提供的乐观锁。在Java中java.util.concurrent.atomic包下面的原子变量类就是使用了乐观锁的一种实现方式CAS实现的。

  __乐观锁常见的两种实现方式__ 

   * 1.版本号机制

   一般是在数据表中加上一个数据版本号version字段，表示数据被修改的次数，当数据被修改时，version值会加一。当线程A要更新数据值时，在读取数据的同时也会读取version值，在提交更新时，若刚才读取到的version值为当前数据库中的version值相等时才更新，否则重试更新操作，可根据自定义次数实现重试次数，直到更新成功。

   * 2.版本号机制

   即compare and swap（比较与交换），是一种有名的无锁算法。无锁编程，即不使用锁的情况下实现多线程之间的变量同步，也就是在没有线程被阻塞的情况下实现变量的同步，所以也叫非阻塞同步（Non-blocking Synchronization）。CAS算法涉及到三个操作数

		需要读写的内存值
		进行比较的值 A
		拟写入的新值 B

   当且仅当 V 的值等于 A时，CAS通过原子方式用新值B来更新V的值，否则不会执行任何操作（比较和替换是一个原子操作）。一般情况下是一个自旋操作，即不断的重试。 	

# 乐观锁具体实现

 乐观锁实现可分为四步

#### 1.自定义乐观锁注解加载类EnableOptimisticLock

		@Target({ElementType.TYPE})
		Retention(RetentionPolicy.RUNTIME)
		@Documented
		@Inherited
		@Import({OptimisticLockConfiguration.class})
		public @interface EnableOptimisticLock {
		
		}

  加载OptimisticLockConfiguration类，并实例化注解拦截器optimisticLockAspect
			
		public class OptimisticLockConfiguration {
		
		    private static final Logger logger = LoggerFactory.getLogger(OptimisticLockConfiguration.class);
		
		    @Bean
		    public OptimisticLockAspect optimisticLockAspect(){
		        logger.info("创建OptimisticLockAspect");
		        return new OptimisticLockAspect();
		    }
		
		} 	

### 2. 乐观锁自定义注解

	@Inherited
	@Retention(RetentionPolicy.RUNTIME)
	@Target({ElementType.METHOD})
	public @interface OptimisticLock {

    /**
     * 捕获异常
     * @return
     */
    Class<? extends Exception>[] catchType() default {OptimisticException.class};

    /**
     * 异常重试次数
     * @return
     */
    int retry() default 3;

}		


#### 3.AOP拦截乐观锁注解

	@Aspect
	@Order(-1000) //order值越大，优先级越小
	public class OptimisticLockAspect {
	
    private final static Logger logger = LoggerFactory.getLogger(OptimisticLockAspect.class);


    /**
	 * 第一种方式 ：直接在切点里拦截
     * 拦截AbstractDAO的update方法
     * 用于抛出乐观锁冲突时的异常
     * 因拦截乐观锁，乐观锁只在更新时生效，拦截至具体的方法下
     */
    @Pointcut("execution(int com.kafu.common.dao.DAO.update(com.kafu.common.pojo.po.AbstractPO))")
    public void daoPointcut(){}

    @Around("daoPointcut()")
    public Object doDAOAround(final ProceedingJoinPoint thisJoinPoint) throws Throwable {
        Object[] args = thisJoinPoint.getArgs();
        int count = (int)thisJoinPoint.proceed();
        if((args[0] instanceof Version) && count<=0){
			//根据业务场景，设置各种异常编码，以便支持国际化
            throw new OptimisticException(MessageSourceUtil.getMessage("error.optimistic_lock"));
        }
        return count;
    }


    /**
 	 * 第二种：自定义实现@OptimisticLock注解
     * 拦截任何添加了@OptimisticLock注解的Service方法
     * 捕获乐观锁冲突异常，并重试
     */
    @Pointcut("execution(@com.kafu.common.optimistic.OptimisticLock * *(..))")
    public void servicePointcut(){}

    @Around("servicePointcut()")
    public Object doServiceAround(final ProceedingJoinPoint thisJoinPoint) throws Throwable {
        Signature signature = thisJoinPoint.getSignature();
        MethodSignature methodSignature = (MethodSignature)signature;
        final Method targetMethod = methodSignature.getMethod();
        OptimisticLock optimisticLock = targetMethod.getAnnotation(OptimisticLock.class);
        if(optimisticLock ==null){
            throw new RuntimeException("optimistic lock aop error");
        }

        Class<? extends Exception>[] catchTypes = optimisticLock.catchType();
        if(catchTypes==null || catchTypes.length==0){
            throw new RuntimeException("optimistic lock aop error");
        }
        int retry = optimisticLock.retry();
        Object object = tryServiceProceed(thisJoinPoint, catchTypes, retry);
        return object;
    }

	/**
	/**
     *  尝试正常更新，如更新时抛出异常，则现场休眠100ms后进行重新尝试，默认重试3次
     */
    private Object tryServiceProceed(ProceedingJoinPoint thisJoinPoint, Class<? extends Exception>[] catchTypes, int retry) throws Throwable {
        Object object = null;
        try {
            object = thisJoinPoint.proceed();
        } catch (Throwable throwable) {
            if(retry>0) {
                for (Class<? extends Exception> catchType : catchTypes) {
                    if (catchType.isInstance(throwable)) {
                        try {
                            // 睡100毫秒再试
                            Thread.sleep(100);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                        logger.warn("乐观锁重试,retry="+retry+",method="+thisJoinPoint.getSignature().getName());
                        return tryServiceProceed(thisJoinPoint,catchTypes,--retry);
                    }
                }
            }
            throw throwable;
        }
        return object;
    }
}

### 4. 乐观锁注解加载类绑定至Springboot启动项

	@SpringBootApplication
	@EnableOptimisticLock
	public class KaFuApp extends SpringBootServletInitializer {

    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(KaFuApp.class);
        app.run(args);
    }

    /**
     * 兼容tomcat部署模式
     * @param application
     * @return
     */
    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
        return application.sources(KaFuApp.class);
    	}
	}


# 总结
 在高并发且要求数据安全唯一的情况下，基本都要考虑乐观锁来应对这种业务场景。可通过java提供的自定义注解实现乐观锁的功能。
这里如果所有的DAO层的更新接口均继承父类接口DAO。则不用注解，可直接在切点里拦截。如果有些update没有继承公共的DAO层，则
在更新方法上添加@OptimisticLock即可。两种方式。都很方便。如此使开发时不用考虑具体实现，直接使用@optimisticLock即可。optimisticLock提供可重试次数参数，默认为3次，如果重试3次。不管成功或者失败，结束本次操作。















 