<a name="Spring-IntegratingApacheShirointoSpringbasedApplications"></a>
#Integrating Apache Shiro into Spring-based Applications

This page covers the ways to integrate Shiro into [Spring](http://spring.io)-based applications.

Shiro's JavaBeans compatibility makes it perfectly suited to be configured via Spring XML or other Spring-based configuration mechanisms. Shiro applications need an application singleton `SecurityManager` instance. Note that this does not have to be a _static_ singleton, but there should only be a single instance used by the application, whether its a static singleton or not.

## <a name="Spring-StandaloneApplications"></a>Standalone Applications

Here is the simplest way to enable an application singleton `SecurityManager` in Spring applications:


``` xml
<!-- Define the realm you want to use to connect to your back-end security datasource: -->
<bean id="myRealm" class="...">
    ...
</bean>

<bean id="securityManager" class="org.apache.shiro.mgt.DefaultSecurityManager">
    <!-- Single realm app.  If you have multiple realms, use the 'realms' property instead. -->
    <property name="realm" ref="myRealm"/>
</bean>

<bean id="lifecycleBeanPostProcessor" class="org.apache.shiro.spring.LifecycleBeanPostProcessor"/>

<!-- For simplest integration, so that all SecurityUtils.* methods work in all cases, -->
<!-- make the securityManager bean a static singleton.  DO NOT do this in web         -->
<!-- applications - see the 'Web Applications' section below instead.                 -->
<bean class="org.springframework.beans.factory.config.MethodInvokingFactoryBean">
    <property name="staticMethod" value="org.apache.shiro.SecurityUtils.setSecurityManager"/>
    <property name="arguments" ref="securityManager"/>
</bean>
```

<a name="Spring-WebApplications"></a>
## Web Applications

Shiro has first-rate support for Spring web applications. In a web application, all Shiro-accessible web requests must go through a master Shiro Filter. This filter itself is extremely powerful, allowing for
ad-hoc custom filter chains to be executed based on any URL path expression.

Prior to Shiro 1.0, you had to use a hybrid approach in Spring web applications, defining the Shiro filter and
all of its configuration properties in web.xml but define the `SecurityManager` in Spring XML. This was a little frustrating since you couldn't 1) consolidate your configuration in one place and 2) leverage the configuration power of the more advanced Spring features, like the `PropertyPlaceholderConfigurer` or abstract beans to consolidate common configuration.

Now in Shiro 1.0 and later, all Shiro configuration is done in Spring XML providing access to the more robust Spring configuration mechanisms.

Here is how to configure Shiro in a Spring-based web application:

<a name="Spring-web.xml"></a>
### web.xml

In addition to your other Spring web.xml elements (`ContextLoaderListener`, `Log4jConfigListener`, etc), define the following filter and filter mapping:

``` xml
<!-- The filter-name matches name of a 'shiroFilter' bean inside applicationContext.xml -->
<filter>
    <filter-name>shiroFilter</filter-name>
    <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
    <init-param>
        <param-name>targetFilterLifecycle</param-name>
        <param-value>true</param-value>
    </init-param>
</filter>

...

<!-- Make sure any request you want accessible to Shiro is filtered. /* catches all -->
<!-- requests.  Usually this filter mapping is defined first (before all others) to -->
<!-- ensure that Shiro works in subsequent filters in the filter chain:             -->
<filter-mapping>
    <filter-name>shiroFilter</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```

<a name="Spring-applicationContext.xml"></a>
###applicationContext.xml

In your applicationContext.xml file, define the web-enabled `SecurityManager` and the 'shiroFilter' bean that will be referenced from `web.xml`.

``` xml
<bean id="shiroFilter" class="org.apache.shiro.spring.web.ShiroFilterFactoryBean">
    <property name="securityManager" ref="securityManager"/>
    <!-- override these for application-specific URLs if you like:
    <property name="loginUrl" value="/login.jsp"/>
    <property name="successUrl" value="/home.jsp"/>
    <property name="unauthorizedUrl" value="/unauthorized.jsp"/> -->
    <!-- The 'filters' property is not necessary since any declared javax.servlet.Filter bean  -->
    <!-- defined will be automatically acquired and available via its beanName in chain        -->
    <!-- definitions, but you can perform instance overrides or name aliases here if you like: -->
    <!-- <property name="filters">
        <util:map>
            <entry key="anAlias" value-ref="someFilter"/>
        </util:map>
    </property> -->
    <property name="filterChainDefinitions">
        <value>
            # some example chain definitions:
            /admin/** = authc, roles[admin]
            /docs/** = authc, perms[document:read]
            /** = authc
            # more URL-to-FilterChain definitions here
        </value>
    </property>
</bean>

<!-- Define any javax.servlet.Filter beans you want anywhere in this application context.   -->
<!-- They will automatically be acquired by the 'shiroFilter' bean above and made available -->
<!-- to the 'filterChainDefinitions' property.  Or you can manually/explicitly add them     -->
<!-- to the shiroFilter's 'filters' Map if desired. See its JavaDoc for more details.       -->
<bean id="someFilter" class="..."/>
<bean id="anotherFilter" class="..."> ... </bean>
...

<bean id="securityManager" class="org.apache.shiro.web.mgt.DefaultWebSecurityManager">
    <!-- Single realm app.  If you have multiple realms, use the 'realms' property instead. -->
    <property name="realm" ref="myRealm"/>
    <!-- By default the servlet container sessions will be used.  Uncomment this line
         to use shiro's native sessions (see the JavaDoc for more): -->
    <!-- <property name="sessionMode" value="native"/> -->
</bean>
<bean id="lifecycleBeanPostProcessor" class="org.apache.shiro.spring.LifecycleBeanPostProcessor"/>

<!-- Define the Shiro Realm implementation you want to use to connect to your back-end -->
<!-- security datasource: -->
<bean id="myRealm" class="...">
    ...
</bean>
```

<a name="Spring-EnablingShiroAnnotations"></a>
##Enabling Shiro Annotations

In both standalone and web applications, you might want to use Shiro's Annotations for security checks (for example, `@RequiresRoles`, `@RequiresPermissions`, etc. This requires Shiro's Spring AOP integration to scan for the appropriate annotated classes and perform security logic as necessary.

Here is how to enable these annotations. Just add these two bean definitions to `applicationContext.xml`:

``` xml
<!-- Enable Shiro Annotations for Spring-configured beans.  Only run after -->
<!-- the lifecycleBeanProcessor has run: -->
<bean class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator" depends-on="lifecycleBeanPostProcessor"/>
    <property name="proxyTargetClass" value="true"/>
</bean>
<bean class="org.apache.shiro.spring.security.interceptor.AuthorizationAttributeSourceAdvisor">
    <property name="securityManager" ref="securityManager"/>
</bean>
```

<a name="Spring-SecureSpringRemoting"></a>
##Secure Spring Remoting

There are two parts to Shiro's Spring remoting support: Configuration for the client making the remoting call and configuration for the server receiving and processing the remoting call.

<a name="Spring-ServersideConfiguration"></a>
###Server-side Configuration

When a remote method invocation comes in to a Shiro-enabled server, the [Subject](subject.html "Subject") associated with that RPC call must be bound to the receiving thread for access during the thread's execution. This is done by defining Shiro's `SecureRemoteInvocationExecutor` bean in `applicationContext.xml`:

``` xml
<!-- Secure Spring remoting:  Ensure any Spring Remoting method invocations -->
<!-- can be associated with a Subject for security checks. -->
<bean id="secureRemoteInvocationExecutor" class="org.apache.shiro.spring.remoting.SecureRemoteInvocationExecutor">
    <property name="securityManager" ref="securityManager"/>
</bean>
```

Once you have defined this bean, you must plug it in to whatever remoting `Exporter` you are using to export/expose your services. `Exporter` implementations are defined according to the remoting mechanism/protocol in use. See Spring's [Remoting chapter](http://docs.spring.io/spring/docs/2.5.x/reference/remoting.html) on defining `Exporter` beans.

For example, if using HTTP-based remoting (notice the property reference to the `secureRemoteInvocationExecutor` bean):

``` xml
<bean name="/someService" class="org.springframework.remoting.httpinvoker.HttpInvokerServiceExporter">
    <property name="service" ref="someService"/>
    <property name="serviceInterface" value="com.pkg.service.SomeService"/>
    <property name="remoteInvocationExecutor" ref="secureRemoteInvocationExecutor"/>
</bean>
```

<a name="Spring-ClientsideConfiguration"></a>
###Client-side Configuration

When a remote call is being executed, the `Subject` identifying information must be attached to the remoting payload to let the server know who is making the call. If the client is a Spring-based client, that association is done via Shiro's `SecureRemoteInvocationFactory`:

``` xml
<bean id="secureRemoteInvocationFactory" class="org.apache.shiro.spring.remoting.SecureRemoteInvocationFactory"/>
```

Then after you've defined this bean, you need to plug it in to the protocol-specific Spring remoting `ProxyFactoryBean` you're using.

For example, if you were using HTTP-based remoting (notice the property reference to the `secureRemoteInvocationFactory` bean defined above):

``` xml
<bean id="someService" class="org.springframework.remoting.httpinvoker.HttpInvokerProxyFactoryBean">
    <property name="serviceUrl" value="http://host:port/remoting/someService"/>
    <property name="serviceInterface" value="com.pkg.service.SomeService"/>
    <property name="remoteInvocationFactory" ref="secureRemoteInvocationFactory"/>
</bean>
```

<a name="Spring-Lendahandwithdocumentation"></a>
##Lend a hand with documentation

While we hope this documentation helps you with the work you're doing with Apache Shiro, the community is improving and expanding the documentation all the time. If you'd like to help the Shiro project, please consider correcting, expanding, or adding documentation where you see a need. Every little bit of help you provide expands the community and in turn improves Shiro.

The easiest way to contribute your documentation is to send it to the [User Forum](http://shiro-user.582556.n2.nabble.com/) or the [User Mailing List](mailing-lists.html "Mailing Lists").
<input type="hidden" id="ghEditPage" value="spring.md"></input>
