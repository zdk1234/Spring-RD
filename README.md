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
    * 			——> &colorFactoryBean
    * @return 
    */
    @Bean
    public ColorFactoryBean colorFactoryBean(){
        return new ColorFactoryBean();
    }
}
```
## 2.生命周期
