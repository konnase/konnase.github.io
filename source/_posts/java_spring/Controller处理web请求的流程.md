---
title: Controller处理web请求的流程
p: java_spring/Controller处理web请求的流程
date: 2018-03-17 15:57:39
tags: [spring, controller, dispatcherservlet]
categories: java
typora-root-url: ../../../source
---

## Dispatcher Servlet是怎样被调用的？
我的[这篇博客](https://konnase.github.io/2017/07/24/Spring%E7%9B%B8%E5%85%B3%E7%9F%A5%E8%AF%86/)有讲到过程，再来分析一下Spring4.2.6的源代码

DispatcherServlet类继承关系是DispatcherServlet -> FrameworkServlet -> HttpServletBean -> HttpServlet，由DispatcherServlet负责处理Http请求。
另：Shiro里面的securitymanager的作用跟DispatcherServlet的作用很相似，都是负责全局组件交互的前端控制器。
<!--more-->
初始化过程如下：
首先，HttpServletBean方法执行初始化方法init()：
```java
/**
    * Map config parameters onto bean properties of this servlet, and
    * invoke subclass initialization.
    * @throws ServletException if bean properties are invalid (or required
    * properties are missing), or if subclass initialization fails.
    */
@Override
public final void init() throws ServletException {
    if (logger.isDebugEnabled()) {
        logger.debug("Initializing servlet '" + getServletName() + "'");
    }

    // Set bean properties from init parameters.
    try {
        PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
        BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
        ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
        bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
        initBeanWrapper(bw);
        bw.setPropertyValues(pvs, true);
    }
    catch (BeansException ex) {
        logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
        throw ex;
    }

    // Let subclasses do whatever initialization they like.
    initServletBean();

    if (logger.isDebugEnabled()) {
        logger.debug("Servlet '" + getServletName() + "' configured successfully");
    }
}
```
然后调用子类的initServletBean方法，这里的子类就是FrameworkServlet类，该方法如下：
```java
/**
    * Overridden method of {@link HttpServletBean}, invoked after any bean properties
    * have been set. Creates this servlet's WebApplicationContext.
    */
@Override
protected final void initServletBean() throws ServletException {
    getServletContext().log("Initializing Spring FrameworkServlet '" + getServletName() + "'");
    if (this.logger.isInfoEnabled()) {
        this.logger.info("FrameworkServlet '" + getServletName() + "': initialization started");
    }
    long startTime = System.currentTimeMillis();

    try {
        this.webApplicationContext = initWebApplicationContext();
        initFrameworkServlet();
    }
    catch (ServletException ex) {
        this.logger.error("Context initialization failed", ex);
        throw ex;
    }
    catch (RuntimeException ex) {
        this.logger.error("Context initialization failed", ex);
        throw ex;
    }

    if (this.logger.isInfoEnabled()) {
        long elapsedTime = System.currentTimeMillis() - startTime;
        this.logger.info("FrameworkServlet '" + getServletName() + "': initialization completed in " +
                elapsedTime + " ms");
    }
}
```
initServletBean方法的执行可在控制台看到输出，跟该方法给出的输出是一致的。在Springboot中，servlet是在完成各种bean的加载、jpa的初始化等操作后，在开始接受Http请求时初始化的：
![image](/img/frameworservletonstartup.png)
执行initWebApplicationContext方法，该方法调用onRefresh方法（由子类实现），即由DispatcherServlet类实现的onRefresh方法：
```java
/**
    * This implementation calls {@link #initStrategies}.
    */
@Override
protected void onRefresh(ApplicationContext context) {
    initStrategies(context);
}

/**
    * Initialize the strategy objects that this servlet uses.
    * <p>May be overridden in subclasses in order to initialize further strategy objects.
    */
protected void initStrategies(ApplicationContext context) {
    initMultipartResolver(context);
    initLocaleResolver(context);
    initThemeResolver(context);
    initHandlerMappings(context);
    initHandlerAdapters(context);
    initHandlerExceptionResolvers(context);
    initRequestToViewNameTranslator(context);
    initViewResolvers(context);
    initFlashMapManager(context);
}
```
initStrategies方法初始化很多策略，包括handlerMapping、handlerAdapter等，如果从bean容器中找不到相关策略，则采用缺省的策略，即spring-webmvc jar包的文件org.springframework.web.servlet.DispatcherServlet.properties中的缺省值。如下：
```xml
# Default implementation classes for DispatcherServlet's strategy interfaces.
# Used as fallback when no matching beans are found in the DispatcherServlet context.
# Not meant to be customized by application developers.

org.springframework.web.servlet.LocaleResolver=org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver

org.springframework.web.servlet.ThemeResolver=org.springframework.web.servlet.theme.FixedThemeResolver

org.springframework.web.servlet.HandlerMapping=org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,\
	org.springframework.web.servlet.mvc.annotation.DefaultAnnotationHandlerMapping

org.springframework.web.servlet.HandlerAdapter=org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter,\
	org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter,\
	org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerAdapter

org.springframework.web.servlet.HandlerExceptionResolver=org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerExceptionResolver,\
	org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver,\
	org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver

org.springframework.web.servlet.RequestToViewNameTranslator=org.springframework.web.servlet.view.DefaultRequestToViewNameTranslator

org.springframework.web.servlet.ViewResolver=org.springframework.web.servlet.view.InternalResourceViewResolver

org.springframework.web.servlet.FlashMapManager=org.springframework.web.servlet.support.SessionFlashMapManager
```

DispatcherServlet的初始化过程主要完成两件事。
1. 初始化Spring Web MVC使用的Web上下文，并且可能指定父容器为（ContextLoaderListener加载了根上下文）；
2. 初始化DispatcherServlet使用的策略，如HandlerMapping、HandlerAdapter等。

而在处理Http请求时，主要调用doService方法，该方法将DispatcherServlet的具体参数加入到request中，并委托doDispatch方法完成真正的调度，doDispatch方法实现了以下处理流程：
1. 用户发送请求–> DispatcherServlet，前端控制器收到请求后自己不进行处理，而是委托给其他解析器进行处理，作为统一访问点，进行全局流程控制。
2. DispatherServlet–> HandlerMapping，后者将把请求映射为HandlerExecutionChain对象（包含一个Handler处理器（页面控制器）对象、多个HandlerInterceptor拦截器），通过这种策略模式，很容易添加新的映射策略，得到一个handler
3. DispatcherServlet–> HandlerAdapter，后者将会把处理器包装为适配器，从而支持多种类型的处理器，由适配器执行处理器
4. HandlerAdapter–> 处理器功能处理方法的调用，HandlerAdapter将会根据适配的结果调用真正的处理器的功能处理方法，完成功能处理，并返回一个ModelAndView对象
5. ModelAndView的逻辑视图名–> ViewResolver，ViewResolver将把逻辑视图名解析成具体的View，通过这种策略，很容易更换其他视图技术
6. View–> 渲染，View会根据传进来的Model模型数据进行渲染，此处的Model实际是一个Map数据结构，因此很容易支持其他视图技术
7. 返回控制权给DIspatcherServlet，由DispatcherServlet返回响应给用户，流程结束
```java
//前端控制器分派方法  
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {  
        HttpServletRequest processedRequest = request;  
        HandlerExecutionChain mappedHandler = null;  
        int interceptorIndex = -1;  
  
        try {  
            ModelAndView mv;  
            boolean errorView = false;  
  
            try {  
                   //检查请求是否是multipart（如文件上传），如果是将通过MultipartResolver解析  
                processedRequest = checkMultipart(request);  
                   //步骤2、请求到处理器（页面控制器）的映射，通过HandlerMapping进行映射  
                mappedHandler = getHandler(processedRequest, false);  
                if (mappedHandler == null || mappedHandler.getHandler() == null) {  
                    noHandlerFound(processedRequest, response);  
                    return;  
                }  
                   //步骤3、处理器适配，即将我们的处理器包装成相应的适配器（从而支持多种类型的处理器）  
                HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());  
  
                  // 304 Not Modified缓存支持  
                //此处省略具体代码  
  
                // 执行处理器相关的拦截器的预处理（HandlerInterceptor.preHandle）  
                //此处省略具体代码  
  
                // 步骤4、由适配器执行处理器（调用处理器相应功能处理方法）  
                mv = ha.handle(processedRequest, response, mappedHandler.getHandler());  
  
                // Do we need view name translation?  
                // 在spring4.2.6中，这段代码被写入一个函数中
                if (mv != null && !mv.hasView()) {  
                    mv.setViewName(getDefaultViewName(request));  
                }  
  
                // 执行处理器相关的拦截器的后处理（HandlerInterceptor.postHandle）  
                //此处省略具体代码  
            }  
            catch (ModelAndViewDefiningException ex) {  
                logger.debug("ModelAndViewDefiningException encountered", ex);  
                mv = ex.getModelAndView();  
            }  
            catch (Exception ex) {  
                Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);  
                mv = processHandlerException(processedRequest, response, handler, ex);  
                errorView = (mv != null);  
            }  
  
            //步骤5 步骤6、解析视图并进行视图的渲染  
//步骤5 由ViewResolver解析View（viewResolver.resolveViewName(viewName, locale)）  
//步骤6 视图在渲染时会把Model传入（view.render(mv.getModelInternal(), request, response);）  
            if (mv != null && !mv.wasCleared()) {  
                render(mv, processedRequest, response);  
                if (errorView) {  
                    WebUtils.clearErrorRequestAttributes(request);  
                }  
            }  
            else {  
                if (logger.isDebugEnabled()) {  
                    logger.debug("Null ModelAndView returned to DispatcherServlet with name '" + getServletName() +  
                            "': assuming HandlerAdapter completed request handling");  
                }  
            }  
  
            // 执行处理器相关的拦截器的完成后处理（HandlerInterceptor.afterCompletion）  
            //此处省略具体代码  
  
  
        catch (Exception ex) {  
            // Trigger after-completion for thrown exception.  
            triggerAfterCompletion(mappedHandler, interceptorIndex, processedRequest, response, ex);  
            throw ex;  
        }  
        catch (Error err) {  
            ServletException ex = new NestedServletException("Handler processing failed", err);  
            // Trigger after-completion for thrown exception.  
            triggerAfterCompletion(mappedHandler, interceptorIndex, processedRequest, response, ex);  
            throw ex;  
        }  
  
        finally {  
            // Clean up any resources used by a multipart request.  
            if (processedRequest != request) {  
                cleanupMultipart(processedRequest);  
            }  
        }  
    }  
```

## 哪个类负责拦截Http请求并转发给Dispatcher Servlet的

本篇博客同时参考博客网站ITEYE上开涛的Spring MVC相关教程。

