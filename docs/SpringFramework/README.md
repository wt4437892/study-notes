### 1.9.5 使用范式作为自动装配限定符

除了@Qualifier注解之外，还能使用Java泛型作为一种隐形限定形式。例如，假设你有以下配置：

```java
@Configuration
public class MyConfiguration {

    @Bean
    public StringStore stringStore() {
        return new StringStore();
    }

    @Bean
    public IntegerStore integerStore() {
        return new IntegerStore();
    }
}
```

假设前面的beans实现了一个泛型接口，（即，Store\<String\> 和 Store\<Integer\> ），你能@Autowire注入

Store接口，并且泛型作为限定。如下例所示：

```java
// <String> qualifier, injects the stringStore bean
@Autowired
private Store<String> s1; 

// <Integer> qualifier, injects the integerStore bean
@Autowired
private Store<Integer> s2; 
```

当注入List、Map和array时泛型限定也适用，以下示例注入泛型列表：

```java
// Inject all Store beans as long as they have an <Integer> generic
// Store<String> beans will not appear in this list
@Autowired
private List<Store<Integer>> s;
```

### 1.9.6 使用CustomAutowireConfigurer

CustomAutowireConfigurer是**BeanFactoryPostProcessor**，允许你注册你自己定制限定符注释类型。即使它们没有使用Spring的@Qualifier注释进行注释。以下示例显示了如何使用CustomAutowireConfigurer：

```xml
<bean id="customAutowireConfigurer" class="org.springframework.beans.factory.annotation.CustomAutowireConfigurer">
  <property name="customQualifierTypes">
    <set>
      <value>com.example.module.chapter1_9_6.annotation.FoxQualifier</value>
      <value>com.example.module.chapter1_9_6.annotation.DeerQualifier</value>
      <value>com.example.module.chapter1_9_6.annotation.PigQualifier</value>
    </set>
  </property>
</bean>
```

AutowireCandidateResolver通过以下方式确定autowire候选对象：

- 每个bean定义的autowire-candidate
- \<beans/\>元素上提供的任何default-autowire-candidates
- 是否存在@Qualifier注释以及在CustomAutowireConfigurer中注册的任何自定义注释

当多个bean符合自动装配候选时，“primary”的决定如下：如果候选中恰好有一个bean定义的primary属性设置为true，则选择它。

### 1.9.7 注入@Resource

Spring还通过在字段或bean属性设置方法上使用JSR-250@Resource注释（javax.annotation.Resource）来支持注入。这是JavaEE中的一种常见模式：例如，在JSF管理的bean和JAX-WS端点中。Spring也支持Spring管理对象的这种模式。

@Resource具有名称属性。默认情况下，Spring将该值解释为要注入的bean名称。换句话说，它遵循名称语义，如以下示例所示：

```java
public class SimpleMovieLister {

    private MovieFinder movieFinder;

    @Resource(name="myMovieFinder") 
    public void setMovieFinder(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }
}
```

如果未明确指定名称，则默认名称从字段名或setter方法派生。对于字段，它采用字段名。对于setter方法，它采用bean属性名。以下示例将名为movieFinder的bean注入其setter方法：

```java
public class SimpleMovieLister {

    private MovieFinder movieFinder;

    @Resource
    public void setMovieFinder(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }
}
```

> CommonAnnotationBeanPostProcessor知道的ApplicationContext将注释提供的名称解析为bean名称。如果显式配置Spring的SimpleJndiBeanFactory，则可以通过JNDI解析这些名称。但是，我们建议您依赖默认行为，并使用Spring的JNDI查找功能来保持间接级别。

在没有指定显式名称的@Resource 用法的独有情况下，与@Autowired类似，@Resource查找一个主类型匹配项，而不是特定的命名bean，并解析众所周知的可解析依赖项：BeanFactory、ApplicationContext、ResourceLoader、ApplicationEventPublisher和MessageSource接口。

因此，在下面的示例中，customerPreferenceDao字段首先查找名为“customerPreferenceDao”的bean，然后返回到customerPreferenceDao类型的主类型匹配：

```java
public class MovieRecommender {

    @Resource
    private CustomerPreferenceDao customerPreferenceDao;

    @Resource
    private ApplicationContext context; 

    public MovieRecommender() {
    }

    // ...
}
```

> 根据已知的可解析依赖项类型注入context字段：
> ApplicationContext。

### 1.9.8 使用@Value

@Value通常用于注入外部化属性：

```java
@Component
public class MovieRecommender {

    private final String catalog;

    public MovieRecommender(@Value("${catalog.name}") String catalog) {
        this.catalog = catalog;
    }
}
```

使用以下配置：

```java
@Configuration
@PropertySource("classpath:application.properties")
public class AppConfig {}
```

以及以下application.properties文件

```properties
catalog.name=MovieCatalog
```

在这种情况下，catalog参数和字段将等于MovieCatalog值。

### 1.9.9 使用@PostConstruct 和 @PreDestory

`CommonAnnotationBeanPostProcessor`不仅识别@Resource注释，还识别JSR-250生命周期注释：`javax.annotation.PostConstruct `和`javax.annotation.PreDestroy`. Spring2.5中引入的对这些注释的支持为初始化回调和销毁回调中描述的生命周期回调机制提供了一种替代方法。如果`CommonAnnotationBeanPostProcessor`在`Spring ApplicationContext`中注册，则携带这些注释之一的方法将在生命周期中与相应的Spring生命周期接口方法或显式声明的回调方法在相同的点上调用。在以下示例中，缓存在初始化时预填充，在销毁时清除：

```java
public class CachingMovieLister {

    @PostConstruct
    public void populateMovieCache() {
        // populates the movie cache upon initialization...
    }

    @PreDestroy
    public void clearMovieCache() {
        // clears the movie cache upon destruction...
    }
}
```

 >与@Resource一样，@PostConstruct和@PreDestroy注释类型也是JDK 6到8标准Java库的一部分。但是，整个javax。注释包与JDK 9中的核心Java模块分离，并最终在JDK 11中删除。如果需要，可以使用`javax.annotation-api`工件现在需要通过Maven Central获得，只需像任何其他库一样添加到应用程序的类路径。

## 1.10 类路径扫描与托管构件

本章中的大多数示例都使用XML指定在Spring容器中生成每个BeanDefinition的配置元数据。上一节（基于注释的容器配置）演示了如何通过源代码级注释提供大量配置元数据。然而，即使在这些示例中，“基本”bean定义也在XML文件中显式定义，而注释只驱动依赖项注入。本节描述通过扫描类路径隐式检测候选组件的选项。候选组件是与筛选条件匹配的类，并且具有在容器中注册的对应bean定义。这样就不需要使用XML来执行bean注册。相反，您可以使用注释（例如，@Component）、AspectJ类型表达式或您自己的自定义筛选条件来选择哪些类在容器中注册了bean定义。

>从Spring3.0开始，SpringJavaConfig项目提供的许多特性都是核心Spring框架的一部分。这允许您使用Java定义bean，而不是使用传统的XML文件。查看@Configuration、@Bean、@Import和@DependsOn注释，了解如何使用这些新功能的示例。

### 1.10.1 @Component和更多的原型注释

@Repository注释是实现存储库的角色或原型（也称为数据访问对象或DAO）的任何类的标记。此标记的用途之一是异常的自动转换.

Spring提供了进一步的原型注释：@Component、@Service和@Controller。@Component是任何Spring托管组件的通用原型,@Repository、@Service和@Controller是@Component的专门化，用于更具体的用例（分别在持久层、服务层和表示层）。因此，您可以使用@
Component注释组件类，但是，通过使用@Repository、@Service或@Controller注释它们，您的类更适合通过工具处理或与方面关联。例如，这些原型注释是切入点的理想目标@Repository、@Service和@Controller也可以在Spring框架的未来版本中提供额外的语义。因此，如果您要在服务层使用@Component或@Service之间进行选择，@Service显然是更好的选择。类似地，如前所述，@Repository已经被支持作为持久层中自动异常转换的标记。

### 1.10.2 使用元注释与组合注释

Spring提供的许多注释可以在您自己的代码中用作元注释。元注释是可以应用于其他注释的注释。例如，前面提到的@Service注释是用@Component进行元注释的，如下例所示：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component (1)
public @interface Service {

    // ...
}
```

(1)  @Component使@Service的处理方式与@Component相同。

您还可以组合元注释来创建“组合注释”。例如，SpringMVC中的@RestController注释由@Controller和@ResponseBy组成。

此外，组合注释可以选择从元注释重新声明属性，以允许自定义。当您只想公开元注释属性的子集时，这可能特别有用。例如，Spring的@SessionScope注释将作用域名称硬编码到会话，但仍然允许定制proxyMode。下表显示了SessionScope注释的定义：

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Scope(WebApplicationContext.SCOPE_SESSION)
public @interface SessionScope {

    /**
     * Alias for {@link Scope#proxyMode}.
     * <p>Defaults to {@link ScopedProxyMode#TARGET_CLASS}.
     */
    @AliasFor(annotation = Scope.class)
    ScopedProxyMode proxyMode() default ScopedProxyMode.TARGET_CLASS;

}
```

然后，您可以使用@SessionScope而不声明proxyMode，如下所示：

```java
@Service
@SessionScope
public class SessionScopedService {
    // ...
}
```

您还可以覆盖proxyMode的值，如下例所示：

```java
@Service
@SessionScope(proxyMode = ScopedProxyMode.INTERFACES)
public class SessionScopedUserService implements UserService {
    // ...
}
```

### 1.10.3 自动检测类并注册Bean定义

Spring可以自动检测模式化类，并向ApplicationContext注册相应的BeanDefinition实例。例如，以下两类可用于此类自动检测：

```java
@Service
public class SimpleMovieLister {

    private MovieFinder movieFinder;

    public SimpleMovieLister(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }
}
```

```java
@Repository
public class JpaMovieFinder implements MovieFinder {
    // implementation elided for clarity
}
```

要自动检测这些类并注册相应的bean，需要将@ComponentScan添加到@Configuration类中，其中basePackages属性是这两个类的公共父包。（或者，您可以指定一个逗号、分号或空格分隔的列表，其中包含每个类的父包。）

```java
@Configuration
@ComponentScan(basePackages = "org.example")
public class AppConfig  {
    // ...
}
```

>为简洁起见，前面的示例可以使用注释的value属性（即@ComponentScan（“org.example”））。

以下替代方案使用XML：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        https://www.springframework.org/schema/context/spring-context.xsd">

    <context:component-scan base-package="org.example"/>
</beans>
```

> 使用\<context:component-scan\>隐式启用\<context:annotation-config\>的功能。使用\<context:component-scan\>时，通常不需要包含\<context:annotation-config\>元素。

此外，在使用组件扫描元素时，`AutoWiredNotationBeanPostProcessor`和`CommonAnnotationBeanPostProcessor`都隐式包含在内。这意味着这两个组件被自动检测并连接在一起 — 所有这些都没有XML中提供的任何bean配置元数据。

> 通过包含值为false的`annotation-config`属性，可以禁用`AutoWiredNotationBeanPostProcessor`和`CommonAnnotationBeanPostProcessor`的注册。

### 1.10.4 使用过滤器自定义扫描

默认情况下，使用@Component、@Repository、@Service、@Controller、@Configuration注释的类或使用@Component注释的自定义注释是唯一检测到的候选组件。但是，您可以通过应用自定义过滤器来修改和扩展此行为。将它们添加为@ComponentScan注释的includeFilters或excludeFilters属性（或作为XML配置中<context:component scan>元素的<context:include filter/>或<context:exclude filter/>子元素）。每个过滤器元素都需要类型和表达式属性。下表介绍了过滤选项：

| 过滤器类型           | 案例                       | 描述                                                     |
| -------------------- | -------------------------- | -------------------------------------------------------- |
| annotation (default) | org.example.SomeAnnotation | 在目标组件的类型级别上存在或元存在的注释。               |
| assignable           | org.example.SomeClass      | 目标组件可分配给（扩展或实现）的类（或接口）。           |
| assignable           | org.example..*Service+     | 要由目标组件匹配的AspectJ类型表达式。                    |
| regex                | org\.example\.Default.*    | 由目标组件的类名匹配的正则表达式。                       |
| custom               | org.example.MyTypeFilter   | org.springframework.core.type.TypeFilter接口的自定义实现 |

### 1.10.6  命名自动检测到的组件

当组件作为扫描过程的一部分被自动检测到时，其bean名称由该扫描仪已知的BeanNameGenerator策略生成。默认情况下，任何包含名称值的Spring原型注释（@Component、@Repository、@Service和@Controller）都会将该名称提供给相应的bean定义。

如果这样的注释不包含名称值或任何其他检测到的组件（如自定义过滤器发现的组件），默认bean名称生成器将返回首字母非大写的非限定类名。例如，如果检测到以下组件类，则名称将为myMovieLister和movieFinderImpl：

```java
@Service("myMovieLister")
public class SimpleMovieLister {
    // ...
}
```

```java
@Repository
public class MovieFinderImpl implements MovieFinder {
    // ...
}
```

如果不想依赖默认的bean命名策略，可以提供自定义bean命名策略。首先，实现BeanNameGenerator接口，并确保包含默认的无参数构造函数。然后，在配置扫描器时提供完全限定的类名，如下例注释和bean定义所示。

> 如果由于多个自动检测组件具有相同的非限定类名（即，具有相同名称但在不同包中的类）而遇到命名冲突，则可能需要为生成的bean名称配置默认为完全限定类名的BeanNameGenerator。从SpringFramework 5.2.3,位于`org.springframework.context.annotation`包中 `FullyQualifiedAnnotationBeanAMEGenerator`可用于此类目的。

```java
@Configuration
@ComponentScan(basePackages = "org.example", nameGenerator = MyNameGenerator.class)
public class AppConfig {
    // ...
}
```

```xml
<beans>
    <context:component-scan base-package="org.example"
        name-generator="org.example.MyNameGenerator" />
</beans>
```

作为一般规则，考虑在任何其他组件可能对其进行明确引用时，用注释指定名称。另一方面，只要容器负责布线，自动生成的名称就足够了。

### 1.10.7 为自动检测到的组件提供作用域

与Spring管理的组件一样，自动检测组件的默认和最常见范围是singleton(单例模式)。但是，有时您需要一个可以由@scope注释指定的不同范围。您可以在注释中提供作用域的名称，如下例所示：

```java
@Scope("prototype")
@Repository
public class MovieFinderImpl implements MovieFinder {
    // ...
}
```

> @Scope注释仅在具体bean类（用于带注释的组件）或工厂方法（用于@bean方法）上进行内省。与XMLBean定义相比，没有bean定义继承的概念，类级别的继承层次结构与元数据目的无关。

有关特定于web的作用域（如Spring上下文中的“request”或“session”）的详细信息，请参阅Request、Session、Application和WebSocket作用域。与这些作用域的预构建注释一样，您也可以使用Spring的元注释方法编写自己的作用域注释：例如，使用@Scope（“prototype”）注释的自定义注释元，可能还声明了自定义作用域代理模式。

> 要为范围解析提供自定义策略，而不是依赖基于注释的方法，可以实现ScopeMetadataResolver接口。确保包含默认的无参数构造函数。然后，您可以在配置扫描器时提供完全限定的类名，如下注释和bean定义示例所示：

```java
@Configuration
@ComponentScan(basePackages = "org.example", scopeResolver = MyScopeResolver.class)
public class AppConfig {
    // ...
}
```

```xml
<beans>
    <context:component-scan base-package="org.example" scope-resolver="org.example.MyScopeResolver"/>
</beans>
```

当使用某些非单例作用域时，可能需要为作用域对象生成代理。推理在作用域bean中描述为依赖项。为此，component-scan元素上有一个作用域代理属性。这三个可能的值是：no、interfaces和targetClass。例如，以下配置会产生标准JDK动态代理：

```java
@Configuration
@ComponentScan(basePackages = "org.example", scopedProxy = ScopedProxyMode.INTERFACES)
public class AppConfig {
    // ...
}
```

```xml
<beans>
    <context:component-scan base-package="org.example" scoped-proxy="interfaces"/>
</beans>
```

### 1.10.8 提供带有注释的限定符元数据

@Qualifier注释将在基于注释的带限定符的自动连接微调中讨论。该部分中的示例演示如何使用@Qualifier注释和自定义限定符注释来在解析autowire候选对象时提供细粒度控制。因为这些示例基于XMLBean定义，所以通过使用XML中bean元素的限定符或元子元素，在候选bean定义上提供限定符元数据。当依靠类路径扫描自动检测组件时，您可以为限定符元数据提供候选类上的类型级注释。以下三个示例演示了此技术：

```java
@Component
@Qualifier("Action")
public class ActionMovieCatalog implements MovieCatalog {
    // ...
}
```

```java
@Component
@Genre("Action")
public class ActionMovieCatalog implements MovieCatalog {
    // ...
}
```

```java
@Component
@Offline
public class CachingMovieCatalog implements MovieCatalog {
    // ...
}
```

> 与大多数基于注释的替代方案一样，请记住注释元数据绑定到类定义本身，而XML的使用允许相同类型的多个bean在其限定符元数据中提供变化，因为元数据是按实例而不是按类提供的。

## 1.11 使用JSR330标准注释

从Spring3.0开始，Spring支持JSR-330标准注释（依赖项注入）。这些注释的扫描方式与Spring注释相同。要使用它们，您需要在类路径中有相关的JAR。

```xml
<dependency>
    <groupId>javax.inject</groupId>
    <artifactId>javax.inject</artifactId>
    <version>1</version>
</dependency>
```

### 1.11.1 使用@Inject与@Named依赖注入

你能使用@javax.inject.Inject代替@Autowired如下：

```java
import javax.inject.Inject;

public class SimpleMovieLister {

    private MovieFinder movieFinder;

    @Inject
    public void setMovieFinder(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }

    public void listMovies() {
        this.movieFinder.findMovies(...);
        // ...
    }
}
```

与@Autowired一样，您可以在字段级、方法级和构造函数参数级使用@Inject。此外，您可以将注入点声明为提供者，允许按需访问范围较短的bean，或者通过提供者延迟访问其他bean.get（）调用。以下示例提供了前一示例的变体：

```java
import javax.inject.Inject;
import javax.inject.Provider;

public class SimpleMovieLister {

    private Provider<MovieFinder> movieFinder;

    @Inject
    public void setMovieFinder(Provider<MovieFinder> movieFinder) {
        this.movieFinder = movieFinder;
    }

    public void listMovies() {
        this.movieFinder.get().findMovies(...);
        // ...
    }
}
```

如果要为应注入的依赖项使用限定名称，则应使用@Named注释，如下例所示：

```java
import javax.inject.Inject;
import javax.inject.Named;

public class SimpleMovieLister {

    private MovieFinder movieFinder;

    @Inject
    public void setMovieFinder(@Named("main") MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }

    // ...
}
```

与@Autowired一样，@Inject也可以与java.util.Optional一起使用或@Nullable。这在这里更适用，因为@Inject没有必需的属性。以下两个示例演示了如何使用@Inject和@Nullable：

```java
public class SimpleMovieLister {

    @Inject
    public void setMovieFinder(Optional<MovieFinder> movieFinder) {
        // ...
    }
}
```

```java
public class SimpleMovieLister {

    @Inject
    public void setMovieFinder(@Nullable MovieFinder movieFinder) {
        // ...
    }
}
```

### 1.11.2 @Named和@ManagedBean：@Component的标准等效注解

你能使用`@javax.inject.Named`或`javax.annotation.ManagedBean`替代`@component`，如下例所示：

```java
import javax.inject.Inject;
import javax.inject.Named;

@Named("movieListener")  // @ManagedBean("movieListener") could be used as well
public class SimpleMovieLister {

    private MovieFinder movieFinder;

    @Inject
    public void setMovieFinder(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }

    // ...
}
```

使用@Component而不指定组件名称是非常常见的@Named可以以类似的方式使用，如下例所示：

```java
import javax.inject.Inject;
import javax.inject.Named;

@Named
public class SimpleMovieLister {

    private MovieFinder movieFinder;

    @Inject
    public void setMovieFinder(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }

    // ...
}
```

使用@Named或@ManagedBean时，可以使用与使用Spring注释完全相同的组件扫描方式，如下例所示：

```java
@Configuration
@ComponentScan(basePackages = "org.example")
public class AppConfig  {
    // ...
}
```

> 与@Component不同，JSR-330@Named和JSR-250 ManagedBean注释是不可组合的。您应该使用Spring的原型模型来构建自定义组件注释。

### 1.11.3 JSR-330标准注释的限制

使用标准注释时，您应该知道某些重要功能不可用，如下表所示：

| Spring              | javax.inject.*       | javax.inject 限制/备注                                       |
| ------------------- | -------------------- | ------------------------------------------------------------ |
| @Autowired          | @Inject              | @Inject没有“required”属性。可以与Java8的Optional一起使用。   |
| @Component          | @Named/ @ManagedBean | JSR-330不提供可组合模型，只提供一种识别命名组件的方法        |
| @Scope("singleton") | @Singletion          | JSR-330的默认范围类似于Spring的原型。然而，为了使它与Spring的一般默认值保持一致，Spring容器中声明的JSR-330 bean默认为单例。为了使用singleton以外的范围，您应该使用Spring的@scope注释。javax.inject还提供了@Scope注释。然而，这一条仅用于创建您自己的注释。 |
| @Qualifier          | @Qualifier / @Named  | javax.inject.Qualfier只是用于构建自定义限定符的元注释。具体的字符串限定符（比如Spring的@Qualifier和一个值）可以通过javax.inject.Named进行关联注射 |
| @Value              | -                    | 无等价                                                       |
| @Required           | -                    | 无等价                                                       |
| @Lazy               | -                    | 无等价                                                       |
| ObjectFactory       | Provider             | javax.inject.Provider是Spring的ObjectFactory的直接替代品，只是具有较短的get（）方法名。它还可以与Spring的@Autowired或非注释构造函数和setter方法结合使用 |

## 1.12 基于Java的容器配置

### 1.12.1 基础概念:@Bean与@Configuration

Spring新的Java配置支持的核心构件是@configuration注释类和@Bean注释方法。

@Bean注释用于指示方法实例化、配置和初始化要由SpringIoC容器管理的新对象。对于熟悉Spring的\<beans/\>XML配置的人来说，@Bean注释与\<Bean/\>元素起着相同的作用。您可以对任何Spring@组件使用@Bean注释的方法。但是，它们最常用于@Configuration beans

用@Configuration注释一个类表明它的主要用途是作为bean定义的源。此外，@Configuration类允许通过调用同一类中的其他@bean方法来定义bean之间的依赖关系。最简单的@Configuration类如下所示：

```java
@Configuration
public class AppConfig {

    @Bean
    public MyService myService() {
        return new MyServiceImpl();
    }
}
```

前面的AppConfig类相当于下面的Spring\<beans/\>XML：

```xml
<beans>
    <bean id="myService" class="com.acme.services.MyServiceImpl"/>
</beans>
```

@Bean和@Configuration注释将在以下部分中进行深入讨论。但是，首先，我们将介绍使用基于Java的配置创建spring容器的各种方法。

### 1.12.2 使用AnnotationConfigApplicationContext实例化Spring容器

以下各节记录了Spring在Spring 3.0中引入的注释ConfigApplicationContext。这个多功能的ApplicationContext实现不仅能够接受@Configuration类作为输入，还能够接受普通的@Component类和用JSR-330元数据注释的类。

当@Configuration类作为输入提供时，@Configuration类本身被注册为bean定义，类中所有声明的@bean方法也被注册为bean定义。

当提供@Component和JSR-330类时，它们被注册为bean定义，并且假设必要时在这些类中使用诸如@Autowired或@Inject之类的依赖注入元数据。

#### 简单构造

与实例化ClassPathXmlApplicationContext时使用SpringXML文件作为输入的方式大致相同，在实例化AnnotationConfigApplicationContext时可以使用@Configuration类作为输入。这允许完全不使用XML的Spring容器，如下例所示：

```java
public static void main(String[] args) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
    MyService myService = ctx.getBean(MyService.class);
    myService.doStuff();
}
```

如前所述，AnnotationConfigApplicationContext不仅限于使用@Configuration类。任何@Component或JSR-330注释类都可以作为构造函数的输入提供，如下例所示：

```java
public static void main(String[] args) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(MyServiceImpl.class, Dependency1.class, Dependency2.class);
    MyService myService = ctx.getBean(MyService.class);
    myService.doStuff();
}
```

前面的示例假设MyServiceImpl、Dependency1和Dependency2使用Spring依赖项注入注释，例如@Autowired

#### 通过使用register(Classs<？>...)以编程方式构建容器)

您可以使用无参数构造函数实例化AnnotationConfigApplicationContext，然后使用register（）方法对其进行配置。当以编程方式构建AnnotationConfigApplicationContext时，此方法特别有用。以下示例显示了如何执行此操作：

```java
public static void main(String[] args) {
    AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
    ctx.register(AppConfig.class, OtherConfig.class);
    ctx.register(AdditionalConfig.class);
    ctx.refresh();
    MyService myService = ctx.getBean(MyService.class);
    myService.doStuff();
}
```

#### 使用扫描启用组件scan(String…)

```java
@Configuration
@ComponentScan(basePackages = "com.acme") (1)
public class AppConfig  {
    // ...
}
```

> (1) 此注释启用组件扫描。

在前面的示例中，扫描com.acme包以查找任何带@Component注释的类，这些类在容器中注册为SpringBean定义。AnnotationConfigApplicationContext公开scan（String…) 方法以允许相同的组件扫描功能，如下例所示：

```java
public static void main(String[] args) {
    AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
    ctx.scan("com.acme");
    ctx.refresh();
    MyService myService = ctx.getBean(MyService.class);
}
```

### 1.12.3 使用@Bean注解

#### 声明一个Bean

要声明bean，可以使用@bean注释对方法进行注释。您可以使用此方法在指定为方法返回值的类型的ApplicationContext中注册bean定义。默认情况下，bean名称与方法名称相同。以下示例显示@Bean方法声明：

```java
@Configuration
public class AppConfig {

    @Bean
    public TransferServiceImpl transferService() {
        return new TransferServiceImpl();
    }
}
```

前面的配置与下面的Spring XML完全等效：

```xml
<beans>
    <bean id="transferService" class="com.acme.TransferServiceImpl"/>
</beans>
```

这两个声明都使名为transferService的bean在ApplicationContext中可用，绑定到TransferServiceImpl类型的对象实例，如下图所示：

```text
transferService -> com.acme.TransferServiceImpl
```

您还可以使用默认方法来定义bean。这允许通过在默认方法上实现带有bean定义的接口来组合bean配置。

```java
public interface BaseConfig {

    @Bean
    default TransferServiceImpl transferService() {
        return new TransferServiceImpl();
    }
}

@Configuration
public class AppConfig implements BaseConfig {

}
```

您还可以使用接口（或基类）返回类型声明@Bean方法，如下例所示：

```java
@Configuration
public class AppConfig {

    @Bean
    public TransferService transferService() {
        return new TransferServiceImpl();
    }
}
```

但是，这将高级类型预测的可见性限制为指定的接口类型（TransferService）。然后，容器只在实例化受影响的单例bean之后才知道完整类型（TransferServiceImpl）。非惰性单例bean根据其声明顺序进行实例化，因此您可能会看到不同的类型匹配结果，这取决于另一个组件尝试通过非声明类型进行匹配的时间（例如@Autowired TransferServiceImpl，它仅在transferService bean实例化后才解析）。

> 如果您始终通过声明的服务接口引用您的类型，那么@Bean返回类型可以安全地加入该设计决策。但是，对于实现多个接口的组件或可能由其实现类型引用的组件，更安全的做法是声明可能的最具体的返回类型（至少按照引用bean的注入点的要求）。

#### Bean依赖关系

@Bean注释的方法可以有任意数量的参数来描述构建该Bean所需的依赖关系。例如，如果TransferService需要AccountRepository，我们可以使用方法参数具体化该依赖关系，如下例所示：

```java
@Configuration
public class AppConfig {

    @Bean
    public TransferService transferService(AccountRepository accountRepository) {
        return new TransferServiceImpl(accountRepository);
    }
}
```

解析机制与基于构造函数的依赖项注入几乎相同。

#### 接收生命周期回调

使用@Bean注释定义的任何类都支持常规生命周期回调，并且可以使用JSR-250中的@PostConstruct和@PreDestroy注释。有关更多详细信息，请参见JSR-250注释。
也完全支持常规的Spring生命周期回调。如果bean实现了InitializingBean、DisposableBean或Lifecycle，则容器将调用它们各自的方法。
还完全支持标准的*aware接口集（如BeanFactoryAware、BeanNameware、MessageSourceAware、ApplicationContextAware等）。
@Bean注释支持指定任意初始化和销毁回调方法，就像Spring XML的init method和Bean元素上的destroy method属性一样，如下例所示：

```java
public class BeanOne {

    public void init() {
        // initialization logic
    }
}

public class BeanTwo {

    public void cleanup() {
        // destruction logic
    }
}

@Configuration
public class AppConfig {

    @Bean(initMethod = "init")
    public BeanOne beanOne() {
        return new BeanOne();
    }

    @Bean(destroyMethod = "cleanup")
    public BeanTwo beanTwo() {
        return new BeanTwo();
    }
}
```

> 默认情况下，使用Java配置定义的具有公共close或shundown方法的bean将自动使用销毁回调登记。如果您有一个public close或shutdown方法，并且不希望在容器关闭时调用它，那么可以将@Bea(destroymethod=“”)添加到Bean定义中以禁用默认（推断）模式。
>
> 默认情况下，您可能希望对使用JNDI获取的资源执行此操作，因为其生命周期是在应用程序之外管理的。特别是，请确保始终为数据源执行此操作，因为众所周知，在JavaEE应用程序服务器上会出现问题。
>
> 以下示例显示如何防止数据源的自动销毁回调：
>
> ```java
> @Bean(destroyMethod="")
> public DataSource dataSource() throws NamingException {
>     return (DataSource) jndiTemplate.lookup("MyDS");
> }
> ```
>
> 另外，对于@Bean方法，通常使用编程JNDI查找，通过使用Spring的JNDemplate或JndiLocatorDelegate帮助程序或直接使用JNDI InitialContext，而不是JndiObjectFactoryBean变体（这将迫使您将返回类型声明为FactoryBean类型，而不是实际的目标类型，这使得在其他打算在此处引用所提供资源的@Bean方法中进行交叉引用调用变得更加困难）。

对于上述示例中的BeanOne，在构造期间直接调用init（）方法同样有效，如下例所示：

```java
@Configuration
public class AppConfig {

    @Bean
    public BeanOne beanOne() {
        BeanOne beanOne = new BeanOne();
        beanOne.init();
        return beanOne;
    }

    // ...
}
```

> 当您直接在Java中工作时，您可以对对象执行任何您喜欢的操作，而不必总是依赖于容器生命周期。

### 1.12.5 基于java的组合配置

##### 注入对导入的@Bean定义的依赖项

前面的例子虽然有效，但过于简单。在大多数实际场景中，bean在配置类之间相互依赖。使用XML时，这不是问题，因为不涉及编译器，您可以声明ref=“someBean”并信任Spring在容器初始化期间解决它。当使用@Configuration类时，Java编译器会对配置模型进行约束，因为对其他bean的引用必须是有效的Java语法。

幸运的是，解决这个问题很简单。正如我们已经讨论过的，@Bean方法可以有任意数量的参数来描述Bean依赖关系。考虑下面几个更真实的场景，其中有几个@Configuration类，每个依赖于其他声明的bean：

```java
@Configuration
public class ServiceConfig {

    @Bean
    public TransferService transferService(AccountRepository accountRepository) {
        return new TransferServiceImpl(accountRepository);
    }
}

@Configuration
public class RepositoryConfig {

    @Bean
    public AccountRepository accountRepository(DataSource dataSource) {
        return new JdbcAccountRepository(dataSource);
    }
}

@Configuration
@Import({ServiceConfig.class, RepositoryConfig.class})
public class SystemTestConfig {

    @Bean
    public DataSource dataSource() {
        // return new DataSource
    }
}

public static void main(String[] args) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(SystemTestConfig.class);
    // everything wires up across configuration classes...
    TransferService transferService = ctx.getBean(TransferService.class);
    transferService.transfer(100.00, "A123", "C456");
}
```

还有另一种方法可以达到同样的效果。请记住，@Configuration类最终只是容器中的另一个bean：这意味着它们可以利用@Autowired和@Value injection以及与任何其他bean相同的其他特性。

> 确保以这种方式注入的依赖项仅为最简单的类型@Configuration类在上下文初始化过程中很早就被处理，强制以这种方式注入依赖项可能会导致意外的早期初始化。只要有可能，就采用基于参数的注入，如前一示例所示。
> 另外，通过@Bean对BeanPostProcessor和BeanPactoryPostProcessor定义要特别小心。这些方法通常应该声明为静态@Bean方法，而不是触发其包含的配置类的实例化。否则，@Autowired和@Value可能无法在配置类本身上工作，因为可以将其创建为早于AutowiredNotationBeanPostProcessor的bean实例。

以下示例显示如何将一个bean自动连接到另一个bean：

```java
@Configuration
public class ServiceConfig {

    @Autowired
    private AccountRepository accountRepository;

    @Bean
    public TransferService transferService() {
        return new TransferServiceImpl(accountRepository);
    }
}

@Configuration
public class RepositoryConfig {

    private final DataSource dataSource;

    public RepositoryConfig(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Bean
    public AccountRepository accountRepository() {
        return new JdbcAccountRepository(dataSource);
    }
}

@Configuration
@Import({ServiceConfig.class, RepositoryConfig.class})
public class SystemTestConfig {

    @Bean
    public DataSource dataSource() {
        // return new DataSource
    }
}

public static void main(String[] args) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(SystemTestConfig.class);
    // everything wires up across configuration classes...
    TransferService transferService = ctx.getBean(TransferService.class);
    transferService.transfer(100.00, "A123", "C456");
}
```

> @Configuration类中的构造函数注入仅在Spring Framework 4.3中受支持。还要注意，如果目标bean只定义一个构造函数，则不需要指定@Autowired。

##### 完全限定导入的bean以便于导航

在前面的场景中，使用@Autowired可以很好地工作并提供所需的模块化，但是确定被注入bean定义的确切声明位置仍然有点模棱两可。例如，作为一名查看ServiceConfig的开发人员，您如何确切地知道@Autowired AccountRepository bean的声明位置？它在代码中不显式，这可能很好。请记住，springtoolsforeclipse提供了一种工具，可以呈现图形，显示一切是如何连接的，这可能就是您所需要的。另外，您的JavaIDE可以轻松找到AccountRepository类型的所有声明和用法，并快速显示返回该类型的@Bean方法的位置。
在这种歧义是不可接受的情况下，并且希望从IDE内从一个“配置”类到另一个“配置”类的直接导航时，请考虑自动配置配置本身。以下示例显示了如何执行此操作：

```java
@Configuration
public class ServiceConfig {

    @Autowired
    private RepositoryConfig repositoryConfig;

    @Bean
    public TransferService transferService() {
        // navigate 'through' the config class to the @Bean method!
        return new TransferServiceImpl(repositoryConfig.accountRepository());
    }
}
```

在前面的情况中，定义AccountRepository的位置是完全显式的。但是，ServiceConfig现在与RepositoryConfig紧密耦合。这就是折衷。通过使用基于接口或基于抽象类的@Configuration类，可以在一定程度上缓解这种紧密耦合。考虑下面的例子：

```java
@Configuration
public class ServiceConfig {

    @Autowired
    private RepositoryConfig repositoryConfig;

    @Bean
    public TransferService transferService() {
        return new TransferServiceImpl(repositoryConfig.accountRepository());
    }
}

@Configuration
public interface RepositoryConfig {

    @Bean
    AccountRepository accountRepository();
}

@Configuration
public class DefaultRepositoryConfig implements RepositoryConfig {

    @Bean
    public AccountRepository accountRepository() {
        return new JdbcAccountRepository(...);
    }
}

@Configuration
@Import({ServiceConfig.class, DefaultRepositoryConfig.class})  // import the concrete config!
public class SystemTestConfig {

    @Bean
    public DataSource dataSource() {
        // return DataSource
    }

}

public static void main(String[] args) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(SystemTestConfig.class);
    TransferService transferService = ctx.getBean(TransferService.class);
    transferService.transfer(100.00, "A123", "C456");
}
```

现在ServiceConfig与具体的DefaultRepositoryConfig松散耦合，内置IDE工具仍然很有用：您可以轻松获得RepositoryConfig实现的类型层次结构。这样，导航@Configuration类及其依赖项与导航基于接口的代码的通常过程没有什么不同。

> 如果您想影响某些bean的启动创建顺序，请考虑将其中一些bean声明为@Lazy（用于创建第一个访问而不是启动），或者作为@DependsOn某些其他bean（确保在当前bean之前创建特定的其他bean，而不是后者的直接依赖性暗示）。

#### 有条件地包括@Configuration类或@Bean方法

根据某些任意系统状态，有条件地启用或禁用完整的@Configuration类，甚至单个@Bean方法通常很有用。一个常见的例子是，只有在Spring环境中启用了特定的概要文件时，才使用@Profile注释激活Bean（有关详细信息，请参阅Bean定义概要文件）。

@Profile注释实际上是通过使用更灵活的注释@Conditional实现的。@Conditional注释表示特定的`org.springframework.context.annotation.Condition`在注册@Bean之前应咨询的实现。

Condition接口的实现提供了一个matches（…) 方法返回true或false。例如，下面的列表显示了@Profile使用的实际条件实现：

```java
@Override
public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
    // Read the @Profile annotation attributes
    MultiValueMap<String, Object> attrs = metadata.getAllAnnotationAttributes(Profile.class.getName());
    if (attrs != null) {
        for (Object value : attrs.get("value")) {
            if (context.getEnvironment().acceptsProfiles(((String[]) value))) {
                return true;
            }
        }
        return false;
    }
    return true;
}
```

有关更多详细信息，请参阅[@Conditional](https://docs.spring.io/spring-framework/docs/5.3.14/javadoc-api/org/springframework/context/annotation/Conditional.html) javadoc。

#### 结合Java和XML配置

Spring的@Configuration类支持并不打算100%完全替代springxml。一些工具，如SpringXML名称空间，仍然是配置容器的理想方式。在XML方便或必要的情况下，您可以选择：或者使用“以XML为中心”的方式实例化容器，例如使用ClassPathXmlApplicationContext，或者使用AnnotationConfigApplicationContext和@ImportResource注释以“以Java为中心”的方式实例化容器，以便根据需要导入XML。

##### 以XML为中心使用@Configuration类

最好从XML引导Spring容器，并以特别的方式包含@Configuration类。例如，在使用SpringXML的大型现有代码库中，更容易根据需要创建@Configuration类，并从现有XML文件中包含它们。在本节后面，我们将介绍在这种“以XML为中心”的情况下使用@Configuration类的选项。

##### 将@Configuration类声明为纯Spring\<bean/\>元素

请记住，@Configuration类最终是容器中的bean定义。在本系列示例中，我们创建了一个名为AppConfig的@Configuration类，并将其包含在系统测试配置中。xml作为\<bean/\>定义。由于\<context:annotation-config/\>已打开，因此容器可以识别@Configuration注释并正确处理AppConfig中声明的@Bean方法。
以下示例显示了Java中的一个普通配置类：

```java
@Configuration
public class AppConfig {

    @Autowired
    private DataSource dataSource;

    @Bean
    public AccountRepository accountRepository() {
        return new JdbcAccountRepository(dataSource);
    }

    @Bean
    public TransferService transferService() {
        return new TransferService(accountRepository());
    }
}
```

以下示例显示了system-test-config.xml文件的一部分:

```xml
<beans>
    <!-- enable processing of annotations such as @Autowired and @Configuration -->
    <context:annotation-config/>
    <context:property-placeholder location="classpath:/com/acme/jdbc.properties"/>

    <bean class="com.acme.AppConfig"/>

    <bean class="org.springframework.jdbc.datasource.DriverManagerDataSource">
        <property name="url" value="${jdbc.url}"/>
        <property name="username" value="${jdbc.username}"/>
        <property name="password" value="${jdbc.password}"/>
    </bean>
</beans>
```

下面的示例显示了一种可能的jdbc.properties文件：

```properties
jdbc.url=jdbc:hsqldb:hsql://localhost/xdb
jdbc.username=sa
jdbc.password=
```

```java
public static void main(String[] args) {
    ApplicationContext ctx = new ClassPathXmlApplicationContext("classpath:/com/acme/system-test-config.xml");
    TransferService transferService = ctx.getBean(TransferService.class);
    // ...
}
```

> 在system-test-config.properties.xml文件中，AppConfig\<bean/\>不声明id元素。虽然这样做是可以接受的，但这是不必要的，因为没有其他bean引用它，而且不太可能通过名称显式地从容器中获取它。类似地，DataSourceBean仅按类型自动连接，因此不严格要求显式bean id。

##### 使用\<context:component-scan/\>获取@Configuration类

因为@Configuration是用@Component进行元注释的，@Configuration注释的类自动成为组件扫描的候选对象。使用前面示例中描述的相同场景，我们可以重新定义系统测试配置。xml以利用组件扫描。注意，在这种情况下，我们不需要显式声明\<context:annotation-config/\>，因为\<context:component-scan/\>启用了相同的功能。

##### 以@Configuration类为中心使用带有@ImportResource的XML

在@Configuration类是配置容器的主要机制的应用程序中，可能仍然需要至少使用一些XML。在这些场景中，您可以使用@ImportResource并只定义所需的XML。这样做可以实现一种“以Java为中心”的方法来配置容器，并将XML保持在最低限度。以下示例（包括一个配置类、一个定义bean的XML文件、一个属性文件和一个主类）显示了如何使用@ImportResource注释来实现“以Java为中心”的配置，该配置根据需要使用XML：

```java
@Configuration
@ImportResource("classpath:/com/acme/properties-config.xml")
public class AppConfig {

    @Value("${jdbc.url}")
    private String url;

    @Value("${jdbc.username}")
    private String username;

    @Value("${jdbc.password}")
    private String password;

    @Bean
    public DataSource dataSource() {
        return new DriverManagerDataSource(url, username, password);
    }
}
```

```xml
properties-config.xml
<beans>
    <context:property-placeholder location="classpath:/com/acme/jdbc.properties"/>
</beans>
```

```properties
jdbc.properties
jdbc.url=jdbc:hsqldb:hsql://localhost/xdb
jdbc.username=sa
jdbc.password=
```

```java
public static void main(String[] args) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
    TransferService transferService = ctx.getBean(TransferService.class);
    // ...
}
```

## 1.13 环境抽象

Enviroment接口是集成在容器中的抽象，它对应用程序环境的两个关键方面建模：profiles和properties。

profiles文件是一个命名的、逻辑的bean定义组，只有在给定概要文件处于活动状态时，才会向容器注册。bean可以被分配给一个概要文件，不管是用XML定义的还是带有注释的。与概要文件相关的环境对象的作用是确定哪些概要文件（如果有）当前处于活动状态，以及默认情况下哪些概要文件（如果有）应处于活动状态。

properties在几乎所有的应用程序中都扮演着重要的角色，可能来自各种来源：属性文件、JVM系统属性、系统环境变量、JNDI、servlet上下文参数、特殊属性对象、映射对象等等。与属性相关的环境对象的作用是为用户提供一个方便的服务界面，用于配置属性源和解析属性。

### 1.13.1 Bean定义概要文件

Bean Definition Profiles文件在核心容器中提供了一种机制，允许在不同环境中注册不同的Bean。“环境”一词对不同的用户意味着不同的东西，此功能可以帮助处理许多用例，包括：

- 在开发中使用内存中的数据源，而不是在QA或生产中从JNDI查找相同的数据源。
- 仅在将应用程序部署到性能环境时注册监视基础结构。
- 为客户A和客户B部署注册定制的bean实现。

考虑实际应用中需要数据源的第一个用例。在测试环境中，配置可能类似于以下内容：

```java
@Bean
public DataSource dataSource() {
    return new EmbeddedDatabaseBuilder()
        .setType(EmbeddedDatabaseType.HSQL)
        .addScript("my-schema.sql")
        .addScript("my-test-data.sql")
        .build();
}
```

现在考虑如何将这个应用程序部署到QA或生产环境中，假设应用程序的数据源已注册到生产应用服务器的JNDI目录。我们的dataSource bean现在看起来如下所示：

```java
@Bean(destroyMethod="")
public DataSource dataSource() throws Exception {
    Context ctx = new InitialContext();
    return (DataSource) ctx.lookup("java:comp/env/jdbc/datasource");
}
```

问题是如何根据当前环境在使用这两种变体之间切换。随着时间的推移，Spring用户已经设计了许多方法来实现这一点，通常依赖于系统环境变量和包含${placeholder}标记的XML\<import/\>语句的组合，这些标记根据环境变量的值解析为正确的配置文件路径。Bean定义概要文件是一个核心容器特性，它为这个问题提供了解决方案。

如果我们概括前面的特定于环境的bean定义示例中所示的用例，我们最终需要在某些上下文中注册某些bean定义，而不是在其他上下文中注册。您可以说，您希望在情况a中注册bean定义的某个概要文件，在情况B中注册另一个概要文件。我们首先更新配置以反映这一需求。

#### 使用@Profile

@Profile注释允许您在一个或多个指定的概要文件处于活动状态时指示组件符合注册条件。使用前面的示例，我们可以按如下方式重写数据源配置：

```java
@Configuration
@Profile("development")
public class StandaloneDataConfig {

    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.HSQL)
            .addScript("classpath:com/bank/config/sql/schema.sql")
            .addScript("classpath:com/bank/config/sql/test-data.sql")
            .build();
    }
}
```

```java
@Configuration
@Profile("production")
public class JndiDataConfig {

    @Bean(destroyMethod="")
    public DataSource dataSource() throws Exception {
        Context ctx = new InitialContext();
        return (DataSource) ctx.lookup("java:comp/env/jdbc/datasource");
    }
}
```

> 如前所述，对于@Bean方法，您通常选择使用编程JNDI查找，方法是使用Spring的JNDemplate/JndiLocatorDelegate帮助程序或前面所示的直接JNDI InitialContext用法，而不是JndiObjectFactoryBean变体，这将迫使您将返回类型声明为FactoryBean类型。

配置文件字符串可能包含简单的配置文件名称（例如，production）或配置文件表达式。配置文件表达式允许表达更复杂的配置文件逻辑（例如，production&us east）。配置文件表达式中支持以下运算符：

- !: 配置文件的逻辑“非”
- &:配置文件的逻辑“和”
- |:配置文件的逻辑“或”

> 如果不使用括号，则不能混合使用&和|运算符。例如，production&us east | eu central不是一个有效的表达。它必须表示为production &（us-east| eu-central）。

您可以使用@Profile作为元注释来创建自定义组合注释。以下示例定义了一个自定义@Production注释，您可以将其用作@Profile（“Production”）的插入式替换：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Profile("production")
public @interface Production {
}
```

> 如果@Configuration类被标记为@Profile，那么所有与该类关联的@Bean方法和@Import注释都将被绕过，除非一个或多个指定的概要文件处于活动状态。如果@Component或@Configuration类用@Profile（{“p1”，“p2”}）标记，则除非激活了配置文件“p1”或“p2”，否则不会注册或处理该类。如果给定配置文件的前缀为NOT运算符（！），仅当配置文件未处于活动状态时，才注册带注释的元素。例如，给定@Profile（{“p1”，“！p2”}），如果配置文件“p1”处于活动状态或配置文件“p2”处于非活动状态，则将发生注册。

@Configuration文件也可以在方法级别声明为只包含配置类的一个特定bean（例如，对于特定bean的替代变量），如下例所示：

```java
@Configuration
public class AppConfig {

    @Bean("dataSource")
    @Profile("development") 
    public DataSource standaloneDataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.HSQL)
            .addScript("classpath:com/bank/config/sql/schema.sql")
            .addScript("classpath:com/bank/config/sql/test-data.sql")
            .build();
    }

    @Bean("dataSource")
    @Profile("production") 
    public DataSource jndiDataSource() throws Exception {
        Context ctx = new InitialContext();
        return (DataSource) ctx.lookup("java:comp/env/jdbc/datasource");
    }
}
```

- `standaloneDataSource`方法仅在开发配置文件中可用。
- `jndiDataSource` 方法仅在生产配置文件中可用。

> 对于@Bean方法上的@Profile，可能会应用一种特殊的场景：对于相同Java方法名称的重载@Bean方法（类似于构造函数重载），需要在所有重载方法上一致地声明@Profile条件。如果条件不一致，则重载方法中只有第一个声明上的条件起作用。因此，@Profile不能用于选择具有特定参数签名的重载方法。同一bean的所有工厂方法之间的解析在创建时遵循Spring的构造函数解析算法。
> 如果要定义具有不同概要条件的替代bean，请使用@bean name属性使用指向相同bean名称的不同Java方法名称，如前一示例所示。如果参数签名都是相同的（例如，所有变体都没有arg工厂方法），这是在有效Java类中首先表示这种安排的唯一方法（因为只有一个特定名称和参数签名的方法）。
