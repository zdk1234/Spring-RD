# Spring注解驱动开发
## 1.组件注册
### 1.1 @Configuration&@Bean给容器中注册组件
```java
@Configuration
public class MainConfig {
    @Bean
    public Employee getEmployee(){
        return new Employee();
    }
}
```
>通过`@Configuration`和`@Bean` 给spring ioc容器注入组件

### 1.2 @ComponentScan-自动扫描组件&指定扫描规则
```java
@Configuration
@ComponentScans(
        value = {
                @ComponentScan(value = "com.zdk", includeFilters = {
                        @Filter(type = FilterType.ANNOTATION, classes = {Controller.class}),
                        @Filter(type = FilterType.ASSIGNABLE_TYPE, classes = {BookService.class}),
                        @Filter(type = FilterType.CUSTOM, classes = {MyTypeFilter.class})
                }, useDefaultFilters = false)
        }
)
public class MainConfig {
    //TODO
}
```
>`@ComponentScans`指定需要扫描的包  
  
`excludeFilters = Filter[]`指定扫描的时候按照什么规则排除那些组件  
`includeFilters = Filter[]`指定扫描的时候只需要包含哪些组件  
`FilterType.ANNOTATION`按照注解  
`FilterType.ASSIGNABLE_TYPE`按照给定的类型  
`FilterType.ASPECTJ`使用ASPECTJ表达式  
`FilterType.REGEX`使用正则指定  
`FilterType.CUSTOM`使用自定义规则

### 1.3 @Scope-设置组件作用域
>`@Scope`标注在组件容器上 并赋值达到对组件作用域的限定  

`prototype`多实例的：ioc容器启动并不会去调用方法创建对象放在容器中。 每次获取的时候才会调用方法创建对象；  
`singleton`单实例的（默认值）：ioc容器启动会调用方法创建对象放到ioc容器中。以后每次获取就是直接从容器（map.get()）中拿  
`request`同一次请求创建一个实例  
`session`同一个session创建一个实例  

### 1.4 @Lazy-bean懒加载
`@Lazy`标注的组件在容器启动的时候不会创建对象，第一次获取Bean时创建对象并初始化

### 1.5 @Conditional-按照条件注册bean
>`@Conditional`按照一定的条件进行判断，满足条件给容器中注册bean,通过下面的例子`@Conditional(LinuxCondition.class)`就能实现
```java
    /**
     * 判断是否linux系统
     */
    public class LinuxCondition implements Condition {
    	/**
    	 * ConditionContext：判断条件能使用的上下文（环境）
    	 * AnnotatedTypeMetadata：注释信息
    	 */
    	@Override
    	public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
    		//1、能获取到ioc使用的beanfactory
    		ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
    		//2、获取类加载器
    		ClassLoader classLoader = context.getClassLoader();
    		//3、获取当前环境信息
    		Environment environment = context.getEnvironment();
    		//4、获取到bean定义的注册类
    		BeanDefinitionRegistry registry = context.getRegistry();
    		String property = environment.getProperty("os.name");
    		//可以判断容器中的bean注册情况，也可以给容器中注册bean
    		boolean definition = registry.containsBeanDefinition("person");
    		return property.contains("linux");
    	}
    }
```
### 1.6 @Import-给容器中快速导入一个组件
1.`@Import`(要导入到容器中的组件)；容器中就会自动注册这个组件，id默认是全类名  
2.`ImportSelector`返回需要导入的组件的全类名数组；
```java
//自定义逻辑返回需要导入的组件
public class MyImportSelector implements ImportSelector {
	//返回值，就是到导入到容器中的组件全类名
	//AnnotationMetadata:当前标注@Import注解的类的所有注解信息
	@Override
	public String[] selectImports(AnnotationMetadata importingClassMetadata) {
		//importingClassMetadata
		//方法不要返回null值
		return new String[]{"com.atguigu.bean.Blue","com.atguigu.bean.Yellow"};
	}
}
```
3.`ImportBeanDefinitionRegistrar`手动注册bean到容器中
```java
public class MyImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {
    /**
     * AnnotationMetadata：当前类的注解信息
     * BeanDefinitionRegistry:BeanDefinition注册类；
     * 把所有需要添加到容器中的bean；调用BeanDefinitionRegistry.registerBeanDefinition手工注册进来
     */
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        boolean definition = registry.containsBeanDefinition("com.atguigu.bean.Red");
        boolean definition2 = registry.containsBeanDefinition("com.atguigu.bean.Blue");
        if (definition && definition2) {
            //指定Bean定义信息
            RootBeanDefinition beanDefinition = new RootBeanDefinition(RainBow.class);
            //注册一个Bean，指定bean名
            String beanName = "rainBow";
            boolean isInUse = registry.isBeanNameInUse(beanName);
            if (!isInUse) {
                registry.registerBeanDefinition(beanName, beanDefinition);
            }
        }
    }
}
```
### 1.7 使用FactoryBean注册组件
```java
//创建一个Spring定义的FactoryBean
public class ColorFactoryBean implements FactoryBean<Color> {
	//返回一个Color对象，这个对象会添加到容器中
	@Override
	public Color getObject() throws Exception {
		System.out.println("ColorFactoryBean...getObject...");
		return new Color();
	}
	@Override
	public Class<?> getObjectType() {
		return Color.class;
	}
	//是单例？
	//true：这个bean是单实例，在容器中保存一份
	//false：多实例，每次获取都会创建一个新的bean；
	@Override
	public boolean isSingleton() {
		return false;
	}
}

@Configuration
public class MainConfig {
    /**
    * 注意 默认获取到的是工厂bean调用getObject创建的对象
    * 要获取工厂Bean本身，我们需要给id前面加一个&
    * @return 
    */
    @Bean
    public ColorFactoryBean colorFactoryBean(){
        return new ColorFactoryBean();
    }
}
```
### 1.8 组件注册总结
1、包扫描+组件标注注解（@Controller/@Service/@Repository/@Component）[自己写的类]  
2、@Bean[导入的第三方包里面的组件]  
3、@Import[快速给容器中导入一个组件]  
>@Import(要导入到容器中的组件)；容器中就会自动注册这个组件，id默认是全类名  
>ImportSelector:返回需要导入的组件的全类名数组；  
>ImportBeanDefinitionRegistrar:手动注册bean到容器中  

4、使用Spring提供的 FactoryBean（工厂Bean）;  
>默认获取到的是工厂bean调用getObject创建的对象  
>要获取工厂Bean本身，我们需要给id前面加一个& &colorFactoryBean 
 
## 2.生命周期
### 2.1 @Bean指定初始化和销毁方法
>指定`@Bean`中的`initMethod`和`destroyMethod`属性完成
```java
@ComponentScan("com.atguigu.bean")
@Configuration
public class MainConfigOfLifeCycle {
	//指定Car类中的init和detory方法，作为实例初始化之后调用的，和销毁之后调用的方法
	@Bean(initMethod="init",destroyMethod="detory")
	public Car car(){
		return new Car();
	}
}
```
### 2.2 InitializingBean和DisposableBean的前置后置方法
>容器组件实现`InitializingBean`对`afterPropertiesSet()`进行覆盖，实现`DisposableBean`对`destroy()`方法进行覆盖
```java
@Component
public class Cat implements InitializingBean, DisposableBean {
    public Cat() {
        System.out.println("cat constructor...");
    }
    @Override
    public void destroy() throws Exception {
        System.out.println("cat...destroy...");
    }
    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("cat...afterPropertiesSet...");
    }
}
```
### 2.3 @PostConstruct&@PreDestroy的前置后置方法
>容器组件在方法上增加`@PostConstruct`:对象创建并赋值之后调用;`@PreDestroy`:容器移除对象之前调用
```java
@Component
public class Dog {
    public Dog() {
        System.out.println("dog constructor...");
    }
    //对象创建并赋值之后调用
    @PostConstruct
    public void init() {
        System.out.println("Dog....@PostConstruct...");
    }
    //容器移除对象之前
    @PreDestroy
    public void detory() {
        System.out.println("Dog....@PreDestroy...");
    }
}
```
### 2.4 BeanPostProcessor后置处理器
>容器组件实现`BeanPostProcessor`并覆盖`postProcessBeforeInitialization`和`postProcessAfterInitialization`
```java
@Component
public class MyBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("postProcessBeforeInitialization..." + beanName + "=>" + bean);
        return bean;
    }
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("postProcessAfterInitialization..." + beanName + "=>" + bean);
        return bean;
    }
}
```
### 2.5 生命周期总结
>以下总结参考[Spring Bean的生命周期中各方法的执行顺序](https://www.jianshu.com/p/80d4fa132747)
#### 2.5.1 常用的设定方式
1.通过实现`InitializingBean`接口来定制初始化之后的操作方法；  
2.通过实现`DisposableBean`接口来定制销毁之前的操作方法；  
3.通过元素的`init-method`属性指定初始化之后调用的操作方法；  
4.通过元素的`destroy-method`属性指定销毁之前调用的操作方法；  
5.在指定方法上加上`@PostConstruct`注解来制定该方法是在初始化之后调用；  
6.在指定方法上加上`@PreDestroy`注解来制定该方法是在销毁之前调用；  
7.实现`BeanNameAware`接口设置Bean的ID或者Name;  
8.实现`BeanFactoryAware`接口设置BeanFactory;  
9.实现`ApplicationContextAware`接口设置ApplicationContext；  
10.注册实现了`BeanPostProcessor`的Bean后处理器，通过`postProcessBeforeInitialization`和`postProcessAfterInitialization`方法对初始化的Bean进行自定义处理;
#### 2.5.2 初始化过程中各方法的执行顺序
1.调用构造器`Bean.constructor`进行实例化;  
2.调用`Setter`方法，设置属性值;  
3.调用`BeanNameAware.setBeanName`设置Bean的ID或者Name;  
4.调用`BeanFactoryAware.setBeanFactory`设置BeanFactory;  
5.调用`ApplicationContextAware.setApplicationContext`置ApplicationContext；  
6.调用`BeanPostProcessor`的预先初始化方法，如下：  
BeanPostProcessor1.postProcessBeforeInitialization  
BeanPostProcessor2.postProcessBeforeInitialization  
……  
7.调用由`@PostConstruct`注解的方法；  
8.调用`InitializingBean.afterPropertiesSet`；  
9.调用`Bean.init-mehod`初始化方法；  
10.调用`BeanPostProcessor`的后初始化方法，如下：  
BeanPostProcessor1.postProcessAfterInitialization  
BeanPostProcessor2.postProcessAfterInitialization  
……  
#### 2.5.3 容器关闭时，Bean销毁过程中各方法的执行顺序
1.调用由`@PreDestroy`注解的方法  
2.调用`DisposableBean`的`destroy()`;  
3.调用定制的`destroy-method`方法;  

## 3 属性赋值
### 3.1 @Value赋值 & @PropertySource加载外部配置文件
>`@Value`注解中可以写基本数值，可以写`SpEl`表达式，也可以写`${}`取出运行环境变量的值（需要通过在配置类`@PropertySource`中配置属性文件）
```java
public class Person {
    //使用@Value赋值；
    //1、基本数值
    //2、可以写SpEL； #{}
    //3、可以写${}；取出配置文件【properties】中的值（在运行环境变量里面的值）
    @Value("张三")
    private String name;
    @Value("#{20-2}")
    private Integer age;
    @Value("${person.nickName}")
    private String nickName;
    //getset...
}
//使用@PropertySource读取外部配置文件中的k/v保存到运行的环境变量中;加载完外部的配置文件以后使用${}取出配置文件的值
@PropertySource(value={"classpath:/person.properties"})
@Configuration
public class MainConfigOfPropertyValues {
	@Bean
	public Person person(){
		return new Person();
	}
}
```
## 4 自动装配
### 4.1 @Autowired & @Qualifier & @Primary
@Autowired：自动注入  
1）默认优先按照类型去容器中找对应的组件:`applicationContext.getBean(BookDao.class)`;找到就赋值  
2）如果找到多个相同类型的组件，再将属性的名称作为组件的id去容器中查找`applicationContext.getBean("bookDao")`  
3）`@Qualifier("bookDao")`：使用@Qualifier指定需要装配的组件的id，而不是使用属性名  
4）自动装配默认一定要将属性赋值好，没有就会报错； 可以使用`@Autowired(required=false)`;  
5）`@Primary`：让Spring进行自动装配的时候，默认使用首选的bean；也可以继续使用@Qualifier指定需要装配的bean的名字  
### 4.2 @Resource&@Inject
关于自动注入Spring还支持使用`@Resource`(JSR250)和`@Inject`(JSR330)[java规范的注解]  
`@Resource`:可以和@Autowired一样实现自动装配功能；默认是按照组件名称进行装配的；没有能支持@Primary功能没有支持@Autowired(reqiured=false);  
`@Inject`:需要导入javax.inject的包，和Autowired的功能一样。没有required=false的功能；
`@Autowired`:Spring定义的； `@Resource`、`@Inject`都是java规范
### 4.3 自动装配方式
>装配方式分为三种:方法注入、构造器注入、接口注入

@Autowired:构造器，参数，方法，属性；都是从容器中获取参数组件的值  
1）[标注在方法位置]：@Bean+方法参数；参数从容器中获取;默认不写@Autowired效果是一样的；都能自动装配  
2）[标在构造器上]：如果组件只有一个有参构造器，这个有参构造器的@Autowired可以省略，参数位置的组件还是可以自动从容器中获取  
3）放在参数位置(接口注入?)  
### 4.4 Aware注入Spring底层组件&原理
自定义组件想要使用Spring容器底层的一些组件（ApplicationContext，BeanFactory，xxx）；自定义组件实现xxxAware；在创建对象的时候，会调用接口规定的方法注入相关组件；Aware；把Spring底层一些组件注入到自定义的Bean中；
xxxAware：功能使用xxxProcessor；ApplicationContextAware==》ApplicationContextAwareProcessor；
### 4.5 @Profile环境搭建
`Profile`:Spring为我们提供的可以根据当前环境，动态的激活和切换一系列组件的功能；
>指定组件在哪个环境的情况下才能被注册到容器中，不指定，任何环境下都能注册这个组件

1)加了环境标识的bean，只有这个环境被激活的时候才能注册到容器中。默认是default环境  
2)写在配置类上，只有是指定的环境的时候，整个配置类里面的所有配置才能开始生效  
3)没有标注环境标识的bean在任何环境下都是加载的；  
