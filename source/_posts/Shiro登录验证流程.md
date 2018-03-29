---
title: Shiro登录验证流程
date: 2018-03-19 15:14:13
tags: [shiro, authentication]
---

# Shiro登录验证的实现和验证流程

## 实现登录验证

### 核心类

1. 需要ShiroConfiguration：在这个类中主要是注入shiro的filterFactoryBean和securityManager等对象。

2. 需要StatelessAccessControlFilter：这个类中实现访问控制过滤，当我们访问url的时候，这个类中的两个方法会进行拦截处理。

3. 需要StatelessAuthorizingRealm：这个类中主要是身份认证，验证信息是否合理，是否有角色和权限信息。

4. 需要StatelessAuthenticationToken：在shiro中有一个我们常用的UsernamePasswordToken，因为我们需要这里需要自定义一些属性值，比如：消息摘要，参数Map。

5. 需要StatelessDefaultSubjectFactory：由于我们编写的是无状态的，每人情况是会创建session对象的，那么我们需要修改createSubject关闭session的创建。

6. 需要HmacSHA256Utils：Java 加密解密之消息摘要算法，对我们的参数信息进行处理。

<!--more-->

### 配置文件ShiroConfig

本篇博客以sinosteel代码分析。以上用到的核心类在com.sinosteel.framework.config.shiro包下面实现了。首先看com.sinosteel.framework.config.shiro.ShiroConfig类，其中ShiroFilterFactoryBean和DefaultWebSecurityManager这两项是在Spring中引入Shiro的基本配置，前者用于配置过滤器的过滤规则，后者用于生成全局的securitymanager。该类将核心类都注册到IoC容器中。：
```java
@Configuration
public class ShiroConfig 
{ 
    /** 
     * ShiroFilterFactoryBean 处理拦截资源文件问题。 
     * 注意：单独一个ShiroFilterFactoryBean配置是或报错的，以为在 
     * 初始化ShiroFilterFactoryBean的时候需要注入：SecurityManager 
     * 
        Filter Chain定义说明 
       1、一个URL可以配置多个Filter，使用逗号分隔 
       2、当设置多个过滤器时，全部验证通过，才视为通过 
       3、部分过滤器可指定参数，如perms，roles 
     * 
     */  
    @Bean
    public ShiroFilterFactoryBean shiroFilter(SecurityManager securityManager)
    {
    	ShiroFilterFactoryBean factoryBean = new ShiroFilterFactoryBean();
    	factoryBean.setSecurityManager(securityManager);

        Map<String,String> filterChainDefinitionMap = new LinkedHashMap<String, String>();

        //<!-- authc:所有url都必须认证通过才可以访问; anon:所有url都都可以匿名访问-->
        filterChainDefinitionMap.put("/login", "anon");
        //filterChainDefinitionMap.put("/druid", "anon");
        filterChainDefinitionMap.put("/services/**", "statelessAuthc");
        factoryBean.setFilterChainDefinitionMap(filterChainDefinitionMap);
        
        factoryBean.getFilters().put("statelessAuthc", statelessAuthcFilter());
        
    	return factoryBean;
    }
   
    @Bean
    public DefaultWebSecurityManager securityManager()
    {
        DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager();
        
        securityManager.setSubjectFactory(subjectFactory());
        securityManager.setSessionManager(sessionManager());
        securityManager.setRealm(statelessRealm());

        /*
         * 禁用使用Sessions 作为存储策略的实现，但它没有完全地禁用Sessions
            * 所以需要配合context.setSessionCreationEnabled(false);
         */
        ((DefaultSessionStorageEvaluator)((DefaultSubjectDAO)securityManager.getSubjectDAO()).getSessionStorageEvaluator()).setSessionStorageEnabled(false);
        
        return securityManager;
    }  

    //生成subject
    @Bean
    public DefaultWebSubjectFactory subjectFactory()
    {
        StatelessDefaultSubjectFactory subjectFactory = new StatelessDefaultSubjectFactory();
        return subjectFactory;
    }

    /**

     * session管理器：

     * sessionManager通过sessionValidationSchedulerEnabled禁用掉会话调度器，

     * 因为我们禁用掉了会话，所以没必要再定期过期会话了。

     * @return

     */
    @Bean
    public DefaultSessionManager sessionManager()
    {
    	DefaultSessionManager sessionManager = new DefaultSessionManager();
    	sessionManager.setSessionValidationSchedulerEnabled(false);
    	return sessionManager;
    }
    
    //自定义realm
    @Bean
    public StatelessAuthorizingRealm statelessRealm()
    {
    	StatelessAuthorizingRealm realm = new StatelessAuthorizingRealm();
    	return realm;
    }
   
    //访问控制器，相当于spring mvc中的DispatcherServlet
    @Bean
    public StatelessAccessControlFilter statelessAuthcFilter()
    {
    	StatelessAccessControlFilter statelessAuthcFilter = new StatelessAccessControlFilter();
    	return statelessAuthcFilter;
    }

    //定义切面，最后这两个Bean是用于权限控制的，不是用于登录验证，这里可以不看
    @Bean
    public AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor(SecurityManager securityManager)
    {
    	AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor = new AuthorizationAttributeSourceAdvisor();
    	authorizationAttributeSourceAdvisor.setSecurityManager(securityManager);
    	return authorizationAttributeSourceAdvisor;
    }

    /**

     * Add.5.2

     *   自动代理所有的advisor:

     *   由Advisor决定对哪些类的方法进行AOP代理。

     */
    //自动将shiro中的切面应用到匹配的Bean中（即为目标Bean创建代理实例）
    @Bean
    public DefaultAdvisorAutoProxyCreator getDefaultAdvisorAutoProxyCreator() 
    {
    	DefaultAdvisorAutoProxyCreator daap = new DefaultAdvisorAutoProxyCreator();
    	daap.setProxyTargetClass(true);
    	return daap;
    }
}
```
### 无状态web配置
sinosteel启用的是无状态的web配置，需要配置三个地方：
1. 根据上下文创建subject的时候，需要关闭session的创建，在StatelessDefaultSubjectFactory的createSubject方法中实现：
```java
public class StatelessDefaultSubjectFactory extends DefaultWebSubjectFactory
{
	 @Override
	 public Subject createSubject(SubjectContext context) 
	 {
		 //不创建session.
		 context.setSessionCreationEnabled(false);
		 return super.createSubject(context);
	 }
}
```

2. 禁用使用session作为存储策略的实现，由securitymanager的subjectDao的sessionStorageEvaluator进行管理，下面是ShiroConfig里面securitymanager方法下的一条语句：
```java
/*
* 禁用使用Sessions 作为存储策略的实现，但它没有完全地禁用Sessions
* 所以需要配合context.setSessionCreationEnabled(false);
*/
((DefaultSessionStorageEvaluator)((DefaultSubjectDAO)securityManager.getSubjectDAO()).getSessionStorageEvaluator()).setSessionStorageEnabled(false);
```
3. 禁用会话调度器，由sessionManager管理：
```java
//ShiroConfig里的一个Bean
@Bean
public DefaultSessionManager sessionManager()
{
    DefaultSessionManager sessionManager = new DefaultSessionManager();
    sessionManager.setSessionValidationSchedulerEnabled(false);
    return sessionManager;
}
```
### 请求控制拦截
首先是用于加密解密的消息摘要算法的实现，com.sinosteel.framework.utils.encryption.HmacSHA256Util类。里面的digest方法用于生成客户端和服务端消息摘要。
接着是实现StatelessAuthenticationToken类，用于保存用户名和摘要，较为简单。
然后开始编写访问控制过滤器StatelessAccessControlFilter：
```java
public class StatelessAccessControlFilter extends AccessControlFilter
{

    //返回true表示允许访问；这里默认false，表示对每一个请求都进行权限验证
	@Override
	protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue)
           throws Exception 
	{
		return false;
    }
 
    //isAccessAllowed返回false才调用，表示需要进一步认证。
    @Override
    protected boolean onAccessDenied(ServletRequest request, ServletResponse response) throws Exception 
    {
    	String clientDigest = request.getParameter("clientDigest");
    	String username = request.getParameter("username");

        //生成无状态token，交由subject完成登录验证
    	StatelessAuthenticationToken token = new StatelessAuthenticationToken(username, clientDigest);
    	
    	try 
    	{
            //交给subject进行登录验证，最终会调用StatelessAuthorizingRealm进行处理
    		getSubject(request, response).login(token);
    		return true;  //验证完成，交由下一个filter过滤
    	} 
    	catch (Exception e) 
    	{
    		e.printStackTrace();
    		onLoginFail(response);
           
    		return false;
    	}
    }
   
    private void onLoginFail(ServletResponse response) throws IOException 
    {
    	HttpServletResponse httpResponse = (HttpServletResponse) response;
    	httpResponse.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
    	httpResponse.getWriter().write("login error");
    }
}

```
getSubject(request, response).login(token);将进入StatelessAuthorizingRealm获取验证信息，完成登录验证。该方法的核心就是获取到AccessControlFilter传递过来的StatelessAuthenticationToken中的参数进行消息摘要，然后生成对象SimpleAuthenticationInfo交给Shiro进行比对，看客户端发来的token与服务端的是否匹配。
```java
public class StatelessAuthorizingRealm extends AuthorizingRealm
{
	//...省略部分代码
   
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) throws AuthenticationException 
    {	
        //获取AccessControlFilter传递过来的StatelessAuthenticationToken
    	StatelessAuthenticationToken statelessToken = (StatelessAuthenticationToken)token;
    	String username = (String)statelessToken.getPrincipal();
    	
        //获取用户的信息
    	JSONObject userInfoJson = CacheUtil.getUserInfoJson(username);
    	if(userInfoJson == null)
    	{ 	
    		User user = userRepository.findByUsername(username);  
        	if(user == null)
        	{
                return null;
            }
        	
        	userInfoJson = CacheUtil.saveUserInfoCache(user);
    	}
    	
        //在服务端生成客户端消息摘要
    	String serverDigest = HmacSHA256Util.digest(getKey(username), userInfoJson.getString("password"));

        //返回authenticationInfo
    	SimpleAuthenticationInfo authenticationInfo = new SimpleAuthenticationInfo(username, serverDigest, getName());
    	return authenticationInfo;
    }
    
    //doGetAuthorizationInfo省略

    //Key的生成策略
    private String getKey(String username) 
    {
		return username;
	}
}

```
至此，登录验证就实现了。总结一下，首先引入Shiro基本配置，然后编写StatelessDefaultSubjectFactory类，用以生产subject；配置无状态web登录；编写消息摘要算法用以消息加密和StatelessAuthenticationToken用以存储用户名和摘要；编写StatelessAccessControlFilter实现对登录请求的过滤，编写StatelessAuthorizingRealm获取token并生成服务端token，用于shiro认证。



## 验证流程源代码分析
首先是StatelessAccessControlFilter过滤器，拦截请求，调用onAccessDenid方法实现登录验证：
```java
//isAccessAllowed返回false才调用，表示需要进一步认证。
@Override
protected boolean onAccessDenied(ServletRequest request, ServletResponse response) throws Exception 
{
    String clientDigest = request.getParameter("clientDigest");
    String username = request.getParameter("username");

    //生成无状态token，交由subject完成登录验证
    StatelessAuthenticationToken token = new StatelessAuthenticationToken(username, clientDigest);
    
    try 
    {
        //交给subject进行登录验证，最终会调用StatelessAuthorizingRealm进行处理
        getSubject(request, response).login(token);
        return true;  //验证完成，交由下一个filter过滤
    } 
    catch (Exception e) 
    {
        e.printStackTrace();
        onLoginFail(response);
        
        return false;
    }
}
```
onAccessDenid方法调用getSubject(request, response).login(token)进行登录验证，进入subject.login()方法，即调用接口类Subject的实现类org.apache.shiro.subject.support.DelegatingSubject类中的login方法：
```java
public void login(AuthenticationToken token) throws AuthenticationException {
    clearRunAsIdentitiesInternal();
    //最终交由securityManager执行login操作
    Subject subject = securityManager.login(this, token);

    PrincipalCollection principals;

    String host = null;

    if (subject instanceof DelegatingSubject) {
        DelegatingSubject delegating = (DelegatingSubject) subject;
        //we have to do this in case there are assumed identities - we don't want to lose the 'real' principals:
        principals = delegating.principals;
        host = delegating.host;
    } else {
        principals = subject.getPrincipals();
    }

    if (principals == null || principals.isEmpty()) {
        String msg = "Principals returned from securityManager.login( token ) returned a null or " +
                "empty value.  This value must be non null and populated with one or more elements.";
        throw new IllegalStateException(msg);
    }
    this.principals = principals;
    this.authenticated = true;  //认证通过
    if (token instanceof HostAuthenticationToken) {
        host = ((HostAuthenticationToken) token).getHost();
    }
    if (host != null) {
        this.host = host;
    }
    Session session = subject.getSession(false);
    if (session != null) {
        this.session = decorate(session);
    } else {
        this.session = null;
    }
}
```
继续调用securityManager.login(this, token);进入securityManager的login方法，即调用接口SecurityManager的实现类org.apache.shiro.mgt.DefaultSecurityManager类的login方法，该方法接收subject和token参数，如果验证成功，则构造一个Subject，代表已经验证的用户返回给DelegatingSubject类的login方法。代码如下：
```java
/**
    * First authenticates the {@code AuthenticationToken} argument, and if successful, constructs a
    * {@code Subject} instance representing the authenticated account's identity.
    如果构造成功则返回一个Subject实例代表作为已经验证的用户的凭证
    * <p/>
    * Once constructed, the {@code Subject} instance is then {@link #bind bound} to the application for
    * subsequent access before being returned to the caller.
    *
    * @param token the authenticationToken to process for the login attempt.
    * @return a Subject representing the authenticated user.
    * @throws AuthenticationException if there is a problem authenticating the specified {@code token}.
    */
public Subject login(Subject subject, AuthenticationToken token) throws AuthenticationException {
    AuthenticationInfo info;
    try {
        info = authenticate(token);
    } catch (AuthenticationException ae) {
        try {
            onFailedLogin(token, ae, subject);
        } catch (Exception e) {
            if (log.isInfoEnabled()) {
                log.info("onFailedLogin method threw an " +
                        "exception.  Logging and propagating original AuthenticationException.", e);
            }
        }
        throw ae; //propagate
    }

    //构造Subject，代表已经验证的用户
    Subject loggedIn = createSubject(token, info, subject);

    onSuccessfulLogin(token, info, loggedIn);

    return loggedIn;
}
```
接着继续执行authenticate(token)方法，返回一个AuthenticationInfo，从这里可以看出，这个AuthenticationInfo肯定是要从我们自定义的StatelessAuthorizingRealm里面取得。我们一步一步分析，authenticate(token)方法如下：
```java
/**
    * Delegates to the wrapped {@link org.apache.shiro.authc.Authenticator Authenticator} for authentication.
    */
public AuthenticationInfo authenticate(AuthenticationToken token) throws AuthenticationException {
    return this.authenticator.authenticate(token);
}

```
提示我们将转到org.apache.shiro.authc.Authenticator 执行authenticate(token)方法，即进入其实现类org.apache.shiro.authc.AbstractAuthenticator执行该方法：
```java
/*
* @param token the submitted token representing the subject's (user's) login principals and credentials.
* @return the AuthenticationInfo referencing the authenticated user's account data.
* @throws AuthenticationException if there is any problem during the authentication process - see the
*                                 interface's JavaDoc for a more detailed explanation.
*/
public final AuthenticationInfo authenticate(AuthenticationToken token) throws AuthenticationException {

    //token为空
    if (token == null) {
        throw new IllegalArgumentException("Method argument (authentication token) cannot be null.");
    }

    log.trace("Authentication attempt received for token [{}]", token);

    AuthenticationInfo info;
    try {
        info = doAuthenticate(token);
        if (info == null) {
            String msg = "No account information found for authentication token [" + token + "] by this " +
                    "Authenticator instance.  Please check that it is configured correctly.";
            throw new AuthenticationException(msg);
        }
    } catch (Throwable t) {
        AuthenticationException ae = null;
        if (t instanceof AuthenticationException) {
            ae = (AuthenticationException) t;
        }
        if (ae == null) {
            //Exception thrown was not an expected AuthenticationException.  Therefore it is probably a little more
            //severe or unexpected.  So, wrap in an AuthenticationException, log to warn, and propagate:
            String msg = "Authentication failed for token submission [" + token + "].  Possible unexpected " +
                    "error? (Typical or expected login exceptions should extend from AuthenticationException).";
            ae = new AuthenticationException(msg, t);
            if (log.isWarnEnabled())
                log.warn(msg, t);
        }
        try {
            notifyFailure(token, ae);
        } catch (Throwable t2) {
            if (log.isWarnEnabled()) {
                String msg = "Unable to send notification for failed authentication attempt - listener error?.  " +
                        "Please check your AuthenticationListener implementation(s).  Logging sending exception " +
                        "and propagating original AuthenticationException instead...";
                log.warn(msg, t2);
            }
        }


        throw ae;
    }

    log.debug("Authentication successful for token [{}].  Returned account [{}]", token, info);

    notifySuccess(token, info);

    return info;
}
```
接着进入doAuthenticate(token)方法，该方法交由AbstractAuthenticator的子类org.apache.shiro.authc.pam.ModularRealmAuthenticator类实现：
```java
/**
* Attempts to authenticate the given token by iterating over the internal collection of
* {@link Realm}s.  For each realm, first the {@link Realm#supports(org.apache.shiro.authc.AuthenticationToken)}
* method will be called to determine if the realm supports the {@code authenticationToken} method argument.
* <p/>
* If a realm does support
* the token, its {@link Realm#getAuthenticationInfo(org.apache.shiro.authc.AuthenticationToken)}
* method will be called.  If the realm returns a non-null account, the token will be
* considered authenticated for that realm and the account data recorded.  If the realm returns {@code null},
* the next realm will be consulted.  If no realms support the token or all supporting realms return null,
* an {@link AuthenticationException} will be thrown to indicate that the user could not be authenticated.
* <p/>
* After all realms have been consulted, the information from each realm is aggregated into a single
* {@link AuthenticationInfo} object and returned.
*
*/
protected AuthenticationInfo doAuthenticate(AuthenticationToken authenticationToken) throws AuthenticationException {
    assertRealmsConfigured();
    Collection<Realm> realms = getRealms();
    if (realms.size() == 1) {
        return doSingleRealmAuthentication(realms.iterator().next(), authenticationToken);
    } else {
        return doMultiRealmAuthentication(realms, authenticationToken);
    }
}
```
如注解所说，上述方法遍历所有realms，检查realms是否支持传入的AuthenticationToken，如果支持则调用该realm的getAuthenticateInfo方法，具体实现可分为对单个realm和多个realm两种方法，下面只看doMutiRealmAuthentication方法：
```java
protected AuthenticationInfo doMultiRealmAuthentication(Collection<Realm> realms, AuthenticationToken token) {

    /*验证策略
    llSuccessFulStrategy:所有Realm验证成功才算成功，且返回所有Realm身份验证成功的认证信息，如果有一个失败就失败了。

    AtLeastOneSuccessFulAtrategy:只要有一个Realm验证成功即可，和FirstSuccessfulStrategy 不同，将返回所有Realm身份验证成功的认证信息；

    FirstSuccessFulStrategy:只要有一个 Realm 验证成功即可，只返回第一个 Realm 身份验证成功的认证信息
    */
    AuthenticationStrategy strategy = getAuthenticationStrategy();

    AuthenticationInfo aggregate = strategy.beforeAllAttempts(realms, token);

    if (log.isTraceEnabled()) {
        log.trace("Iterating through {} realms for PAM authentication", realms.size());
    }

    for (Realm realm : realms) {

        aggregate = strategy.beforeAttempt(realm, token, aggregate);

        if (realm.supports(token)) {

            log.trace("Attempting to authenticate token [{}] using realm [{}]", token, realm);

            AuthenticationInfo info = null;
            Throwable t = null;
            try {
                info = realm.getAuthenticationInfo(token);
            } catch (Throwable throwable) {
                t = throwable;
                if (log.isWarnEnabled()) {
                    String msg = "Realm [" + realm + "] threw an exception during a multi-realm authentication attempt:";
                    log.warn(msg, t);
                }
            }

            aggregate = strategy.afterAttempt(realm, token, info, aggregate, t);

        } else {
            log.debug("Realm [{}] does not support token {}.  Skipping realm.", realm, token);
        }
    }

    aggregate = strategy.afterAllAttempts(token, aggregate);

    return aggregate;
}
```
这里面就调用了realm的getAuthenticationInfo方法，及调用org.apache.shiro.realm.Realm接口的getAuthenticationInfo方法，该方法在org.apache.shiro.realm.AuthenticatingRealm类中实现，如下：
```java
public final AuthenticationInfo getAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {

    //查缓存，如果缓存有AuthenticationInfo，则返回
    AuthenticationInfo info = getCachedAuthenticationInfo(token);
    if (info == null) {
        //otherwise not cached, perform the lookup:
        info = doGetAuthenticationInfo(token);
        log.debug("Looked up AuthenticationInfo [{}] from doGetAuthenticationInfo", info);
        if (token != null && info != null) {
            cacheAuthenticationInfoIfPossible(token, info);
        }
    } else {
        log.debug("Using cached authentication info [{}] to perform credentials matching.", info);
    }

    if (info != null) {
        assertCredentialsMatch(token, info);
    } else {
        log.debug("No AuthenticationInfo found for submitted AuthenticationToken [{}].  Returning null.", token);
    }

    return info;
}
```
执行info = doGetAuthenticationInfo(token);因为AuthorizingRealm继承自AuthenticatingRealm，而AuthorizingRealm将doGetAuthenticationInfo(token)方法交由其子类实现，故doGetAuthenticationInfo(token)方法由我们自定义的com.sinosteel.framework.core.auth.StatelessAuthorizingRealm实现，如下：
```java
@Override
protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) throws AuthenticationException 
{	
    StatelessAuthenticationToken statelessToken = (StatelessAuthenticationToken)token;
    String username = (String)statelessToken.getPrincipal();
    
    JSONObject userInfoJson = CacheUtil.getUserInfoJson(username);
    if(userInfoJson == null)
    { 	
        User user = userRepository.findByUsername(username);  
        if(user == null)
        {
            return null;
        }
        
        userInfoJson = CacheUtil.saveUserInfoCache(user);
    }
    
    String serverDigest = HmacSHA256Util.digest(getKey(username), userInfoJson.getString("password"));

    SimpleAuthenticationInfo authenticationInfo = new SimpleAuthenticationInfo(username, serverDigest, getName());
    return authenticationInfo;
}
```
一层层将AuthenticationInfo返回给securityManager.login(this, token);用于构造已经验证的用户。至此登录验证的流程就分析完了。


> 本篇博客登录验证的实现参考ITEYE博客网站上林祥纤的[Spring boot之无状态](http://412887952-qq-com.iteye.com/blog/2359084)系列以及极客学院系列文章[跟我学Shiro](http://wiki.jikexueyuan.com/project/shiro/interceptor.html)