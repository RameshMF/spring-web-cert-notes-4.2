#Basic MVC Configuration
###Dispatcher Servlet
- Implementation of Front Controller Pattern (see Patterns of Enterprise Application Architecture)
- Single point of entry - handle all incomming requests
- Orchestrates request handling by delegating requests to additional components (@Controllers, Views, View Resolvers, handler mappers, ...)
- Can map directly to /, but then exception for handling static resources needs to be configured

###Basic Request handling process
- Client sends a request to a specific URL
- Dispatcher Servlet receives the request
- DS passes request to a specific controller depending on the URL requested
- Controller returns LOGICAL view name and model to DS
- DS consults view resolvers until actual View is determined to render the output
- DS contacts the chosen view (e.g. Thymeleaf, Freemarker, JSP) with model data and it renders the output depending on the model data
- Rendered output is returned to the client as response

###Application Contexts
- Two application contexts are created: Root And Web
- Root Application Context: Web independent; Services, Repositories, ...
- Web Application Context: Contains only web layer specific beans - Controllers, Views, Wier Resolvers,...
- Web Application Context is a child context of Root Application Context
- If a bean is not found in a context, it is searched in parent context (but not vice versa)

**Root webapp context**
- Need to declare org.springframework.web.context.ContextLoaderListener or only DispatcherServlet's context will be created
- If contextConfigLocation is not declared, it defaults to WEB-INF\applicationContext.xml
- ContextLoaderListener - Bootstrap listener to start up and shut down Spring's root WebApplicationContext. Optional.
```xml
<context-param>
  <param-name>contextConfigLocation</param-name>
  <!--Defaults to WEB-INF\applicationContext.xml if not specified-->
  <param-value>classpath:app-config.xml</param-value>
</context-param>
<listener>
  <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```
**Child webapp context**  
If contextConfigLocation is not declared, it defaults to WEB-INF\dispatcher-servlet.xml
```xml
<servlet>
  <servlet-name>dispatcher</servlet-name>
  <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
  <init-param>
    <param-name>contextConfigLocation</param-name>
    <!--Defaults to WEB-INF\dispatcher-servlet.xml-->
    <param-value>classpath:mvc-config.xml</param-value>
  </init-param>
</servlet> 
```

###Minimal MVC Configuration
web.xml + WEB-INF/dispatcher-servlet.xml (for xml, annotation based alternative instead)

**web.xml**  
Declare DispatcherServlet as servlet and provide servlet-mapping
```xml
<servlet>
  <servlet-name>dispatcher</servlet-name>
  <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
</servlet>

<servlet-mapping>
  <servlet-name>dispatcher</servlet-name>
  <url-pattern>/*</url-pattern>
</servlet-mapping>
```
**WEB-INF/dispatcher-servlet.xml**  
In configuration file for Web Application Context declare a controller to serve requests (assuming controller returns directly content using @ResponseBody and no view resolving is required)  
```xml
<bean id="myController" class="com.example.MyController" />
```

###Configuration without web.xml 
- From Servlet 3.x web.xml is optional
- Can declare servlets and filters using annotations or implementing interface ServletContainerInitializer
- Needs Servlet 3.x compatible application server
- Servlet container looks for classes implementing ServletContainerInitializer (Spring provides SpringServletContainerInitializer)
- SpringServletContainerInitializer looks for classes implementing WebApplicationInitializer, which specify configuration instead of web.xml
- Spring provides two convenience implementations
    - AbstractContextLoaderInitializer - only Registers ContextLoaderListener
    - AbstractAnnotationConfigDispatcherServletInitializer - Registers ContextLoaderListener and defines Dispatcher Servlet, expects JavaConfig
    
###XML MVC namespace
- In xml config, `<mvc:>` namespace can be used to greatly simplify configuration compared to previous versions
- `<mvc:annotation-driven/>` to enable Default @MVC setup
    - @EnableWebMvc is equivalent is Java Configuration
    - Enables @Controller mappings using @RequestMapping
    - Enables features such as REST, validation, formatting, controller exception handling
    - From Spring 3.0+, not backwards compatible with Spring 2
    - Omit when maintaining Spring 2 backwards compatibility
- `<mvc:default-servlet-handler/>`
    - Needed If mapping Dispatcher Servlet in web.xml to / using servlet-mapping, so exceptions for static resources can be set using `<mvc:resources ... />`
    - `<mvc:resources mapping="/resources/**" location="classpath:/META-INF/web-content" cachePeriod="315569" />`
    - If mapped to /, static resources need special exception not to be handled by dispatcher servlet.
    - Configured in spring boot by default
-  Mapping urls to views without controllers
    - ```<mvc:view-controller path="/foo" view-name = "foo"/>```
    - ```<mvc:redirect-view-controller path="/old" redirect-url="/new" status-code="308" keep-query-params="true"/>```

###Java Config - @EnableWebMvc
- Equivalent of `<mvc:annotation-driven/>`
- On class level on @Configuration class
- Customisation - @Configuration class implements WebMvcConfigurer
    - Or for convenience extends WebMvcConfigurerAdapter, which implements given interface with empty methods
    - Methods - addInterceptors, addResourceHandlers (similar to `<mvc:resource>`), configureDefaultServletHandling, addControllers, configureContentNegotiation
- Not specified by Spring Boot, but similar config is; Should not be specified when using Boot; If specified boot skips mvc autoconfiguration and needs to be done manually
  
----------------
  
#MVC Components
###Model
- Is always created and passed to the view
- If a mapped controller method has Model as a method parameter, then a model instance is automatically injected by Spring to that method
- Any attributes set on injected model are preserved and passed to the View
```java
public String personDetail(Model model) {
  ...
  model.addAttribute("name", "Joe");
  ...
}
```
- Can be also added without attribute name `model.addAttribute(person);` Attribute name is then set depending on the added type name. E.g. Person type results in "person" name.

###View
- Template for rendering output to client based on Model data
- Display: JSP, Thymeleaf, Freemarker, Velocity,...
- Content: JSON, XML, PDF,...
- Implements View interface - defines which content type and how to render
```java
public interface View {
  String getContentType();
  void render(Map<String, ?> model, HttpServletRequest request, HttpServletResponse response) throws Exception;
}
```
- JstlView - forwards to JSP page, model attributes are passed to the JSP as `${attributeName}`
- Content generating view - unlike views which generate html and are declared and managed by Spring, these need to be declared manually
    - Create view class which extends specific parent - e.g. AstractPdfView and implement details of generating content manually
    - protected void buildPdfDocument(Map<String, Object> model, Document document, PdfWriter pdfWriter, HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse) throws Exception

###Controller
- POJO annotated @Controller
- Has methods which corresponds to specific urls, incl. variants for different HTTP methods (GET, POST), parameters provided etc.
- Methods are mapped to URLs usually using annotations @RequestMapping
- Controller fills Model with data, which is later provided to the View
- Controller can directly return data using @ResponseBody annotation
- Or can return a Logical view name as String. That is later resolved to a specific view using ViewResolvers
- If returns null or return type is void, the view name will be determined from URL requested
    - remove leading slash and remove extension
    - using default RequestToViewNameTranslator
- When specified as method parameters, certain objects can be automatically injected by spring to be used inside the controller methods
    - Model, HttpServletRequest, HttpServletResponse, Locale, Principal, HttpSession, HttpEntity<?>, TimeZone,...
- Can be well unit tested (is POJO) without container

----------------

#Mapping Controllers
###Handler Resolving process
1. Client sends a request to a specific url
2. Dispatcher servlet consults handler mappings, specific handler is returned based on the request - `getHandler(request)`
3. Handler is invoked through HandlerAdapter - `handle(request,response,handler)`  

Controller is a specific type of Handler.

###Handler Mapping
- RequestMappingHandlerMapping - enabled by default - Takes into account @RequestMapping on class and method level of @Controllers
- ControllerClassNameHandlerMapping - @RequestMapping on method level, but not on class level, class level is mapped from controller name
    - (PersonController → /person/*) + @RequestMapping("edit") → /person/edit
    - If request mapping not specified on method level, method name is taken instead -  edit() → /edit
- SimpleUrlHandlerMapping
    - Mapping is defined declaratively
```xml
<bean class="org.springframework.web.servlet.handler.SimpleUrlHandlerMapping">
  <property name= "mappings" >
    <value>
      /home=homeController
      /persons/**=personsController
    </value>
  </property>
</bean>
```
- Can have multiple chained HandlerMappings with specified order (same as view resolvers)
    - The first match wins
    - return of null means chain continues
- `<mvc:interceptors>` or `WebMvcConfigurer.addInterceptors()` adds interceptors for all mappings. To add interceptor to a specific mapping:
```xml
<bean class="org.springframework.web.servlet.mvc.support.ControllerClassNameHandlerMapping">
  <property name="interceptors" >
    <list>
      <bean class="com.example.FooInterceptor"/>
      <bean class="com.example.BarInterceptor"/>
    </list>
  </property>
</bean>
```
- Spring boot enables by default RequestMappingHandlerMapping, BeanNameUrlHandlerMapping, SimpleUrlHandlerMapping

###HandlerAdapter
- Adapter for invoking specific Handlers (one type of handler is @Controller)
- Has method handle(request,response,controller)
- Shields Dispatcher Servlet from specific types of Handlers
- WebflowHandlerAdapter, HttpRequestHandlerAdapter (remoting, resource handling,...)
- Spring Boot enables by default RequestMappingHandlerAdapter, HttpRequestHandlerAdapter, SimpleControllerHandlerAdapter
- RequestMappingHandlerAdapter
    - Adapts calls to @RequestMapping methods
    - Injects method params such as Model, HttpServletRequest, @PathVariables etc.
    - Interprets method return value - logical view name/ModelAndView/@ResponseBody/...
    - Supports `Optional<>` - for @RequestParam, @RequestHeader ("Pragma"),...
    - Enabled by default
    
   
----------------

#Controllers - Accessing Request Data
###@RequestMapping
- Can be on class level or method level of a Controller
- `@RequestMapping("/foo" )` - maps to /foo url, applies to all HTTP methods
- `@RequestMapping(value="/foo" ,method=RequestMethod.POST)` - Maps to /foo url, but only HTTP POST method
- `@RequestMapping(value="/foo", params={"bar"})` - Maps to /foo url but only when foo and bar params are included with any value (/foo?bar=something). Can provide multiple parameters.
- `@RequestMapping(value="/foo" , params={ "bar=baz"})` - maps to /foo url but only when bar parameter is provided with a value of 'baz'.
- Methods without requestMapping are ignored even when @RequestMapping is on a class level
- @RequestMapping("/foo") on class level and @RequestMapping("/bar") on a method level results in mapping to url /foo/bar

###@RequestParam
- Access to url params (/url?paramName=paramValue)
- Injected to controller methods as method parameters
- Does type conversion (also supports primitives)
- Can result in exception if not present or if type mismatch
- public String personDetail(@RequestParam("id") long id)
- Can be set as optional either by @RequestParam(value="id", required=false ) or by declaring type as Optional<?>
- If parameter name not declared, defaults to method's parameter name
    - @RequestParam long id → id param name (Java8+ or Debug Symbols enabled)

###@PathVariable
- Access to path segments (before ? in url)
- Used in REST
- eg. can extract person id=123 from /persons/123
```java
@RequestMapping("/persons/{id}" )
public String personDetail (@PathVariable ("id" ) long id) {...}
```
- Can use regex
- PathVariable and RequestParam can be formatted using @NumberFormat or @DataTimeFormat annotations
- If name not specified, it defaults to method param name
```java
@RequestMapping("/persons/{id}")
public String personDetail (@PathVariable long id) {...}
```

###Accessing request data
- If HttpServletRequest declared as controller method parameter, it is automatically injected by spring
    - Harder to mock and test, but spring provides MockHttpServletRequest
- Following annotations can be used on method parameters
    - Request data through SPEL and @Value annotation: @Value ("#{request.method}"), @Value ("#{request.requestURI}")
    - HTTP Headers: @RequestHeader ("user-agent")
    - Cookies: @CookieValue ("jsessionid")

###Intercepting Controller methods using HandlerInterceptor
- Good when handling cross-cutting concerns (security, logging,...)
- Interface HandlerInterceptor, which can define logic to be performed before controller execution, after it and after rendering output from the View
```java
boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception;`
void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception;
void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception;
```
- For convenience, there is HandlerInterceptorAdapter, which implements the interface with all methods empty
- If preHandle returns false, controller is not invoked
- Multiple interceptors allowed (interceptor chain)

**Java Configuration - in WebMvcConfigurerAdapter**  
```java
@Override
public void addInterceptors(InterceptorRegistry registry) {
  registry.addInterceptor(new FooInterceptor());
}
```  
**XML Configuration using `<mvc:interceptors>`**  
```xml
<mvc:interceptors>
  <bean class="com.example.FooInterceptor"/>
  <mvc:interceptor>
    <mvc:mapping path="/bar/*"/>
    <mvc:exclude-mapping path="/bar/exclude"/>
    <bean class="com.example.barInterceptor"/>
  </mvc:interceptor>
</mvc:interceptors>
```

----------------

#Resolving Views
###View Resolution Sequence
1. Controller returns logical view name to DispatcherServlet.
2. ViewResolvers are asked in sequence (based on their Order).
3. If ViewResolver matches the logical view name then returns which View should be used to render the output. If not, it returns null and the chain continues to the next ViewResolver.
4. Dispatcher Servlet passes the model to the Resolved View and it renders the output.

###ViewResolver
- Returns View to handle to output rendering based on Logical View Name (provided by the controller) and locale
- This way controller is not coupled to specific view technology (returns only logical view name)
- Default View resolver already configured is InternalResourceViewResolver, which is used to render JSPs (JstlView). Configures prefix and suffix to logical view name which then results to path to specific JSP.
```xml
<bean class= "org.springframework.web.servlet.view.InternalResourceViewResolver" >
  <property name= "prefix" value= "/WEB-INF/" />
  <property name ="suffix" value =".jsp" />
</bean>
```
###View Resolver Chain
- Beans of ViewResolver are discovered by Type and added to View Resolver Chain
- When a controller returns a logical view name, Dispatcher Servlet queries ViewResolvers in the chain depending on their Order. When first resolver returns View, chain does not continue
- Some resolvers can be only last (JSTL, JSON, XSLT,...), other anywhere in the chain (Tiles, Velocity, Freemarker,...) - they return null if view not resolved → chain continues
- Order can be set  

**In Java Config**  
```java
@Bean
public BeanNameViewResolver beanNameViewResolver() {
  BeanNameViewResolver resolver = new BeanNameViewResolver();
  resolver.setOrder(1);
  return resolver;
}
```  
**In XML**
```xml
<bean class="org.springframework.web.servlet.view.BeanNameViewResolver">
  <property name="order" value="1" />
</bean>
```  
**In XML using mvc namespace - order is determined by order of resolver elements in the tag**
```xml
<mvc:view-resolvers>
  <!--Order 0, BeanNameViewResolver-->
  <mvc:bean-name/>
  <!--Order 1, TilesViewResolver-->
  <mvc:tiles/>
</mvc:view-resolvers>
```

###Additional View Resolvers
- UrlBasedViewResolver
    -  Logical view name is resolved to a resource location
    -  Can use ":redirect" and ":forward" prefix
    -  Many implementations: InternalResourceViewResolver (default, JSP), FreeMarkerViewResolver, XsltViewResolver, VelocityViewResolver,...
- BeanNameViewResolver
    -  Logical view name is interpreted as a bean name (bean which implements View interface)
- XmlViewResolver
    - Similar to BeanNameViewResolver, but does not search for every bean, but just in a specific xml file

###Content Negotiation
- One resource can be rendered as different type to the client depending on the request
- Can be based on http header, file extension or http request parameter
- Can be achieved by having multiple controller methods, one for each content type - not recommended
- Can be achieved by having one controller method with branching logic depending or request details - not recommended

###ContentNegotiatingViewResolver

- Preferred way is to have a special view resolver to do the content negotiation logic - ContentNegotiatingViewResolver
- CNVR delegates to other view resolvers
    - View interface has getContentType() method, which returns content type the view produces (JstlView has text/html)
    - After delegated resolver returns view, CNVR checks whether its content type (by calling getContentType()) is what was requested by the client
    - If so, the view is returned to the dispatcher servlet, otherwise next view resolver is called by the CNVR
- CNVR must be the first
- CNVR can have following properties configured
    - order - same as other view resolvers, must be first
    - viewResolvers - can specify to which view resolvers will delegate to; by default to all of them
    - contentNegotiationManager - if not specified, default ContentNegotiationManager will be used
    - useNotAcceptableStatusCode - if true, will return HTTP 406 when view not resolved, otherwise the view resolver chain will just continue

###ContentNegotiationManager
- The ContentNegotiatingViewResolver handles delegation to other view resolvers and checking whether they are able to provide desired content type. ContentNegotiationManager decides what the desired type is based on the client’s request.
- @EnableWebMvc or `<mvc:annotation-driven />` creates default ContentNegotiationManager
- Accept header usually not used as browsers by default always send text/html
- If match is not found, default content the can be specified using defaultContentType property
- Mapping of format (html) to mime-type (text/html) is done either by JAF - Java Activation Framework (by default; can be disabled by useJaf=false) or by specifying a map of format→mime-type pairs using mediaTypes

**Content type resolution process order**
   1. Check if there is an extension in url (.json); favorPathExtension=true
   2. Check if there is url parameter format (?format=json); favorParameter=true; "format" parameter name can be changed using parameterName property
   3. Check if there is a HTTP Accept header (Accept: application/json); ignoreAcceptHeader=false


----------------

#Composite Views - Apache Tiles
###Apache Tiles
- Tempting framework for composite views
- Used to be part of Struts 1
- Implementation of Composite View Pattern (see Patterns of Enterprise Application Architecture)
- Tiles 1 not Supported, Tiles 2 since Spring 2.5, Tiles 3 since Spring 3.2
- Tiles definition in xml file - tiles.xml
- Each definition represents a view
```xml
<tiles-definitions>
  <definition name="base" template="/WEB-INF/tiles/main.jsp" />
  <definition name="foo" extends="base">
    <put-attribute name="title" value="Foo" />
  </definition>
  <definition name="bar" extends="base">
    <put-attribute name="title" value="Bar" />
  </definition>
</tiles-definitions>
```
- In JSP, attributes defined in tiles.xml using putAttribute are rendered using `<tiles:insertAttribute name="title"/>`
- Can use wildcards
- name foo/bar → {1}/{2} → title = foo, content = bar
```xml
<definition name="*/*" extends="base">
  <put-attribute name="title" value="{1}" />
  <put-attribute name="content" value="{2}" />
</definition>
```
- Tiles attributes can be either String, template (if starts with /) or another tiles definition (if definition name matches)
- A definition can inherit from other (using 'extends=parentDefinitionName')

###Setting up Tiles in Spring

- Set Up TilesConfigurer
- Add TilesViewResolver instead of default InternalResourceViewResolver
- Create tiles.xml with tile definitions
- Create JSP templates used for tile rendering

###TilesConfigurer configuration

**Java Configuration**  
```java
@Bean
public TilesConfigurer tilesConfigurer() {
  TilesConfigurer tilesConfigurer = new TilesConfigurer();
  tilesConfigurer.setDefinitions("/WEB-INF/tiles.xml", "/WEB-INF/foo/tiles.xml");
  tilesConfigurer.setCheckRefresh(true); // default false
  tilesConfigurer.setValidateDefinitions(true);
  return tilesConfigurer;
}
```
**XML Configuration (Tiles 3+)**  
- If no definition locations specified, defaults to /WEB-INF/tiles.xml
```xml
<mvc:tiles-configurer id="tilesConfigurer" check-refresh="true" validate-definitions ="true">
  <mvc:definitions location="/WEB-INF/tiles.xml"/>
  <mvc:definitions location="/WEB-INF/foo/tiles.xml"/>
</mvc:tiles-configurer>
```
**XML Configuration (Tiles 2)**   
- manually by declaring bean of type TilesConfigurer in xml (Tiles 2) 

###TilesViewResolver configuration

**XML Configuration - using `<mvc:>` namespace**  
```xml
<mvc:view-resolvers>
  <mvc:tiles/>
</mvc:view-resolvers>
```
**XML Configuration - specifically declaring as a bean**  
```xml
<bean class = "org.springframework.web.servlet.view.tiles3.TilesViewResolver"/>
```
**Java Configuration**  
```java
@Bean
public TilesViewResolver tilesViewResolver() {
  return new TilesViewResolver();
}
```

----------------

#Resources
###Url Fingerprinting & Cache Busting

- Static resources (JS, CSS,…) caching with long periods (like a year)
- When resource changes, cache needs to be invalidated (busted)
- Each resource url has added "fingerprint" - String like version number or hash of contents of the file
- When resource changes, the fingerprint changes as well → url changes and browser considers it a new resource


###Resource Resolvers

- Spring can define chained Resource Handlers, which from given path resolve specific resource
- If a resolver does not resolve, the next one in the chain gets the chance
- Handler can have attacher Resource Resolver, which find the actual resource and can also alter resource URL
- Handler can also have ResourceTransformer, which can alter contents of the resource
- Resolver can point to compressed version of resource, add fingerprint to the url,...
- GzipResourceResolver can be used to server compressed versions of resources
- To enable resource versions in URL
    - ResourceUrlProviderExposingInterceptor filter must be declared as a servlet filter in web.xml
    - JSP, there is special tag for urls: `<script src="<spring:url value="/resources/foo.js"/>"></script>`, which resolves the url to its versioned variant  

**Resource Handler and resolver configuration in Java Config - WebMvcConfigurerAdapter**  
```java
@Value("${app.version}")
protected String version;

@Override
public void addResourceHandlers(ResourceHandlerRegistry registry) {
  ResourceResolver resolver = new VersionResourceResolver();
  resolver.addFixedVersionStrategy(version, "/**/*.js") //Resources matching will have version specified in version variable
          .addContentVersionStrategy("/**")); //Resources matching will have version based on their content hash

  registry.addResourceHandler("/resources/**")
          .addResourceLocations("classpath:/META-INF/web-content/")
          .resourceChain(true)
          .addResolver(resolver);
}
```

**XML config using `<mvc:resources>`**
```java
<mvc:resources mapping="/resources/**" cachePeriod="31556926" location="/, classpath:/META-INF/web-content/">
  <mvc:resource-chain resource-cache="true">
    <mvc:resolvers>
      <mvc:version-resolver>
        <mvc:fixed-version-strategy version="${app.version}" patterns="/**/*.js"/>
        <mvc:content-version-strategy patterns="/**"/>
      </mvc:version-resolver>
    </mvc:resolvers>
  </mvc:resource-chain>
</mvc:resources>
```
###MessageSource

- Externalisation of messages, for internationalization
- Only one bean per applicationContext named messageSource (bean is discovered by name messageSource)
- Using Java's ResourceBundle
- Can use either ResourceBundleMessageSource or ReloadableResourceBundleMessageSource (allows changing properties files and  automatically reloading changes)  

**XML config**  
basename foo resolves to foo resource bundle (foo.properties, foo_en.properties, foo_fr.properties,...)
```java
<bean id="messageSource" class="org.springframework.context.support.ResourceBundleMessageSource">
  <property name="basenames">
    <list>
      <value>classpath:messages/foo<value/>
    </list>
  </property>
</bean>
```

**Java Config**  
```java
@Bean
MessageSource messageSource() {
  ResourceBundleMessageSource source = new ResourceBundleMessageSource();
  source.setBasename("classpath:messages/foo");
  return source;
}
```

###Retrieving messages

- Inject MessageSource in the class
- Then `messageSource.getMessage(code, args, defaultMessage, locale);`
    - args can be used to fill placeholders in the message
- JstlView supports displaying messages from messageSource using
```xml
<%@ taglib prefix="fmt" uri="http://java.sun.com/jsp/jstl/fmt" %>
<fmt:message key="person.firstname" />
```
               
----------------

#MVC Forms - Basics

###Basic Workflow

1. Form page fetched through HTTP GET
2. Form is submitted through HTTP POST
3. Server-Side form validation, save data in form object
4. Successful Submit results in POST-Redirect-GET to avoid re-POST (Search forms do not follow this - submitted with GET instead of post, no P-R-G as repost is not an issue)

###Redirects

- After POST, on success, redirect (HTTP 302) should be performed to a new resource obtained by GET to prevent resubmit
- Controller method adds "redirect:" prefix to logical view name
- It is a new request, all data is lost, if something needs to be passed to redirected page, there are two options  

**Pass as url parameters in newly requested resource (/success?firstName=John&lastName=Doe)**  
```java
@RequestMapping(method = RequestMethod.POST)
public String editPerson(@ModelAttribute("personAttribute") Person person, RedirectAttributes attributes) {
  attributes.addAttribute("firstName", person.getFirstName());
  attributes.addAttribute("lastName", person.getLastName());
  return "redirect:success";
}
```

**Store params in flash scope on server - Better for complex objects, data not visible on client in the url**  
```java
@RequestMapping(method = RequestMethod.POST)
public String editPerson(@ModelAttribute("personAttribute") Person person, RedirectAttributes attributes) {
  attributes.addFlashAttribute("firstName", person.getFirstName());
  attributes.addFlashAttribute("lastName", person.getLastName());
  return "redirect:success";
}
```
###Form Object

- Could be domain object directly but has several disadvantages
    - Security concerns
    - Limitations - default constructor, getters and setters
    - Presentation layer logic leaks to Domain object (formatting, ...)
- Better - dedicated form object
    - Specific to presentation layer
    - Just what specific form needs, nothing extra
    - Not always is form direct representation of domain object
    - Contains formatting and validation, type conversion to domain object

###Managing form object
Form needs to be accessed across multiple requests (initial GET, then POST, again on submit error,…), possible approaches are:

**Create new instance every request**  
- Good when form represents creation of a new object
```java
@RequestMapping(method = RequestMethod.GET)
public String getPerson(Model model) {
  model.addAttribute("person", new Person());
  return "editPerson";
}
```

**Retrieve on every request using @ModelAttribute**  
- Good when form represents existing object
- In PUT method, Person is injected from @ModelAttribute method and overridden with data sent in form from client

```java
@ModelAttribute
public Person addToModel(@PathVariable String personId) {
  return personService.get(personId);
}

@RequestMapping(method = RequestMethod.GET)
public String getPerson() {
  return "editPerson";
}

@RequestMapping(method = RequestMethod.PUT)
public String editPerson(Person person) {
  return "redirect:success";
}
```

**Retrieve on every request using  @SessionAttributes ("person")**  
- May not scale well (uses session)

```java
@Controller
@RequestMapping(path = "edit")
@SessionAttributes("person")//"person" model attribute should be stored in session
public class PersonController {
  
  @RequestMapping(method = RequestMethod.GET)
  public String getPerson(@PathVariable String personId, Model model) {
    model.addAttribute("person", personService.get(personId));//put to model AND to session
    return "editPerson";
  }

  @RequestMapping(method = RequestMethod.PUT)
  public String editPerson(Person person, SessionStatus sessionStatus) {
    personService.update(person);
    sessionStatus.setComplete();//"person" can be removed from the session
    return "redirect:success";
  }
}
```

###JSP form support

- Use Spring’s custom tag library for forms `<%@ taglib prefix="form" uri="http://www.springframework.org/tags/form" %>`
- Instead of regular html `<form>` element use `<form:form>`
- Use `<form:input>` instead of regular `<input>` tag

###JSP - Form

- modelAtribute links to attribute in model with "person" name - Form Object. Form object is POJO.
- When form is rendered for the first time, the form is pre-filled with data from modelAttribute
- `<form:errors>` shows error messages
```xml
<form:form name="personForm" action="..." method="post" modelAttribute="person">
  ...
</form:form>
```

###JSP - Input

- Types - color, date, datetime, datetime-local, email, month, number,range, search, tel, time, url, week
- Additional tags - checkbox, checkboxes, hidden, label, password, radiobutton, radiobuttons, textarea
```xml
<form:form modelAttribute="person" ...>
  <!--maps to person.firstName-->
  <form:input path="firstName" />
  <form:input path="favoriteColor" type="color" />
</form:form>
```

###JSP - Select

**Fixed values**  
```xml
<form:select path="gender">
  <form:option label="Male" value="M"/>
  <form:option label="Female" value="F"/>
<form:select/>
```

**Collection**  
```xml
<form:select path="contactPersonId" items="${contacts}" itemLabel="lastName" itemValue="id" />
```

**Map -  map key is interpreted as a select item value and map value as a select item label**  
```xml
<form:select path="contactPersonId" items="${contacts}"/>
```

**Combining static and dynamic items**  
```xml
<form:select path="contactPersonId">
  <form:option value="None">None</form:option>
  <form:options items="${contacts}" itemLabel="lastName" itemValue="id" />
<form:select/>
```

----------------

#MVC FORMS - Binding, Validation & Formatting
###Data Binding - View

- Form tags automatically generate id and name attributes same as path
- Following Form’s fields are bound to attribute "person" from model
- Form input fields are bound to fields of "person" by "path" attribute (path="firstName" → person.firstName)
- When form is rendered to the client for the first time, field values are pre-filled from the Model
```xml
<form:form modelAttribute="person" action="edit" method="post">
  <form:label path="firstName">First Name</form:label>
  <form:input path="firstName"/>
  <form:label path="lastName">Last Name</form:label>
  <form:input path="lastName"/>
  <form:button>Submit</form:button>
</form:form>
```

###Data Binding - Controller

In the GET method,fetching of view containing form is handled
- Controller can provide model data, which will be pre-filled in the form once it is rendered
- "person" attribute name added to model matches modelAttribute="person" in jsp form
```java
@RequestMapping(method = RequestMethod.GET)
public String getPerson(Model model) {
  model.addAttribute("person", new Person("John", "Doe"));
  return "edit";
}
```
In the POST method, submit of form is handled
- Person is automatically injected by spring if declared as method param and through @ModelAttribute
- Is available as "personAttribute" variable in rendered tsp - here "done"
- If annotation not provided, variable name is based on type - Person type → "person" variable name
- When injecting method param Person, Spring looks for "person" attribute in model, if not found, new Person is created and its fields are populated from request params
```java
@RequestMapping(method = RequestMethod.POST)
public String editPerson(@ModelAttribute("person") Person person) {
  return "done";
}
```
Can specify binding configuration - @InitBinder annotation either directly in Controller or ControllerAdvice
- Required fields
- Validators
- Allowed fields - Either whitelist or blacklist; Whitelist preferred as it is safer
```java
@InitBinder
public void initBinder(WebDataBinder binder) {
  binder.setAllowedFields( "firstName", "lastName");
}

@InitBinder
public void initBinder(WebDataBinder binder) {
  binder.setDisallowedFields( "id", "*Id"); //can use wildcards
}
```

###Binding Errors

- In controller method params, immediately after injected Form Object, BindingResult can be declared
- `public String editPerson(@ModelAttribute("personAttribute") Person person, BindingResult bindingResult)`
- bindingResult.hasErrors() can be checked, if so → return form view again for user to correct errors
- Display errors on JSP using `<form:errors path="firstName" />`  for specific field or `<form:errors path="*" />` for all
- Error messages are automatically taken from messageSource
```
typeMismatch - Any error of "type mismatch" during binding
typeMismatch.firstName - type mismatch for specific field name
typeMismatch.person.amount - type mismatch for specific form object and field
typeMismatch.com.example.SerialNumber  - type mismatch for specific target type
```
Additional error message types
- required - required fields can be set in @InitBinder method binder.setRequiredFields( "firstName", "lastName" )
- methodInvocation - if getter or setter on form fails

###Validation

Spring supports JSR-303 Bean Validation
- Hibernate is reference implementation
- Annotations on fields
- @NotNull, @NotEmpty (not null and not empty collection or string; Hibernate specific), @Min, @Max, @Pattern (regex), @Size (collection size or string length)
```java
@Size(min=3 , max= 20)
private String firstName;
```

If form Object as Controller method param is marked as @Valid, form is validated and validation errors are added to BindingResult with regular binding errors, which can be displayed by <form:errors>
```java
public String editPerson(@Valid Person person, BindingResult bindingResult)
```
- `<mvc:annotation-driven>` or `@EnableWebMvc` enables JSR-303 validation globally (needs JSR-303 implementation on classpath)
- Can define error messages in MessageSource
```
NotNull - General for any validation failure for given annotation
NotNull.firstName - For any validation failure for given annotation and field
NotNull.person.firstName - For any validation failure for given annotation, Form Object and its field
NotNull.com.example.SerialNumber - For any validation failure for given Type
```
- Can define custom JSR-303 annotations
- Alternative is custom Validator implementing `org.springframework.validation.Validator`
```java
public interface Validator {
boolean supports(Class<?> clazz);
  void validate(Object target, Errors errors);
}
```
- Validation failed using errors.rejectValue(String field, String errorCode); - error code interpretable as a message key.
- Register validator using @InitBinder in controller
```java
@InitBinder
public void initBinder(WebDataBinder binder) {
  binder.setValidator(new FooValidator());
}
```
###Formatters

- Format data from Form fields (String usually) to Form Object and vice versa
- `<mvc:annotation-driven>` or `@EnableWebMvc` already registers default set of formatters

**Formatting using  Annotations**  
    - Can be either directly on field of Form Object or on Controller method parameters
```java
@DateTimeFormat(iso=ISO.DATE)
@NumberFormat(style=Style.CURRENCY)
```

**Formatting using FMT tags in JSP**  
- `<fmt:formatNumber value="${account.interestAmount}" type="percent”>`
- `<fmt:formatDate value=“${person.dateOfBirth}" pattern="MM/dd/yyyy" />`

**Formatting using formatters implementing Formatter Interface**  
Implement Formatter interface
```java
class DurationFormatter implements Formatter<Duration> {
  public Duration parse(String text, Locale locale) throws ParseException {
    return Duration.parse(text);
  }

  public String print(Duration object, Locale locale) {
    return object.toString();
  }
}
```
Then registry in WebMvcConfigurerAdapter
```java
@Override
public void addFormatters(FormatterRegistry registry) {
  registry.addFormatter(new DurationFormatter());
}
```


----------------

#Exceptions
###Exception Handling
@ResponseStatus on an exception class will result in given HTTP status code being return when annotated exception is thrown
```java
@ResponseStatus(HttpStatus.NOT_FOUND)
public class PersonNotFoundException { … }
```

If exception in third party library or additional logic needs to be performed, approach above cannot be used
- Need to have method annotated @ExceptionHandler(ExceptionClass.class) in @Controller
- Can combine with @ResponseStatus
```java
@ResponseStatus(HttpStatus.NOT_FOUND)
@ExceptionHandler({PersonNotFoundException.class})
public void handlePersonNotFound() {
  ...
}
```
- Cannot declare Model as method parameter, but most others can be - HttpServletRequest, HttpServletResponse,...
- Can return error view or content directly or status code with empty body
- Exception caught can be injected into method, only exceptions of type declared in method type are passed to method

```java
@ExceptionHandler
public String handleException(IOException e) {…}
```

- Can define multiple exceptions `@ExceptionHandler({FooException.class, BarException.class})`
- Additional way to map URL to static codes directly

**XML Config**  
```xml
<mvc:status-controller path="/notfound" status-code="404" />
```

**Java Config**  
```java
class Config extends WebMvcConfigurerAdapter {
  @Override
  public void addViewControllers(ViewControllerRegistry registry) {
    registry.addStatusController("/notfound", HttpStatus.NOT_FOUND);
  }
}
```

###Controller Advice

- @ControllerAdvice - Annotation on class level, which now acts like an Interceptor
- Applies to any request on any @Controller, can be narrowed down to specific controllers using
    - `@ControllerAdvice(assignableTypes={FooController.class, BarController.class})` - Only for specific controllers
    - `@ControllerAdvice(annotations={RestController.class})` - Only for specific annotations
    - `@ControllerAdvice(basePackages={"com.example"})` - Only for specific packages
- @ModelAttribute in @ControllerAdvice - Add data to model for any request
- @InitBinder in @ControllerAdvice - Set up binding for any request
- @ExceptionHandler - Global exception handling for any request
- To enable a specific class annotated by @ControllerAdvice, it is just enough for it to be a Spring managed bean - either `<bean>` in xml, @Bean in java config or discovered by component scan
- @ControllerAdvice can implement `ResponseBodyAdvice<>`
    - Has method supports() and beforeBodyWrite()
    - AbstractJsonpResponseBodyAdvice - subclass to write JSONP as a response

###HandlerExceptionResolver

- Interface to be implemented by classes resolving exceptions from handlers (@Controller is a type of handler)
- Can be chained, is ordered, if returns null chain continues
- Prepares model and view to be rendered on error
- By default enabled resolvers - DefaultHandlerExceptionResolver, ResponseStatusExceptionResolver
```java
public interface HandlerExceptionResolver {
  ModelAndView resolveException(HttpServletRequest req, HttpServletResponse resp Object handler, Exception e);
}
```

###DefaultHandlerExceptionResolver

- Converts the most common Spring exceptions to HTTP Status codes
    - NoSuchRequestHandlingMethodException → 404
    - HttpRequestMethodNotSupportedException → 405
    - ...
- ResponseStatusExceptionResolver → supports @ResponseStatus on Exception classes
- ExceptionHandlerExceptionResolver → supports @ExceptionHandler on @Controller and @ControllerAdvice methods

###SimpleMappingExceptionResolver

- HandlerExceptionResolver implementation
- maps exception class names to view names
- Adds exception object to model as an attribute  'exception'
- Can define default error view and default HTTP status code
- Can be extended, easier than custom implementation of HandlerExceptionResolver
- Does not log exceptions by default, can be enabled in children by overriding methods from AbstractHandlerExceptionResolver  

**XML Configuration**  
```xml
<bean class="org.springframework.web.servlet.handler.SimpleMappingExceptionResolver">
  <property name="defaultErrorView" value="error" />
  <property name="exceptionMappings">
    <map>
      <entry key="IllegalArgumentException" value="illegalArgument" />
    </map>
  </property>
  <property name="defaultStatusCode" value="500"/>
</bean>
```

**Java Configuration**  
```java
@Bean
public SimpleMappingExceptionResolver simpleMappingExceptionResolver() {
  SimpleMappingExceptionResolver resolver = new SimpleMappingExceptionResolver();
  resolver.setDefaultErrorView("error");
  resolver.setDefaultStatusCode(500);
  Properties exceptionMappings = new Properties();
  exceptionMappigs.put("IllegalArgumentException", "illegalArgument");
  resolver.setExceptionMappings(exceptionMappigs);

  return resolver;
}
```

###ResponseEntityExceptionHandler

- Alternative to DefaultHandlerExceptionResolver suitable for REST
- Similar in functionality, but instead of ModelAndView returns ResponseEntity (more suitable for REST as instead of error view, content should be returned directly)
- Extend to customize

###Spring Boot Exception Handling

- Any unhanded exceptions are watched and redirected to internal controller, which returns /error view
- Can define custom error page
- Whitelabel Error Page is spring’s default
- Can be disabled using `error.whitelabel.enabled=false`


----------------


#REST

- Representational State Transfer
- Architectural style
- Stateless (clients maintains state, not server), scalable; → do not use HTTP session
- Usually over http, but not necessarily
- Entities (e.g. Person) are resources represented by URIs
- HTTP methods (GET, POST, PUL, DELETE) are actions performed on resource (like CRUD)
- Resource can have multiple representations (different content type)
- Request specifies desired representation using HTTP Accept header, extension in url (.json) or parameter in url (format=json)
- Response states delivered representation type using Content-Type HTTP header

###HATEOAS

- Hypermedia As The Engine of Application State
- Response contains links to other items and actions → can change behavior without changing client
- Decoupling of client and server

###RestTemplate

- Can be used to simplify HTTP calls to RESTful api
- Message converters supported - Jackson, GSON
- Automatic input/output conversion - using HttpMessageConverters
    - StringHttpMessageConvertor
    - MarshallingHttpMessageConvertor
    - MappingJackson2XmlHttpMessageConverter
    - GsonHttpMessageConverter
    - RssChannelHttpMessageConverter
- AsyncRestTemplate - similar to regular one, allows asynchronous calls, returns ListenableFuture

###JAX-RS

- Java API for RESTful web services
- Part of Java EE6
- Jersey is reference implementation  

```java
@Path("/persons/{id}")
public class PersonService {
  @GET
  public Person getPerson(@PathParam("id") String id) {
    return findPerson(id);
  }
}
```

###Spring Rest Support

- @ResponseStatus can set HTTP response status code
    - If used, void return type means no View (empty response body) and not default view!
    - 2** - sucess (201 Created, 204 No Content,...)
    - 3** - redirect
    - 4** - client error (404 Not found, 405 Method Not Allowed, 409 Conflict,...)
    - 5** - server error
- @ResponseBody before controller method return type means that the response should be directly rendered to client and not evaluated as a logical view name
    -  Uses converters to convert return type of controller method to requested content type
- @RequestHeader can inject value from HTTP request header as a method parameter
- @RequestBody - Injects body of HTTP request, uses converters to convert request data based on content type
    - public void updatePerson(@RequestBody Person person, @PathVariable("id") int personId)
    - Person can be converted from JSON, XML, ...
- HttpEntity<>
    - Controller can return HttpEntity instead of view name - and directly set response body, HTTP response headers
    - `return new HttpEntity<ReturnType>(responseBody, responseHeaders);`
- HttpMessageConverter
    - Converts HTTP response (annotated by @ResponseBody) and request body data (annotated by @RequestBody)
    - `@EnableWebMvc` or `<mvc:annotation-driven/>` enables it
    - XML, Forms, RSS, JSON,...
- @RequestMapping can have attributes produces and consumes to specify input and output content type
    - `@RequestMapping (value= "/person/{id}", method=RequestMethod. GET, produces = {"application/json"})`
    - `@RequestMapping (value= "/person/{id}", method=RequestMethod. POST, consumes = { "application/json" })`
- Content Type Negotiation can be used both for views and REST endpoints
- @RestController - Equivalent to @Controller, where each method is annotated by @ResponseBody
- By default, html forms support only POST and GET as methods
    - spring's JSP taglib `<form:form>` supports all (by including hidden field for methods used)
    - `<form:form method= "put" ...>`
    
    
----------------
    
    
#AJAX
###AJAX basics

- Asynchronous Javascript And XML
- Usually JSON instead of XML
- Spring has no JavaScript integration by default
- Instead of full page reload, only part of the page is asynchronously updated
- Using XMLHttpRequest object  

```javascript
function performAjaxCall() {
  var xhr = new XMLHttpRequest();
  
  xhr.onreadystatechange = function() {
    if (this.readystate == 4) {
      //handle response here
    }
  }
  
  xhr.open('GET', 'targetUrl');
  xhr.send();
}
```
###jQuery

- Syntax: $(selector).action();
    - $ means jQuery
    - Selector query selects one or more html elements
    - Action is a method to be performed on selected elements
- Selectors
    - `#header` - element with id="header"
    - `.label` - element with class="label"
    - `a` - elements of type <a>
    - `ul li` - all `<li>` elements inside of `<ul>` element
- Basic functions
    - `val()` gets value of an element
    - `val("foo")` sets value "foo" to an element
    - `html()` gets html content of an element
    - `html("<br>")` sets html content of an element
    - `append("<br>")` appends html to current html content of and element
    - `empty()` removes html content of an element
    - `hide()` hides an element
    - `show()` shows a hidden element
    - `fadeIn(), fadeOut()` animations when showing/hiding
    - `$(function() { … });`   executes on page load
    - `on('event', function() { … })` executes function on a specific event - e.g. 'click'
- Call Ajax using jQuery
    - `$.get(url, parameters, callback, type)`
    - `$.getJSON(url, parameters, callback)`

###CORS

- Browsers implement Same-Origin policy - ajax call allowed only from same origin - domain and port
- Measure to prevent Cross Site Scripting attacks
- CORS = Cross-Origin Resource Sharing, enables cross domain ajax in browsers
- Useful e.g. for calling remote REST api on different domain

**CORS Flow**  
1. A cross-domain ajax request from javascript is initiated from my-site.com to other-site.com  
2. Browsers sends OPTIONS http request to other-site.com with header Origin: my-site.com  
3. other-site.com responds with http header Access-Control-Allow-Origin with lists of allowed sites (or * wildcard for all) or may refuse to serve response to provided origin  
4. Browser will not allow the request when server does not provide Access-Control-Allow-Origin header or if the sites mentioned do not match the origin  

**@CrossOrigin**  
- enables CORS for a controller
- Both on class and/or method level
- @CrossOrigin means all origins are allowed
- Can restrict to specific origins with @CrossOrigin(origins="http://safe-site.com")  
    
**Enable CORS in XML - Application wide** 
```xml
<mvc:cors>
  <mvc:mapping path="/api/**"
       allowed-origins="http://domain1.com, http://domain2.com"
       allowed-methods="GET, PUT"
       allowed-headers="header1, header2, header3"/>
  <mvc:mapping path="/resources/**" allowed-origins="http://domain1.com" />
</mvc:cors>
```

**Enable CORS in Java Config - Application wide**
```java
@Configuration
@EnableWebMvc
public class WebConfig extends WebMvcConfigurerAdapter {
  @Override
  public void addCorsMappings(CorsRegistry registry) {
    registry.addMapping("/api/**")
            .allowedOrigins("http://domain2.com")
            .allowedMethods("PUT", "DELETE")
            .allowedHeaders("header1", "header2", "header3");
  }
}
```