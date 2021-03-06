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

#### XML Bean定义概要文件

XML对应项是\<beans\>元素的profile属性。我们前面的示例配置可以在两个XML文件中重写，如下所示：

```xml
<beans profile="development"
    xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:jdbc="http://www.springframework.org/schema/jdbc"
    xsi:schemaLocation="...">

    <jdbc:embedded-database id="dataSource">
        <jdbc:script location="classpath:com/bank/config/sql/schema.sql"/>
        <jdbc:script location="classpath:com/bank/config/sql/test-data.sql"/>
    </jdbc:embedded-database>
</beans>
```

```xml
<beans profile="production"
    xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:jee="http://www.springframework.org/schema/jee"
    xsi:schemaLocation="...">

    <jee:jndi-lookup id="dataSource" jndi-name="java:comp/env/jdbc/datasource"/>
</beans>
```

也可以避免在同一文件中拆分和嵌套\<beans/\>元素，如下例所示：

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:jdbc="http://www.springframework.org/schema/jdbc"
    xmlns:jee="http://www.springframework.org/schema/jee"
    xsi:schemaLocation="...">

    <!-- other bean definitions -->

    <beans profile="development">
        <jdbc:embedded-database id="dataSource">
            <jdbc:script location="classpath:com/bank/config/sql/schema.sql"/>
            <jdbc:script location="classpath:com/bank/config/sql/test-data.sql"/>
        </jdbc:embedded-database>
    </beans>

    <beans profile="production">
        <jee:jndi-lookup id="dataSource" jndi-name="java:comp/env/jdbc/datasource"/>
    </beans>
</beans>
```

spring-bean.xsd 已被约束为仅允许作为文件中最后一个元素的元素。这将有助于提供灵活性，而不会导致XML文件混乱。

> XML对应项不支持前面描述的配置文件表达式。但是，可以使用！操作人员也可以通过嵌套配置文件应用逻辑“和”，如下例所示：
>
> ```xml
> <beans xmlns="http://www.springframework.org/schema/beans"
>     xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
>     xmlns:jdbc="http://www.springframework.org/schema/jdbc"
>     xmlns:jee="http://www.springframework.org/schema/jee"
>     xsi:schemaLocation="...">
> 
>     <!-- other bean definitions -->
> 
>     <beans profile="production">
>         <beans profile="us-east">
>             <jee:jndi-lookup id="dataSource" jndi-name="java:comp/env/jdbc/datasource"/>
>         </beans>
>     </beans>
> </beans>
> ```
>
> 在前面的示例中，如果production和useast概要文件都处于活动状态，那么数据源bean将被公开。

#### 激活配置文件

现在我们已经更新了配置，我们仍然需要指示Spring哪个配置文件是活动的。如果我们现在启动示例应用程序，我们将看到抛出NoSuchBeanDefinitionException，因为容器找不到名为dataSource的Spring bean。

激活概要文件可以通过多种方式完成，但最简单的方法是通过ApplicationContext提供的环境API以编程方式完成。以下示例显示了如何执行此操作：

```java
AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
ctx.getEnvironment().setActiveProfiles("development");
ctx.register(SomeConfig.class, StandaloneDataConfig.class, JndiDataConfig.class);
ctx.refresh();
```

此外，还可以通过spring.profiles.active属性声明性地激活概述，可以通过系统环境变量、JVM系统属性、web.xml中的servlet上下文参数指定，甚至作为JNDI中的条目（请参阅PropertySource抽象）。在集成测试中，可以通过使用spring测试模块中的@ActiveProfiles注释来声明活动概要文件（请参阅环境概要文件的上下文配置）。

请注意，配置文件不是“非此即彼”的命题。您可以一次激活多个配置文件。通过编程，您可以为setActiveProfiles（）方法提供多个配置文件名称，该方法接受String… 可变参数。以下示例激活多个配置文件：

```java
ctx.getEnvironment().setActiveProfiles("profile1", "profile2");
```

声明性地说，spring.profiles.active可接受以逗号分隔的配置文件名称列表，如下例所示：

```sh
-Dspring.profiles.active="profile1,profile2"
```

#### 缺省概述

默认配置文件表示默认启用的配置文件。考虑下面的例子：

```java
@Configuration
@Profile("default")
public class DefaultDataConfig {

    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.HSQL)
            .addScript("classpath:com/bank/config/sql/schema.sql")
            .build();
    }
}
```

如果没有激活的配置文件，则创建datasource。您可以将此视为为为一个或多个bean提供默认定义的一种方式。如果启用了任何配置文件，则默认配置文件不适用。

您可以在环境中使用setDefaultProfiles（）更改默认配置文件的名称，或者以声明方式使用spring.profiles.default属性。

### 1.13.2 PropertySource抽象

Spring的环境抽象在可配置的属性源层次结构上提供搜索操作。考虑下面的列表：

```java
ApplicationContext ctx = new GenericApplicationContext();
Environment env = ctx.getEnvironment();
boolean containsMyProperty = env.containsProperty("my-property");
System.out.println("Does my environment contain the 'my-property' property? " + containsMyProperty);
```

在前面的代码片段中，我们看到了询问Spring是否为当前环境定义了my-property属性的高级方法。为了回答这个问题，环境对象对一组PropertySource对象执行搜索。PropertySource是对任何键值对源的简单抽象，Spring的StandardEnvironment配置了两个PropertySource对象 — 一个表示JVM系统属性集（system.getProperties（）），另一个表示系统环境变量集（system.getenv（））。

> 这些默认属性源用于StandardEnvironment，用于独立应用程序。StandardServleteEnvironment使用其他默认属性源填充，包括servlet配置和servlet上下文参数。它可以选择启用JndiPropertySource。有关详细信息，请参阅javadoc。

具体地说，当您使用StandardEnvironment时，调用env。如果运行时存在my-property系统属性或my-property环境变量，则containsProperty（“my-property”）返回true。

> 执行的搜索是分层的。默认情况下，系统属性优先于环境变量。因此，如果在调用env.getProperty（“my-properties”）期间恰好在两个位置都设置了my-property属性，系统属性值“wins”并返回。请注意，属性值不会合并，而是完全由前面的条目覆盖。
>
> 对于通用StandardServletEnvironment，完整的层次结构如下所示，最高优先级的条目位于顶部：
>
> 1. ServletConfig参数（如果适用 — 例如，对于DispatcherServlet上下文）
> 2. ServletContext参数（web.xml上下文参数条目）
> 3. JNDI环境变量（java:comp/env/entries）
> 4. JVM系统属性（-D个命令行参数）
> 5. JVM系统环境（操作系统环境变量）

最重要的是，整个机制是可配置的。也许您有一个自定义的属性源，希望将其集成到此搜索中。为此，实现并实例化您自己的PropertySource，并将其添加到当前环境的PropertySources集合中。以下示例显示了如何执行此操作：

```java
ConfigurableApplicationContext ctx = new GenericApplicationContext();
MutablePropertySources sources = ctx.getEnvironment().getPropertySources();
sources.addFirst(new MyPropertySource());
```

在前面的代码中，MyPropertySource在搜索中以最高优先级添加。如果它包含my-property属性，则会检测并返回该属性，以支持任何其他PropertySource中的任何my-property属性。MutablePropertySources API公开了许多方法，这些方法允许对属性源集进行精确操作。

### 1.13.3 使用@PropertySource

@PropertySource注释提供了一种方便的声明性机制，用于将PropertySource添加到Spring的环境中。

给定一个名为app.properties的文件。包含键值对testbean.name=myTestBean，下面的@Configuration类使用@PropertySource的方式是调用testBean.getName（）返回myTestBean：

```java
@Configuration
@PropertySource("classpath:/com/myco/app.properties")
public class AppConfig {

    @Autowired
    Environment env;

    @Bean
    public TestBean testBean() {
        TestBean testBean = new TestBean();
        testBean.setName(env.getProperty("testbean.name"));
        return testBean;
    }
}
```

任何 @PropertySource资源位置中存在的${...}占位符将根据已针对环境注册的属性源集进行解析，如下例所示：

```java
@Configuration
@PropertySource("classpath:/com/${my.placeholder:default/path}/app.properties")
public class AppConfig {

    @Autowired
    Environment env;

    @Bean
    public TestBean testBean() {
        TestBean testBean = new TestBean();
        testBean.setName(env.getProperty("testbean.name"));
        return testBean;
    }
}
```

假设my.placeholder出现在已注册的一个属性源中（例如，系统属性或环境变量），占位符将解析为相应的值。如果不是，则默认值/路径用作默认值。如果未指定默认值且无法解析属性，则会引发IllegalArgumentException。

> 根据Java8约定，@PropertySource注释是可重复的。但是，所有此类@PropertySource注释都需要在同一级别上声明，可以直接在配置类上声明，也可以在同一自定义注释中声明为元注释。不建议混合使用直接注释和元注释，因为直接注释可以有效地覆盖元注释。

### 1.13.4 语句中占位符的解析

历史上，元素中占位符的值只能根据JVM系统属性或环境变量解析。现在已经不是这样了。因为环境抽象集成在整个容器中，所以很容易通过它传递占位符的解析。这意味着您可以以任何方式配置解析过程。可以更改搜索系统属性和环境变量的优先级，也可以完全删除它们。您还可以根据需要将自己的属性源添加到混合中。
具体地说，只要客户属性在环境中可用，无论客户属性定义在何处，以下语句都有效：

```xml
<beans>
    <import resource="com/bank/service/${customer}-config.xml"/>
</beans>
```

## 1.14 注册LoadTimeWeaver

Spring使用LoadTimeWeaver在类加载到Java虚拟机（JVM）时动态转换类。

要启用加载时编织，可以将@EnableLoadTimeWeaving添加到其中一个@Configuration类中，如下例所示：

```java
@Configuration
@EnableLoadTimeWeaving
public class AppConfig {
}
```

或者，对于XML配置，您可以使用context:load-time-weaver元素：

```xml
<beans>
    <context:load-time-weaver/>
</beans>
```

为ApplicationContext配置后，该ApplicationContext中的任何bean都可以实现LoadTimeWeaverAware，从而接收对LoadTimeWeaver实例的引用。这在结合Spring的JPA支持时特别有用，因为JPA类转换可能需要加载时编织。有关更多详细信息，请参阅LocalContainerEntityManagerFactoryBean javadoc。有关AspectJ加载时编织的更多信息，请参阅Spring框架中使用AspectJ的加载时编织。

## 1.15 ApplicationContext的其他功能

正如引言一章中所讨论的，org.springframework.bean.factory包提供了管理和操作bean的基本功能，包括以编程方式。org.springframework.context包添加了ApplicationContext接口，该接口扩展了BeanFactory接口，此外还扩展了其他接口，以更面向应用程序框架的方式提供附加功能。许多人以完全声明的方式使用ApplicationContext，甚至不以编程方式创建它，而是依赖ContextLoader之类的支持类自动实例化ApplicationContext，作为JavaEEWeb应用程序正常启动过程的一部分。

为了以更面向框架的风格增强BeanFactory功能，上下文包还提供以下功能：

- 通过MessageSource接口访问i18n样式的消息。
- 通过ResourceLoader接口访问资源，如URL和文件。
- 事件发布，即通过使用ApplicationEventPublisher接口发布到实现ApplicationListener接口的bean。
- 加载多个（分层）上下文，通过HierarchycalBeanFactory接口将每个上下文集中在一个特定的层上，例如应用程序的web层。

### 1.15.1 使用MessageSource实现国际化

ApplicationContext接口扩展了名为MessageSource的接口，因此提供了国际化（“i18n”）功能。Spring还提供HierarchycalMessageSource接口，它可以分层解析消息。这些接口共同提供了Spring影响消息解决的基础。在这些接口上定义的方法包括：

- String getMessage(String code, Object[] args, String default, Locale loc)：用于从MessageSource检索消息的基本方法。当找不到指定区域设置的消息时，将使用默认消息。使用标准库提供的MessageFormat功能，传入的任何参数都将成为替换值。
- String getMessage(String code, Object[] args, Locale loc)：基本上与前面的方法相同，但有一个区别：不能指定默认消息。如果找不到消息，将引发NoSuchMessageException。
- String getMessage(MessageSourceResolvable resolvable, Locale locale)：前面方法中使用的所有属性也包装在一个名为MessageSourceResolvable的类中，您可以将该类与此方法一起使用。

加载ApplicationContext时，它会自动搜索上下文中定义的MessageSourcebean。bean的名称必须为messageSource。如果找到这样一个bean，那么对前面方法的所有调用都将委托给消息源。如果未找到消息源，ApplicationContext将尝试查找包含同名bean的父级。如果是，它将使用该bean作为消息源。如果ApplicationContext找不到任何消息源，将实例化一个空的DelegatingMessageSource，以便能够接受对上面定义的方法的调用。

Spring提供了三种MessageSource实现，ResourceBundleMessageSource、ReloadableResourceBundleMessageSource和StaticMessageSource。它们都实现了HierarchycalMessageSource，以便执行嵌套消息传递。StaticMessageSource很少使用，但提供了向源添加消息的编程方式。以下示例显示ResourceBundleMessageSource：

```xml
<beans>
    <bean id="messageSource"
            class="org.springframework.context.support.ResourceBundleMessageSource">
        <property name="basenames">
            <list>
                <value>format</value>
                <value>exceptions</value>
                <value>windows</value>
            </list>
        </property>
    </bean>
</beans>
```

该示例假设在类路径中定义了三个名为format、exceptions和windows的资源包。通过ResourceBundle对象解析消息的JDK标准方式处理解析消息的任何请求。在本例中，假设上述两个资源包文件的内容如下：

```properties
# in format.properties
message=Alligators rock!
```

```properties
# in exceptions.properties
argument.required=The {0} argument is required.
```

下一个示例显示了运行MessageSource功能的程序。请记住，所有ApplicationContext实现也是MessageSource实现，因此可以转换为MessageSource接口。

```java
public static void main(String[] args) {
    MessageSource resources = new ClassPathXmlApplicationContext("beans.xml");
    String message = resources.getMessage("message", null, "Default", Locale.ENGLISH);
    System.out.println(message);
}
```

上述程序的结果输出如下：

```text
Alligators rock!
```

总之，MessageSource是在一个名为beans.xml的文件中定义的，它存在于类路径的根目录下。MessageSourceBean定义通过其basenames属性引用了许多资源束。在列表中传递给basenames属性的三个文件作为文件存在于类路径的根目录下，称为format.properties、exceptions.properties和windows.properties。

下一个示例显示传递给消息查找的参数。这些参数将转换为字符串对象，并插入到查找消息中的占位符中。

```xml
<beans>

    <!-- this MessageSource is being used in a web application -->
    <bean id="messageSource" class="org.springframework.context.support.ResourceBundleMessageSource">
        <property name="basename" value="exceptions"/>
    </bean>

    <!-- lets inject the above MessageSource into this POJO -->
    <bean id="example" class="com.something.Example">
        <property name="messages" ref="messageSource"/>
    </bean>

</beans>
```

```java
public class Example {

    private MessageSource messages;

    public void setMessages(MessageSource messages) {
        this.messages = messages;
    }

    public void execute() {
        String message = this.messages.getMessage("argument.required",
            new Object [] {"userDao"}, "Required", Locale.ENGLISH);
        System.out.println(message);
    }
}
```

调用execute（）方法的结果输出如下：

```text
The userDao argument is required.
```

关于国际化（“i18n”），Spring的各种MessageSource实现遵循与标准JDK ResourceBundle相同的区域设置解析和回退规则。简而言之，继续前面定义的示例messageSource，如果您想根据英国（en-GB）语言环境解析消息，您将创建名为format_en_GB.properties、exceptions_en_GB.properties和windows_en_GB.properties。

通常，区域设置解析由应用程序的周围环境管理。在以下示例中，将手动指定解析（英国）消息所依据的区域设置：

```properties
# in exceptions_en_GB.properties
argument.required=Ebagum lad, the ''{0}'' argument is required, I say, required.
```

```java
public static void main(final String[] args) {
    MessageSource resources = new ClassPathXmlApplicationContext("beans.xml");
    String message = resources.getMessage("argument.required",
        new Object [] {"userDao"}, "Required", Locale.UK);
    System.out.println(message);
}
```

运行上述程序的结果输出如下：

```text
Ebagum lad, the 'userDao' argument is required, I say, required.
```

您还可以使用MessageSourceAware接口获取对已定义的任何MessageSource的引用。在创建和配置bean时，在实现MessageSourceAware接口的ApplicationContext中定义的任何bean都会被注入应用程序上下文的MessageSource。

> 因为Spring的MessageSource基于Java的ResourceBundle，所以它不会合并具有相同基本名称的bundle，而是只使用找到的第一个bundle。具有相同基本名称的后续消息包将被忽略。

> 作为ResourceBundleMessageSource的替代方案，Spring提供了一个ReloadableResourceBundleMessageSource类。此变体支持相同的捆绑包文件格式，但比标准的基于JDK的ResourceBundleMessageSource实现更灵活。特别是，它允许从任何Spring资源位置（不仅仅是从类路径）读取文件，并支持bundle属性文件的热重新加载（同时在两者之间高效地缓存它们）。有关详细信息，请参阅ReloadableResourceBundleMessageSourceJavadoc。

### 1.15.2 标准和自定义事件

ApplicationContext中的事件处理通过ApplicationEvent类和ApplicationListener接口提供。如果将实现ApplicationListener接口的bean部署到上下文中，则每次将ApplicationEvent发布到ApplicationContext时，都会通知该bean。本质上，这是标准的观察者设计模式。

> ApplicationContext中的事件处理通过ApplicationEvent类和ApplicationListener接口提供。如果将实现ApplicationListener接口的bean部署到上下文中，则每次将ApplicationEvent发布到ApplicationContext时，都会通知该bean。本质上，这是标准的观察者设计模式。

下表描述了Spring提供的标准事件：

| 事件                       | 解释                                                         |
| -------------------------- | ------------------------------------------------------------ |
| ContextRefreshedEvent      | 初始化或刷新ApplicationContext时发布（例如，使用ConfigurableApplicationContext接口上的refresh（）方法）。在这里，“初始化”意味着加载所有bean，检测并激活后处理器bean，预实例化单例，并且ApplicationContext对象已准备好使用。只要上下文尚未关闭，刷新可以触发多次，前提是所选的ApplicationContext实际上支持此类“热”刷新。例如，XmlWebApplicationContext支持热刷新，但GenericaApplicationContext不支持。 |
| ContextStartedEvent        | 使用ConfigurableApplicationContext接口上的start（）方法启动ApplicationContext时发布。这里，“启动”意味着所有生命周期bean都会收到一个明确的启动信号。通常，此信号用于在显式停止后重新启动bean，但也可用于启动尚未配置为autostart的组件（例如，初始化时尚未启动的组件）。 |
| ContextStoppedEvent        | 使用ConfigurableApplicationContext接口上的stop（）方法停止ApplicationContext时发布。这里，“stopped”意味着所有生命周期bean都会收到一个明确的停止信号。停止的上下文可以通过start（）调用重新启动。 |
| ContextClosedEvent         | 使用ConfigurableApplicationContext接口上的close（）方法或通过JVM关闭挂钩关闭ApplicationContext时发布。在这里，“closed”意味着所有单例bean都将被销毁。一旦上下文关闭，它将达到其生命周期的终点，并且无法刷新或重新启动。 |
| RequestHandledEvent        | 一个特定于web的事件，告诉所有bean HTTP请求已得到服务。此事件在请求完成后发布。此事件仅适用于使用Spring的DispatcherServlet的web应用程序。 |
| ServletRequestHandledEvent | RequestHandledEvent的子类，用于添加特定于Servlet的上下文信息。 |

您还可以创建和发布自己的自定义事件。以下示例显示了一个扩展Spring的ApplicationEvent基类的简单类：

```java
public class BlockedListEvent extends ApplicationEvent {

    private final String address;
    private final String content;

    public BlockedListEvent(Object source, String address, String content) {
        super(source);
        this.address = address;
        this.content = content;
    }

    // accessor and other methods...
}
```

要发布自定义ApplicationEvent，请在ApplicationEventPublisher上调用publishEvent（）方法。通常，这是通过创建一个实现ApplicationEventPublisherAware的类并将其注册为Springbean来完成的。以下示例显示了这样一个类：

```java
public class EmailService implements ApplicationEventPublisherAware {

    private List<String> blockedList;
    private ApplicationEventPublisher publisher;

    public void setBlockedList(List<String> blockedList) {
        this.blockedList = blockedList;
    }

    public void setApplicationEventPublisher(ApplicationEventPublisher publisher) {
        this.publisher = publisher;
    }

    public void sendEmail(String address, String content) {
        if (blockedList.contains(address)) {
            publisher.publishEvent(new BlockedListEvent(this, address, content));
            return;
        }
        // send email...
    }
}
```

在配置时，Spring容器检测到EmailService实现ApplicationEventPublisherAware并自动调用setApplicationEventPublisher（）。实际上，传入的参数是Spring容器本身。您正在通过其ApplicationEventPublisher接口与应用程序上下文交互。

要接收自定义ApplicationEvent，可以创建一个实现ApplicationListener的类，并将其注册为Springbean。以下示例显示了这样一个类：

```java
public class BlockedListNotifier implements ApplicationListener<BlockedListEvent> {

    private String notificationAddress;

    public void setNotificationAddress(String notificationAddress) {
        this.notificationAddress = notificationAddress;
    }

    public void onApplicationEvent(BlockedListEvent event) {
        // notify appropriate parties via notificationAddress...
    }
}
```

请注意，ApplicationListener通常使用自定义事件的类型进行参数化（上例中为BlockedListEvent）。这意味着onApplicationEvent（）方法可以保持类型安全，避免任何向下转换的需要。您可以注册任意数量的事件侦听器，但请注意，默认情况下，事件侦听器同步接收事件。这意味着publishEvent（）方法将一直阻塞，直到所有侦听器都处理完事件。这种同步单线程方法的一个优点是，当侦听器接收到事件时，如果事务上下文可用，它将在发布服务器的事务上下文中操作。如果需要另一种事件发布策略，请参阅javadoc for Spring的ApplicationEventMulticaster接口和SimpleApplicationEventMulticaster实现以了解配置选项。

以下示例显示了用于注册和配置上述每个类的bean定义：

```xml
<bean id="emailService" class="example.EmailService">
    <property name="blockedList">
        <list>
            <value>known.spammer@example.org</value>
            <value>known.hacker@example.org</value>
            <value>john.doe@example.org</value>
        </list>
    </property>
</bean>

<bean id="blockedListNotifier" class="example.BlockedListNotifier">
    <property name="notificationAddress" value="blockedlist@example.org"/>
</bean>
```

综上所述，当调用emailService bean的sendEmail（）方法时，如果有任何电子邮件应该被阻止，则会发布BlockedListEvent类型的自定义事件。blockedListNotifier bean注册为ApplicationListener并接收BlockedListEvent，此时它可以通知适当的各方。

> Spring的事件机制是为相同应用程序上下文中Springbean之间的简单通信而设计的。然而，对于更复杂的企业集成需求，单独维护的Spring集成项目为构建基于众所周知的Spring编程模型的轻量级、面向模式、事件驱动的体系结构提供了完整的支持。

#### 基于注释的事件侦听器

您可以使用@EventListener注释在托管bean的任何方法上注册事件侦听器。BlockedListNotifier可以重写如下：

```java
public class BlockedListNotifier {

    private String notificationAddress;

    public void setNotificationAddress(String notificationAddress) {
        this.notificationAddress = notificationAddress;
    }

    @EventListener
    public void processBlockedListEvent(BlockedListEvent event) {
        // notify appropriate parties via notificationAddress...
    }
}
```

方法签名再次声明它侦听的事件类型，但这次使用了灵活的名称，并且没有实现特定的侦听器接口。只要实际事件类型解析其实现层次结构中的泛型参数，就可以通过泛型缩小事件类型的范围。

如果您的方法应该侦听多个事件，或者如果您想定义它而不使用任何参数，那么还可以在注释本身上指定事件类型。以下示例显示了如何执行此操作：

```java
@EventListener({ContextStartedEvent.class, ContextRefreshedEvent.class})
public void handleContextStart() {
    // ...
}
```

还可以通过使用定义SpEL表达式的注释的condition属性添加额外的运行时筛选，该表达式应匹配以实际调用特定事件的方法。

以下示例显示了如何重写通知程序，使其仅在事件的content属性等于my event时调用：

```java
@EventListener(condition = "#blEvent.content == 'my-event'")
public void processBlockedListEvent(BlockedListEvent blEvent) {
    // notify appropriate parties via notificationAddress...
}
```

每个SpEL表达式根据专用上下文进行计算。下表列出了上下文可用的项，以便您可以将其用于条件事件处理：

| 名称            | 位置               | 描述                                                         | 例子                                                       |
| --------------- | ------------------ | ------------------------------------------------------------ | ---------------------------------------------------------- |
| Event           | root object        | 实际的ApplicationEvent。                                     | #root.event 和 event                                       |
| Arguments array | root object        | 用于调用方法的参数（作为对象数组）。                         | #root.args 或 args args[0]以访问第一个参数等。             |
| *Argument name* | evaluation context | 任何方法参数的名称。如果由于某种原因，名称不可用（例如，因为编译的字节码中没有调试信息），也可以使用#a<#arg>语法使用单个参数，其中<#arg>表示参数索引（从0开始）。 | #blEvent或#a0（也可以使用#p0或#p<#arg>参数表示法作为别名） |

请注意#root。事件允许您访问基础事件，即使您的方法签名实际上引用了已发布的任意对象。

如果需要发布事件作为处理另一个事件的结果，可以更改方法签名以返回应发布的事件，如下例所示：

```java
@EventListener
public ListUpdateEvent handleBlockedListEvent(BlockedListEvent event) {
    // notify appropriate parties via notificationAddress and
    // then publish a ListUpdateEvent...
}
```

> 异步侦听器不支持此功能。

handleBlockedListEvent（）方法为其处理的每个BlockedListEvent发布一个新的ListUpdateEvent。如果需要发布多个事件，则可以返回事件的集合或数组。

#### 异步侦听器

如果希望某个特定的侦听器异步处理事件，可以重用常规的@Async支持。以下示例显示了如何执行此操作：

```java
@EventListener
@Async
public void processBlockedListEvent(BlockedListEvent event) {
    // BlockedListEvent is processed in a separate thread
}
```

使用异步事件时，请注意以下限制：

- 如果异步事件侦听器引发异常，则不会将其传播到调用方。有关更多详细信息，请参阅AsyncUncaughtExceptionHandler。
- 异步事件侦听器方法无法通过返回值来发布后续事件。如果需要发布另一个事件作为处理的结果，请插入ApplicationEventPublisher以手动发布该事件。

#### 排序侦听器

如果需要先调用一个侦听器，然后再调用另一个侦听器，则可以将@Order注释添加到方法声明中，如下例所示：

```java
@EventListener
@Order(42)
public void processBlockedListEvent(BlockedListEvent event) {
    // notify appropriate parties via notificationAddress...
}
```

#### 泛型事件

您还可以使用泛型来进一步定义事件的结构。考虑使用一个EntityCreateEvent\<T\>，其中T是所创建的实际实体的类型。例如，您可以创建以下侦听器定义以仅接收Person的EntityCreateEvent：

```java
@EventListener
public void onPersonCreated(EntityCreatedEvent<Person> event) {
    // ...
}
```

由于类型擦除，仅当激发的事件解析了事件侦听器筛选的泛型参数（即类似类PersonCreateEvent扩展EntityCreateEvent\<Person\>{… }).

在某些情况下，如果所有事件都遵循相同的结构，这可能会变得非常乏味（上例中的事件就是这种情况）。在这种情况下，您可以实现ResolvableTypeProvider来指导框架超越运行时环境提供的功能。以下事件显示了如何执行此操作：

```java
public class EntityCreatedEvent<T> extends ApplicationEvent implements ResolvableTypeProvider {

    public EntityCreatedEvent(T entity) {
        super(entity);
    }

    @Override
    public ResolvableType getResolvableType() {
        return ResolvableType.forClassWithGenerics(getClass(), ResolvableType.forInstance(getSource()));
    }
}
```

> 这不仅适用于ApplicationEvent，还适用于作为事件发送的任何任意对象。

### 1.15.3 方便地访问低级资源

为了更好地使用和理解应用程序上下文，您应该熟悉Spring的资源抽象，如参考资料中所述。

应用程序上下文是ResourceLoader，可用于加载资源对象。资源本质上是JDK中java.net.URL类的一个功能更加丰富的版本。事实上，资源的实现包装了一个java.net.URL实例，如适用。资源可以以透明的方式从几乎任何位置获取低级资源，包括从类路径、文件系统位置、可以用标准URL描述的任何位置以及其他一些变体。如果资源位置字符串是没有任何特殊前缀的简单路径，则这些资源的来源是特定的，并且适合于实际的应用程序上下文类型。

您可以将部署到应用程序上下文中的bean配置为实现特殊的回调接口ResourceLoaderWare，以便在初始化时自动回调，并将应用程序上下文本身作为ResourceLoader传入。您还可以公开Resource类型的属性，以用于访问静态资源。它们像其他属性一样被注入其中。您可以将这些资源属性指定为简单的字符串路径，并依赖于在部署bean时从这些文本字符串到实际资源对象的自动转换。

提供给ApplicationContext构造函数的一个或多个位置路径实际上是资源字符串，并且以简单的形式根据特定的上下文实现进行适当处理。例如，ClassPathXmlApplicationContext将简单位置路径视为类路径位置。您还可以使用带有特殊前缀的位置路径（资源字符串）强制从类路径或URL加载定义，而不管实际的上下文类型如何。

### 1.15.4 应用程序启动跟踪

ApplicationContext管理Spring应用程序的生命周期，并围绕组件提供丰富的编程模型。因此，复杂应用程序可以具有同样复杂的组件图和启动阶段。

使用特定指标跟踪应用程序启动步骤有助于了解启动阶段花费的时间，但也可以作为更好地了解整个上下文生命周期的一种方式。.

AbstractApplicationContext（及其子类）使用ApplicationStartup进行检测，ApplicationStartup收集有关各个启动阶段的StartupStep数据：

- 应用程序上下文生命周期（基本包扫描、配置类管理）
- bean生命周期（实例化、智能初始化、后处理）
- 应用程序事件处理

下面是注释ConfigApplicationContext中的Instrumentation示例：

```java
// create a startup step and start recording
StartupStep scanPackages = this.getApplicationStartup().start("spring.context.base-packages.scan");
// add tagging information to the current step
scanPackages.tag("packages", () -> Arrays.toString(basePackages));
// perform the actual phase we're instrumenting
this.scanner.scan(basePackages);
// end the current step
scanPackages.end();
```

已使用多个步骤检测应用程序上下文。一旦记录下来，就可以使用特定的工具收集、显示和分析这些启动步骤。有关现有启动步骤的完整列表，您可以查看专用的附录部分。

默认的ApplicationStartup实现是一个无操作变量，以最小化开销。这意味着默认情况下，在应用程序启动期间不会收集任何指标。Spring框架附带了一个用于跟踪Java Flight Recorder启动步骤的实现：FlightRecorderApplicationStartup。若要使用此变量，必须在创建后立即将其实例配置为ApplicationContext。

如果开发人员提供自己的AbstractApplicationContext子类，或者希望收集更精确的数据，那么他们也可以使用ApplicationStartup基础结构。

> ApplicationStartup仅用于应用程序启动期间和核心容器；这决不是Java分析工具或指标库（如[Micrometer](https://micrometer.io/)的替代品。）

要开始收集自定义StartupStep，组件可以直接从应用程序上下文获取ApplicationStartup实例，使其组件实现ApplicationStartupAware，或者在任何注入点上请求ApplicationStartup类型。

> 开发人员在创建自定义启动步骤时不应使用“spring.*”命名空间。此名称空间保留供Spring内部使用，可能会发生更改。

### 1.15.5 方便的Web应用程序的ApplicationContext实例化

例如，可以使用ContextLoader以声明方式创建ApplicationContext实例。当然，您也可以使用其中一个ApplicationContext实现以编程方式创建ApplicationContext实例。

您可以使用ContextLoaderListener注册ApplicationContext，如下例所示：

```xml
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>/WEB-INF/daoContext.xml /WEB-INF/applicationContext.xml</param-value>
</context-param>

<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```

侦听器检查contextConfigLocation参数。如果参数不存在，默认情况下侦听器将使用/WEB-INF/applicationContext.xml。当参数确实存在时，侦听器使用预定义的分隔符（逗号、分号和空格）分隔字符串，并使用这些值作为搜索应用程序上下文的位置。也支持Ant样式的路径模式。例如/WEB-INF/\*Context.xml（用于名称以Context.xml结尾且位于WEB-INF目录中的所有文件）和/WEB-INF/\*\*/\*Context.xml（用于WEB-INF任何子目录中的所有此类文件）。

### 1.15.6 将Spring ApplicationContext部署为Java EE RAR文件

可以将Spring ApplicationContext部署为RAR文件，将上下文及其所有必需的bean类和库JAR封装在Java EE RAR部署单元中。这相当于引导一个独立的ApplicationContext（仅托管在JavaEE环境中）来访问JavaEE服务器设施。RAR部署是部署无头WAR文件的更自然的替代方案 — 实际上，一个没有任何HTTP入口点的WAR文件，仅用于在JavaEE环境中引导Spring ApplicationContext。

RAR部署非常适合于不需要HTTP入口点，而只包含消息端点和计划作业的应用程序上下文。这种上下文中的bean可以使用应用服务器资源，例如JTA事务管理器和JNDI绑定的JDBC数据源实例和JMS连接工厂实例，还可以向平台的JMX服务器注册 — 通过Spring的标准事务管理以及JNDI和JMX支持设施。应用程序组件还可以通过Spring的TaskExecutor抽象与应用服务器的JCA WorkManager交互。

有关RAR部署中涉及的配置详细信息，请参阅SpringContextResourceAdapter类的javadoc。

要将Spring ApplicationContext作为Java EE RAR文件进行简单部署，请执行以下操作：

1. 将所有应用程序类打包到一个RAR文件中（这是一个具有不同文件扩展名的标准JAR文件）。
2. 将所有必需的库jar添加到RAR归档的根目录中。
3. 添加META-INF/ra.xml部署描述符（如javadoc for SpringContextResourceAdapter中所示）和相应的SpringXMLBean定义文件（通常为META-INF/applicationContext.xml）。
4. 将生成的RAR文件放到应用程序服务器的部署目录中。

> 这种RAR部署单元通常是独立的。它们不向外界公开组件，甚至不向同一应用程序的其他模块公开组件。与基于RAR的ApplicationContext的交互通常通过与其他模块共享的JMS目的地进行。例如，基于RAR的ApplicationContext还可以调度一些作业或对文件系统中的新文件（或类似文件）作出反应。如果它需要允许从外部进行同步访问，它可以（例如）导出RMI端点，同一台机器上的其他应用程序模块可能会使用这些端点。

## 1.16 Bean Factory

BeanFactoryAPI为Spring的IoC功能提供了基础。它的特定契约主要用于与Spring和相关第三方框架的其他部分集成，其DefaultListableBeanFactory实现是更高级别的GenericaApplicationContext容器中的一个关键委托。

BeanFactory和相关接口（如BeanFactoryAware、InitializingBean、DisposableBean）是其他框架组件的重要集成点。通过不需要任何注释甚至反射，它们允许容器及其组件之间进行非常有效的交互。应用程序级bean可以使用相同的回调接口，但通常更喜欢通过注释或编程配置进行声明性依赖项注入。

请注意，核心BeanFactory API级别及其DefaultListableBeanFactory实现不对要使用的配置格式或任何组件注释进行假设。所有这些风格都是通过扩展（如XmlBeanDefinitionReader和AutowiredNotationBeanPostProcessor）引入的，并作为核心元数据表示在共享BeanDefinition对象上操作。这就是Spring容器如此灵活和可扩展的本质所在。

### 1.16.1 BeanFactory 或 ApplicationContext?

本节解释BeanFactory和ApplicationContext容器级别之间的差异以及对引导的影响。

除非有充分的理由不这样做，否则应该使用ApplicationContext，将GenericaApplicationContext及其子类AnnotationConfigApplicationContext作为自定义引导的常见实现。这些是Spring核心容器的主要入口点，用于所有常见目的：加载配置文件、触发类路径扫描、以编程方式注册bean定义和注释类，以及（从5.0开始）注册函数bean定义。

由于ApplicationContext包含BeanFactory的所有功能，因此通常建议不要使用普通BeanFactory，除非需要完全控制bean处理。在ApplicationContext（如GenericaApplicationContext实现）中，通过约定（即，通过bean名称或bean类型）检测几种bean — 特别是后处理器），而普通的DefaultListableBeanFactory对任何特殊bean都是不可知的。

对于许多扩展容器特性，例如注释处理和AOP代理，BeanPostProcessor扩展点是必不可少的。如果只使用普通的DefaultListableBeanFactory，则默认情况下不会检测和激活此类后处理器。这种情况可能令人困惑，因为您的bean配置实际上没有任何问题。相反，在这种情况下，需要通过额外的设置来完全引导容器。

下表列出了BeanFactory和ApplicationContext接口和实现提供的功能。

| 特点                                  | BeanFactory | ApplicationContext |
| ------------------------------------- | ----------- | ------------------ |
| Bean 初始化/注入                      | Yes         | Yes                |
| 集成生命周期管理                      | No          | Yes                |
| 自动BeanPostProcessor注册             | No          | Yes                |
| 自动BeanFactoryPostProcessor注册      | No          | Yes                |
| 方便的MessageSource访问（用于国际化） | No          | Yes                |
| 内置ApplicationEvent发布机制          | No          | Yes                |

要向DefaultListableBeanFactory显式注册bean后处理器，需要以编程方式调用addBeanPostProcessor，如下例所示：

```java
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
// populate the factory with bean definitions

// now register any needed BeanPostProcessor instances
factory.addBeanPostProcessor(new AutowiredAnnotationBeanPostProcessor());
factory.addBeanPostProcessor(new MyBeanPostProcessor());

// now start using the factory
```

要将BeanFactoryPostProcess应用于普通DefaultListableBeanFactory，需要调用其postProcessBeanFactory方法，如下例所示：

```java
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinitions(new FileSystemResource("beans.xml"));

// bring in some property values from a Properties file
PropertySourcesPlaceholderConfigurer cfg = new PropertySourcesPlaceholderConfigurer();
cfg.setLocation(new FileSystemResource("jdbc.properties"));

// now actually do the replacement
cfg.postProcessBeanFactory(factory);
```

在这两种情况下，显式注册步骤都不方便，这就是为什么在Spring支持的应用程序中，各种ApplicationContext变体比普通的DefaultListableBeanFactory更受欢迎的原因，特别是在典型的企业设置中，依赖BeanFactoryPostProcessor和BeanPostProcessor实例实现扩展容器功能时。

> AnnotationConfigApplicationContext注册了所有常见的批注后处理器，并且可以通过配置批注（如@EnableTransactionManagement）在封面下引入其他处理器。在Spring基于注释的配置模型的抽象级别上，bean后处理器的概念仅仅成为一个内部容器细节。

# 2. 资源

本章介绍Spring如何处理资源以及如何在Spring中使用资源。

## 2.1 介绍

不幸的是，Java的标准`java.net.URL`类和各种URL前缀的标准处理程序不足以让所有人都获得低级别的资源。例如，没有标准化的URL实现可用于访问需要从类路径或相对于ServletContext获取的资源。虽然可以为专门的URL前缀注册新的处理程序（类似于http:等前缀的现有处理程序），但这通常相当复杂，并且URL接口仍然缺少一些需要的功能，例如检查指向的资源是否存在的方法。

## 2.2 Resource接口

`Spring`的位于`org.springframework.core.io.`包中的`Resource`接口是一个更强大的接口，用于抽象对低级资源的访问。下面的列表提供了`Resource`接口的概述。有关更多详细信息，请参阅资源javadoc。

```java
public interface Resource extends InputStreamSource {

    boolean exists();

    boolean isReadable();

    boolean isOpen();

    boolean isFile();

    URL getURL() throws IOException;

    URI getURI() throws IOException;

    File getFile() throws IOException;

    ReadableByteChannel readableChannel() throws IOException;

    long contentLength() throws IOException;

    long lastModified() throws IOException;

    Resource createRelative(String relativePath) throws IOException;

    String getFilename();

    String getDescription();
}
```

如`Resource`接口的定义所示，它扩展了`InputStreamSource`接口。以下列表显示`InputStreamSource`接口的定义：

```java
public interface InputStreamSource {

    InputStream getInputStream() throws IOException;
}
```

资源接口中一些最重要的方法包括：

- `getInputStream()`: 查找并打开资源，返回InputStream以从资源中读取。每个调用都会返回一个新的InputStream。调用方负责关闭流。
- `exist()`: 返回一个`boolean`值，指示此资源是否实际以物理形式存在。
- `isOpen()`: 返回一个`boolean`值，指示此资源是否表示具有开放流的处理。如果为true，则不能多次读取InputStream，必须只读取一次，然后关闭以避免资源泄漏。对于所有常见的资源实现返回false，InputStreamResource除外。
- `getDescription()`: 返回此资源的说明，用于处理资源时的错误输出。这通常是资源的完全限定文件名或实际URL。

其他方法允许您获取表示资源的实际URL或文件对象（如果底层实现兼容并支持该功能）。

`Resource`接口的一些实现还为支持写入的资源实现扩展的`WritableResource`接口。

Spring本身广泛使用资源抽象，当需要资源时，它在许多方法签名中作为参数类型。某些Spring API中的其他方法（如各种`ApplicationContext`实现的构造函数）采用一个`String`，该字符串以未经修饰或简单的形式用于创建适合该上下文实现的资源，或者通过`String`路径上的特殊前缀，让调用方指定必须创建和使用特定的资源实现。

虽然资源接口在Spring和被Spring大量使用，但实际上，它在您自己的代码中作为一个通用实用程序类使用非常方便，以便访问资源，即使您的代码不知道或不关心Spring的任何其他部分。虽然这将您的代码耦合到Spring，但实际上它只将代码耦合到这一小部分实用程序类，这些实用程序类可以作为URL的更强大的替代品，并且可以被视为等同于用于此目的的任何其他库。

`Resource`抽象并不能取代功能。它尽可能地把它包起来。例如，一个URL资源包装一个URL并使用包装后的URL来完成它的工作。

## 2.3 内置Resource实现

### 2.3.1 UrlResource

`UrlResource`封装`java.net.URL`和可用于访问通常可通过URL访问的任何对象例如：文件，Https目标，FTP目标和其他。所有URL都有一个标准化的`String`表示形式，因此使用适当的标准化前缀来表示不同的URL类型。这包括file：用于访问文件系统路径，https：用于通过https协议访问资源，ftp：用于通过ftp访问资源，以及其他。

`UrlResource`由Java代码显式使用`UrlResource`构造函数创建，但通常在调用API方法时隐式创建，该方法采用表示路径的`String`参数。对于后一种情况，JavaBeans属性编辑器最终决定创建哪种类型的资源。如果路径字符串包含一个众所周知的（对于属性编辑器，即）前缀（例如classpath:），它将为该前缀创建一个适当的专用资源。但是，如果它不识别前缀，则会假定该字符串是标准URL字符串，并创建一个`UrlResource`。

### 2.3.2 ClassPathResource

此类表示应从类路径获取的资源。它使用线程上下文类加载器、给定类加载器或给定类来加载资源。

此`Resource`实现支持作为`java.io.File`的决议。如果类路径资源驻留在文件系统中，而不是驻留在jar中且未（通过servlet引擎或任何环境）扩展到文件系统的类路径资源，则为File。为了解决这个问题，各种资源实现始终支持作为`java.net.URL`解析。

`ClassPathResource`是由Java代码通过显式使用`ClassPathResource`构造函数创建的，但通常在调用API方法时隐式创建，该方法采用表示路径的字符串参数。对于后一种情况，JavaBeans属性编辑器识别字符串路径上的特殊前缀classpath:，并在这种情况下创建ClassPathResource。

### 2.3.3 FileSystemResource

这是一个`Resource`对于`java.io.File`实现。它也支持`java.nio.file.path`实现,应用Spring标准的基于字符串的路径转换，但通过`java.nioi.file.Files`执行所有操作。对于纯`java.nio.path.Path`基础支持时改用PathResource。FileSystemResource支持解析为`File`和`URL`。

### 2.3.4 PathResource

这是一个对于`java.nio.file.Path`处理,通过`Path`API执行所有操作和转换的`Resource`实现。它支持解析为文件和URL，还实现了扩展的`WritableResource`接口。`PathResource`实际上是一个纯`java.nio.path.Path`具有不同`createRelative`行为的`FileSystemResource`的基于路径的替代方案。

### 2.3.5 ServletContextResource

这是用于解释相关web应用程序根目录中的相对路径的ServletContext资源的`Resource`实现。

它始终支持流访问和URL访问，但允许仅当web应用程序存档已扩展且资源物理上位于文件系统上时`java.io.File`才能访问文件。它是否被扩展并放在文件系统上，或者直接从JAR或其他地方（比如数据库）访问（这是可以想象的），实际上取决于Servlet容器。

### 2.3.6 InputStreamResource

`InputStreamResource`是给定`InputStream`的资源实现。只有在没有特定的资源实现适用时才应使用它。特别是，在可能的情况下，首选`ByteArrayResource`或任何基于文件的资源实现。
与其他资源实现不同，这是已打开资源的描述符。因此，它从isOpen（）返回true。如果需要将资源描述符保留在某个位置，或者需要多次读取流，请不要使用它。

### 2.3.7 ByteArrayResource

这是给定字节数组的资源实现。它为给定字节数组创建ByteArrayInputStream。
它有助于从任何给定字节数组加载内容，而不必求助于一次性InputStreamResource。

## 2.4 ResourceLoader 接口

ResourceLoader接口是由可以返回（即加载）资源实例的对象实现的。以下列表显示了ResourceLoader接口定义：

```java
public interface ResourceLoader {

    Resource getResource(String location);

    ClassLoader getClassLoader();
}
```

所有应用程序上下文都实现ResourceLoader接口。因此，可以使用所有应用程序上下文来获取资源实例。
在特定应用程序上下文上调用getResource（）时，如果指定的位置路径没有特定前缀，则会返回适合该特定应用程序上下文的资源类型。例如，假设以下代码片段是针对ClassPathXmlApplicationContext实例运行的：

```java
Resource template = ctx.getResource("some/resource/path/myTemplate.txt");
```

针对ClassPathXmlApplicationContext，该代码返回ClassPathResource。如果对FileSystemXmlApplicationContext实例运行相同的方法，它将返回FileSystemResource。对于WebApplicationContext，它将返回ServletContextResource。它同样会为每个上下文返回适当的对象。
因此，您可以按照适合特定应用程序上下文的方式加载资源。
另一方面，您也可以通过指定特殊的classpath:前缀，强制使用ClassPathResource，而不管应用程序上下文类型如何，如下例所示：

```java
Resource template = ctx.getResource("classpath:some/resource/path/myTemplate.txt");
```

类似地，您可以通过指定任何标准`java.net.URL`前缀来强制使用UrlResource。以下示例使用file和https前缀：

```java
Resource template = ctx.getResource("file:///some/resource/path/myTemplate.txt");
```

```java
Resource template = ctx.getResource("https://myhost.com/resource/path/myTemplate.txt");
```

下表总结了将`String`对象转换为`Resource`对象的策略：

| 前缀       | 例子                           | 解释                                            |
| ---------- | ------------------------------ | ----------------------------------------------- |
| classpath: | classpath:com/myapp/config.xml | 从类路径加载。                                  |
| file:      | file:///data/config.xml        | 从文件系统加载为URL。另请参见文件系统资源警告。 |
| https:     | https://myserver/logo.png      | 作为URL加载。                                   |
| (none)     | /data/config.xml               | 取决于基础应用程序上下文。                      |

## 2.5 ResourcePatternResolver 接口

ResourcePatternResolver接口是ResourceLoader接口的扩展，它定义了将位置模式（例如，Ant样式的路径模式）解析为资源对象的策略。

```java
public interface ResourcePatternResolver extends ResourceLoader {

    String CLASSPATH_ALL_URL_PREFIX = "classpath*:";

    Resource[] getResources(String locationPattern) throws IOException;
}
```

如上所述，该接口还为类路径中的所有匹配资源定义了一个特殊的classpath\*：资源前缀。请注意，在这种情况下，资源位置应该是没有占位符的路径 — 例如，classpath\*：/config/bean.xml。JAR文件或类路径中的不同目录可以包含具有相同路径和相同名称的多个文件。有关使用classpath*：资源前缀支持通配符的更多详细信息，请参阅应用程序上下文构造函数资源路径及其子部分中的通配符。

可以检查传入的ResourceLoader（例如，通过ResourceLoaderAWare语义提供的ResourceLoader）是否也实现了此扩展接口。

`PathMatchingResourcePatternResolver`是一个独立的实现，可在`ApplicationContext`外部使用，ResourceArrayPropertyEditor也可用于填充`Resource[]bean`属性。`PathMatchingResourcePatternResolver`能够将指定的资源位置路径解析为一个或多个匹配的资源对象。源路径可以是一个简单的路径，该路径具有到目标资源的一对一映射，或者可以包含特殊的classpath*：前缀和/或内部Ant样式的正则表达式（使用Spring的`org.springframework.util.AntPathMatcher`实用程序进行匹配）。后者两者实际上都是通配符

> 任何标准ApplicationContext中的默认ResourceLoader实际上是实现ResourcePatternResolver接口的PathMatchingResourcePatternResolver的实例。ApplicationContext实例本身也是如此，它还实现了ResourcePatternResolver接口并委托给默认的PathMatchingResourcePatternResolver。
