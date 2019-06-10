---
title: Spring Security架构
date: 2019-04-10 10:03:59
tags: spring security
categories: Technology
---

# 身份验证和访问控制

应用程序安全性可归结为两个或多或少独立的问题：身份验证（您是谁？）和授权（您可以做什么？）。有时人们会说“访问控制”而不是“授权”，这可能会让人感到困惑，但以这种方式来思考是有帮助的，因为“授权”会在其他地方超载。Spring Security的架构旨在将身份验证与授权分开，并为两者提供策略和扩展点。

## 认证

身份验证的主要策略接口`AuthenticationManager`只有一个方法：

```java
public interface AuthenticationManager {

  Authentication authenticate(Authentication authentication)
    throws AuthenticationException;

}
```

AuthenticationManager方法可以做如下三个事情：

1. 如果它可以验证输入是否代表有效的主体，则返回`Authentication`（通常带有`authenticated=true`）。
2. `AuthenticationException`如果它认为输入代表无效的主体，则抛出一个。
3. `null`如果它不能决定返回。

`AuthenticationException`是运行时异常。它通常由应用程序以通用方式处理，具体取决于应用程序的样式或用途。换句话说，通常不希望用户代码捕获并处理它。例如，Web UI将呈现一个页面，表明身份验证失败，后端HTTP服务将发送401响应，有或没有`WWW-Authenticate`标头，具体取决于上下文。

最常用的实现`AuthenticationManager`是`ProviderManager`，它委托给一组`AuthenticationProvider`实例。`AuthenticationProvider`有点像`AuthenticationManager`，但它有一个额外的方法允许调用者查询它是否支持给定的`Authentication`类型：

```java
public interface AuthenticationProvider {

	Authentication authenticate(Authentication authentication)
			throws AuthenticationException;

	boolean supports(Class<?> authentication);

}
```

`supports()`方法中的参数`Class<?>`类型是`Class<? extends Authentication>`（只会询问它是否支持将传递给`authenticate()`方法的内容）。A `ProviderManager`可以通过委托给一个链来支持同一个应用程序中的多个不同的身份验证机制`AuthenticationProviders`。如果一个 `ProviderManager`无法识别特定的`Authentication`实例类型，则会跳过它。

A `ProviderManager`有一个可选的父级，如果所有提供者都返回，它可以查询`null`。如果父母不可用，那么在`AuthenticationException`中返回`null` `Authentication`的结果。

有时，应用程序具有受保护资源的逻辑组（例如，与路径模式匹配的所有Web资源`/api/**`），并且每个组都可以拥有自己的专用`AuthenticationManager`。通常，每个组都有一个 `ProviderManager`，他们共享父母。然后，父母就是一种“全局”资源，充当所有提供者的后备资源。

![具有共同父母的ProviderManagers](https://github.com/spring-guides/top-spring-security-architecture/raw/master/images/authentication.png)

图1. `AuthenticationManager`使用的 层次结构 `ProviderManager`  

## 自定义`AuthenticationManager`

Spring Security提供了一些配置帮助程序，可以快速获取应用程序中设置的常见身份验证管理器功能。最常用的帮助程序`AuthenticationManagerBuilder`非常适合设置内存、JDBC或LDAP用户详细信息，或用于添加自定义`UserDetailsService`。以下是配置全局（父）的应用程序示例`AuthenticationManager`：

```java
@Configuration
public class ApplicationSecurity extends WebSecurityConfigurerAdapter {

   ... // web stuff here

  @Autowired
  public initialize(AuthenticationManagerBuilder builder, DataSource dataSource) {
    builder.jdbcAuthentication().dataSource(dataSource).withUser("dave")
      .password("secret").roles("USER");
  }

}
```

 此示例涉及Web应用程序，但其使用范围`AuthenticationManagerBuilder`更广泛（有关如何实现Web应用程序安全性的更多详细信息，请参阅下文）。请注意，`AuthenticationManagerBuilder`是`@Autowired`进一个方法`@Bean`- 这是使它构建全局（父）`AuthenticationManager`的方法。相反，如果我们这样做：

```java
@Configuration
public class ApplicationSecurity extends WebSecurityConfigurerAdapter {

  @Autowired
  DataSource dataSource;

   ... // web stuff here

  @Override
  public configure(AuthenticationManagerBuilder builder) {
    builder.jdbcAuthentication().dataSource(dataSource).withUser("dave")
      .password("secret").roles("USER");
  }

}
```

（在配置器中使用`@Override`的方法）然后`AuthenticationManagerBuilder`仅用于构建“本地” `AuthenticationManager`，它是全局的子节点。在Spring Boot应用程序中，您可以`@Autowired`将全局应用程序注入另一个bean，但除非您自己明确地公开它，否则不能使用本地bean。

Spring Boot提供了一个默认的全局`AuthenticationManager`（只有一个用户），除非你通过提供自己的`AuthenticationManager`类型的bean来抢占它。除非您主动需要自定义全局`AuthenticationManager`，否则默认设置足够安全，您不必担心它。如果您执行任何构建`AuthenticationManager`的配置，您通常可以在本地执行您正在保护的资源，而不必担心全局默认值。

## 授权或访问控制

  一旦身份验证成功，我们就可以继续授权，这里的核心策略是`AccessDecisionManager`。框架提供了三个实现，并且所有三个委托给一个`AccessDecisionVoter`链，有点像`ProviderManager`委托`AuthenticationProviders`。

一个`AccessDecisionVoter`考虑一个`Authentication`（表示主体）和一个被`ConfigAttributes`装饰的安全的`Object`：

```java
boolean supports(ConfigAttribute attribute);

boolean supports(Class<?> clazz);

int vote(Authentication authentication, S object,
        Collection<ConfigAttribute> attributes);
```

该`Object`在`AccessDecisionManager`和`AccessDecisionVoter`的签名是完全通用的-它代表用户可能要访问的任何东西（网络资源或Java类中的方法是两种最常见的情况）。该`ConfigAttributes`也相当通用的，它代表安全装饰的`Object`，它拥有决定访问它所需的权限级别的元数据。`ConfigAttribute`是一个接口，但它只有一个非常通用的方法并返回一个`String`，所以这些字符串以某种方式编码资源所有者的意图，表达允许谁访问它的规则。典型的`ConfigAttribute`是用户角色的名称（如`ROLE_ADMIN`或`ROLE_AUDIT`），它们通常具有特殊格式（如`ROLE_` 前缀）或表示需要评估的表达式。

 大多数人只使用默认`AccessDecisionManager`——`AffirmativeBased`（如果没有voters拒绝，则获得准入）。定制都倾向于在voters中发生，要么添加新的定制，要么修改现有定制的方式。

在`ConfigAttributes`使用Spring表达式语言（SpEL）是很常见的，例如`isFullyAuthenticated() && hasRole('FOO')`。这是由一个`AccessDecisionVoter`可以处理表达式并为它们创建上下文的支持。要扩展可以处理的表达式范围，需要自定义实现`SecurityExpressionRoot`或者`SecurityExpressionHandler`。

## Web安全

Web层中的Spring Security（用于UI和HTTP后端）基于Servlet `Filters`，因此通常首先查看`Filters`的角色是有帮助的。下图显示了单个HTTP请求的处理程序的典型分层。

![过滤链委托给Servlet](https://github.com/spring-guides/top-spring-security-architecture/raw/master/images/filters.png)

客户端向应用程序发送请求，容器根据请求URI的路径决定哪些过滤器和哪个servlet应用于它。最多只有一个servlet可以处理单个请求，但是过滤器形成一个链，因此它们是有序的，实际上，如果过滤器想要处理请求本身，它可以否决链的其余部分。过滤器还可以修改下游过滤器和servlet中使用的请求和/或响应。过滤器链的顺序非常重要，Spring Boot通过两种机制管理它：一种是`Filter`类型里的`@Beans`可以有一个`@Order`或一个`Ordered`实现，另一种是它们可以是一个`FilterRegistrationBean`，它本身有一个Order作为其API的一部分。一些现成的过滤器定义自己的常量来帮助指示他们彼此相对的顺序（例如Spring Session中的`SessionRepositoryFilter`有一个值为`Integer.MIN_VALUE + 50`的`DEFAULT_ORDER`，它告诉我们它在链条前段，但它不排除它前面的其他过滤器。

Spring Security作为单个`Filter`被安装到链中，其具体类型是`FilterChainProxy`，由于很快就会显现的原因。在Spring Boot应用程序中，安全过滤器是一个位于`ApplicationContext`中的`@Bean`，并且默认安装，以便它应用于每个请求。它被安装的位置被定义为`SecurityProperties.DEFAULT_FILTER_ORDER`，该位置接着被`FilterRegistrationBean.REQUEST_WRAPPER_FILTER_MAX_ORDER`确定（Spring Boot应用程序期望过滤器在包装请求时修改其行为的最大顺序）。除此之外还有更多：从容器的角度来看，Spring Security是一个单独的过滤器，但在其中有一些额外的过滤器，每个过滤器都扮演着特殊的角色。这是一张图片：

![Spring安全过滤器](https://github.com/spring-guides/top-spring-security-architecture/raw/master/images/security-filters.png)

图2. Spring Security是一个物理实体， `Filter`但将处理委托给一系列内部过滤器

事实上，安全过滤器中甚至还有一层间接：它通常作为一个安装在容器中`DelegatingFilterProxy`，而不必是Spring `@Bean`。代理委托给一个`FilterChainProxy`，它是一个`@Bean`，通常具有固定名称`springSecurityFilterChain`。`FilterChainProxy`包含内部排列为过滤器链（或链）的所有安全逻辑。所有过滤器都具有相同的API（它们都实现了`Filter`Servlet规范中的接口），并且它们都有机会否决链的其余部分。  

可以有多个过滤器链，所有过滤器链都由Spring Security在同一顶层`FilterChainProxy`管理，并且容器都是未知的。Spring Security过滤器包含过滤器链列表，并将请求分派给与其匹配的第一个链。下图显示了基于匹配请求路径（`/foo/**`之前匹配`/**`）发生的调度。这是非常常见的，但不是匹配请求的唯一方法。此调度过程的最重要特征是只有一个链处理请求。

![安全过滤器调度](https://github.com/spring-guides/top-spring-security-architecture/raw/master/images/security-filters-dispatch.png)

图3. Spring Security `FilterChainProxy`将请求分派给匹配的第一个链。  

没有自定义安全配置的Spring Boot应用程序有若干个（称为N）过滤器链，通常n = 6。前（N-1）链那里只是忽略静态资源模式，如`/css/**`和`/images/**`以及误差视图`/error`（路径可以由用户用来控制`security.ignored`从`SecurityProperties`配置Bean）。最后一个链匹配所有路径即 `/**`并且更活跃，包含用于身份验证，授权，异常处理，会话处理，标题写入等的逻辑。默认情况下，此链中总共有11个过滤器，但通常没有必要让用户关注使用哪些过滤器以及何时使用过滤器。

Spring Security内部的所有过滤器都不为容器所知，这一点很重要，尤其是在Spring Boot应用程序中，默认情况下，所有`@Beans`类型`Filter`的容器都会自动注册到容器中。因此，如果要将自定义过滤器添加到安全链，则需要将其设置为`@Bean`或将其包装在`FilterRegistrationBean`，它明确禁用容器注册的过程。

## 创建和自定义筛选链

Spring Boot应用程序（带有`/**`请求匹配器的应用程序）中的默认回调过滤器链具有`SecurityProperties.BASIC_AUTH_ORDER`中预定义的顺序。您可以通过`security.basic.enabled=false`设置完全关闭它，或者您可以将其用作后备，只需使用较低的顺序定义其他规则。要做到这一点，只需添加一个`WebSecurityConfigurerAdapter`（或`WebSecurityConfigurer`）类型的`@Bean`并用`@Order`来装饰类。例：

```java
@Configuration
@Order(SecurityProperties.BASIC_AUTH_ORDER - 10)
public class ApplicationConfigurerAdapter extends WebSecurityConfigurerAdapter {
  @Override
  protected void configure(HttpSecurity http) throws Exception {
    http.antMatcher("/foo/**")
     ...;
  }
}
```

此bean将导致Spring Security添加新的过滤器链并排序在回调之前。

与另一组资源相比，许多应用程序对一组资源具有完全不同的访问规则。例如，承载UI和支持API的应用程序可能支持基于cookie的身份验证，其中重定向到UI部件的登录页面，而基于令牌的身份验证具有401响应未经身份验证的API部件请求。每组资源都有自己`WebSecurityConfigurerAdapter`的独特顺序和自己的请求匹配器。如果匹配规则重叠，则最早的有序过滤器链将获胜。

## 请求匹配调度和授权

安全过滤器链（或等效的 `WebSecurityConfigurerAdapter`）具有请求匹配器，用于决定是否将其应用于HTTP请求。一旦决定应用特定过滤器链，则不应用其他过滤器链。但是在过滤器链中，您可以通过在`HttpSecurity`配置器中设置其他匹配器来对授权进行更精细的控制。例：

```java
@Configuration
@Order(SecurityProperties.BASIC_AUTH_ORDER - 10)
public class ApplicationConfigurerAdapter extends WebSecurityConfigurerAdapter {
  @Override
  protected void configure(HttpSecurity http) throws Exception {
    http.antMatcher("/foo/**")
      .authorizeRequests()
        .antMatchers("/foo/bar").hasRole("BAR")
        .antMatchers("/foo/spam").hasRole("SPAM")
        .anyRequest().isAuthenticated();
  }
}
```

配置Spring Security最容易犯的错误之一就是忘记这些匹配器适用于不同的进程，一个是整个过滤器链的请求匹配器，另一个是选择要应用的访问规则。

## 将应用程序安全规则应用于Actuator Rules

如果您将Spring Boot Actuator用于管理端点，您可能希望它们是安全的，并且默认情况下它们将是安全的。实际上，只要将Actuator添加到安全应用程序中，您就会获得仅适用于执行器端点的附加过滤器链。它由一个仅匹配执行器端点的请求匹配器定义，并且其顺序`ManagementServerProperties.BASIC_AUTH_ORDER`比默认的`SecurityProperties`回调过滤器少5，因此在回退之前会查询它。

如果您希望将应用程序安全规则应用于actuator 端点，则可以添加比actuator 端点更早排序的过滤器链以及包含所有actuator 端点的请求匹配器。如果您更喜欢执行器端点的默认安全设置，那么最简单的方法是在actuator 端点之后添加自己的过滤器，但比回退更早（例如`ManagementServerProperties.BASIC_AUTH_ORDER + 1`）。例：

```java
@Configuration
@Order(ManagementServerProperties.BASIC_AUTH_ORDER + 1)
public class ApplicationConfigurerAdapter extends WebSecurityConfigurerAdapter {
  @Override
  protected void configure(HttpSecurity http) throws Exception {
    http.antMatcher("/foo/**")
     ...;
  }
}
```
Web层中的Spring Security目前与Servlet API相关联，因此它仅在嵌入式或其他方式在servlet容器中运行应用程序时才真正适用。但是，它不依赖于Spring MVC或Spring Web堆栈的其余部分，因此可以在任何servlet应用程序中使用，例如使用JAX-RS的应用程序。

## 方法安全

除了支持保护Web应用程序外，Spring Security还支持将访问规则应用于Java方法执行。对于Spring Security，这只是一种不同类型的“受保护资源”。对于用户来说，这意味着使用相同格式的`ConfigAttribute`字符串（例如角色或表达式）声明访问规则，但是在代码中的不同位置。第一步是启用方法安全性，例如在我们的应用程序的顶级配置中：

```java
@SpringBootApplication
@EnableGlobalMethodSecurity(securedEnabled = true)
public class SampleSecureApplication {
}
```

然后我们可以直接装饰方法资源，例如

```java
@Service
public class MyService {

  @Secured("ROLE_USER")
  public String secure() {
    return "Hello Security";
  }

}
```

此示例是具有安全方法的服务。如果Spring创建了`@Bean`这种类型，那么它将被代理，并且调用者必须在实际执行该方法之前通过安全拦截器。如果访问被拒绝，则调用者将获得`AccessDeniedException`而不是实际的方法结果。  

其他注解可以在方法中使用以执行安全性约束，特别是`@PreAuthorize`和`@PostAuthorize`，它分别允许你写含有参数引用的表达式和返回值的方法。

将Web安全性和方法安全性结合起来并不罕见。过滤器链提供用户体验功能，如身份验证和重定向到登录页面等，方法安全性可在更细粒度的级别提供保护。

# 使用线程

Spring Security基本上是线程绑定的，因为它需要使当前经过身份验证的主体可供各种下游消费者使用。基本构建块`SecurityContext`可能包含一个`Authentication`（当用户登录时，它将`Authentication`是明确的`authenticated`）。您始终可以访问和操作`SecurityContext`通过静态的方法，在`SecurityContextHolder`这些方法中依次操作 `TheadLocal`，例如

```java
SecurityContext context = SecurityContextHolder.getContext();
Authentication authentication = context.getAuthentication();
assert(authentication.isAuthenticated);
```

用户应用程序代码并不经常做这个操作，但它可以是有用的，比如，如果需要编写一个自定义的验证过滤器（虽然即使这样，也有Spring Security的基类，可用于需要避免使用`SecurityContextHolder`的）。

如果需要访问Web端点中当前经过身份验证的用户，则可以在`@RequestMapping`标记的方法中使用参数。例如

```java
@RequestMapping("/foo")
public String foo(@AuthenticationPrincipal User user) {
  ... // do stuff with user
}
```

此注解将`Authentication`拉出`SecurityContext`并调用其`getPrincipal()`上的方法以生成方法参数。`Principal`in中的类型`Authentication`取决于`AuthenticationManager`用于验证身份验证的内容，因此这对于获取对用户数据的类型安全引用来说是一个有用的小技巧。

如果spring security是使用`Principal`从`HttpServletRequest`将型的`Authentication`，所以你也可以直接使用：

```java
@RequestMapping("/foo")
public String foo(Principal principal) {
  Authentication authentication = (Authentication) principal;
  User = (User) authentication.getPrincipal();
  ... // do stuff with user
}
```

如果您需要编写在不使用Spring Security时有效的代码（在加载`Authentication`类时需要更加防御），这有时会很有用。

## 异步处理安全方法

由于`SecurityContext`是线程绑定的，如果要进行任何调用安全方法的后台处理，例如使用`@Async`，则需要确保传播上下文。这归结为包裹`SecurityContext`与任务（最多`Runnable`，`Callable`即在后台执行等）。Spring Security提供了一些帮助，使这更容易，例如包装`Runnable`和`Callable`。要传播需要提供的`SecurityContext`to `@Async`方法，`AsyncConfigurer`并确保其`Executor`类型正确：

```java
@Configuration
public class ApplicationConfiguration extends AsyncConfigurerSupport {

  @Override
  public Executor getAsyncExecutor() {
    return new DelegatingSecurityContextExecutorService(Executors.newFixedThreadPool(5));
  }

}  
```