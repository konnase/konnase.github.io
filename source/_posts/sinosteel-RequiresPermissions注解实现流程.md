---
title: sinosteel @RequiresPermissions注解实现流程
date: 2017-11-25 14:41:00
tags: [spring, aop, annotation, interceptor, shiro]
---

[Github代码仓库](https://github.com/DimitriZhao/sinosteel.git)

## shiro注解授权源码实现
### 授权流程
概括的说，shiro在实现注解授权时采用的是Spring AOP的方式。
在Controller里面的相关方法上添加@RequiresPermissions()注解，Shiro即可自动调用AuthorizationAttributeSourceAdvisor进行AOP操作。从自定义的com.sinosteel.framework.config.shiro.ShiroConfig类开始，该类用@Configuration注解，表示为一个Spring IoC容器，往IoC容器中加入两个Bean
```java
@Bean
public AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor(SecurityManager securityManager)
{
	AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor = new AuthorizationAttributeSourceAdvisor();
	authorizationAttributeSourceAdvisor.setSecurityManager(securityManager);
	return authorizationAttributeSourceAdvisor;
}

@Bean
public DefaultAdvisorAutoProxyCreator getDefaultAdvisorAutoProxyCreator() 
{
	DefaultAdvisorAutoProxyCreator daap = new DefaultAdvisorAutoProxyCreator();
	daap.setProxyTargetClass(true);
	return daap;
}
```
<!--more-->
1. AuthorizationAttributeSourceAdvisor继承自StaticMethodMatcherPointcutAdvisor，StaticMethodMatcherPointcutAdvisor为静态方法匹配器切点定义的切面，默认情况下，匹配所有的目标类；继承自PointcutAdvisor，代表具有切点的切面，它包含Advice和Pointcut两个类，这样我们就可以通过类、方法名以及方法方位等信息灵活地定义切面的连接点，提供更具适用性的切面。将shiro的securityManager传入AuthorizationAttributeSourceAdvisor中，用于后续获取用户的信息。
2. DefaultAdvisorAutoProxyCreator表示基于Advisor匹配机制的自动代理创建器：它会对容器中所有的Advisor进行扫描，自动将这些切面应用到匹配的Bean中（即为目标Bean创建代理实例），是Spring为我们提供的自动代理机制，让容器为我们自动生成代理，把我们从烦琐的配置工作中解放出来。

AuthorizationAttributeSourceAdvisor类定义了一个切面，如下：
```java
public class AuthorizationAttributeSourceAdvisor extends StaticMethodMatcherPointcutAdvisor {

    private static final Logger log = LoggerFactory.getLogger(AuthorizationAttributeSourceAdvisor.class);

    private static final Class<? extends Annotation>[] AUTHZ_ANNOTATION_CLASSES =
            new Class[] {
                    RequiresPermissions.class, RequiresRoles.class,
                    RequiresUser.class, RequiresGuest.class, RequiresAuthentication.class
            };

    protected SecurityManager securityManager = null;

    /**
     * Create a new AuthorizationAttributeSourceAdvisor.
     */
    public AuthorizationAttributeSourceAdvisor() {
        **setAdvice**(new AopAllianceAnnotationsAuthorizingMethodInterceptor());
    }

    public SecurityManager getSecurityManager() {
        return securityManager;
    }

    public void setSecurityManager(org.apache.shiro.mgt.SecurityManager securityManager) {
        this.securityManager = securityManager;
    }

    /**
     * Returns <tt>true</tt> if the method has any Shiro annotations, false otherwise.
     * The annotations inspected are:
     * <ul>
     * <li>{@link org.apache.shiro.authz.annotation.RequiresAuthentication RequiresAuthentication}</li>
     * <li>{@link org.apache.shiro.authz.annotation.RequiresUser RequiresUser}</li>
     * <li>{@link org.apache.shiro.authz.annotation.RequiresGuest RequiresGuest}</li>
     * <li>{@link org.apache.shiro.authz.annotation.RequiresRoles RequiresRoles}</li>
     * <li>{@link org.apache.shiro.authz.annotation.RequiresPermissions RequiresPermissions}</li>
     * </ul>
     *
     * @param method      the method to check for a Shiro annotation
     * @param targetClass the class potentially declaring Shiro annotations
     * @return <tt>true</tt> if the method has a Shiro annotation, false otherwise.
     * @see org.springframework.aop.MethodMatcher#matches(java.lang.reflect.Method, Class)
     */
    public boolean matches(Method method, Class targetClass) {
        Method m = method;

        if ( isAuthzAnnotationPresent(m) ) {
            return true;
        }

        //The 'method' parameter could be from an interface that doesn't have the annotation.
        //Check to see if the implementation has it.
        if ( targetClass != null) {
            try {
                m = targetClass.getMethod(m.getName(), m.getParameterTypes());
                if ( isAuthzAnnotationPresent(m) ) {
                    return true;
                }
            } catch (NoSuchMethodException ignored) {
                //default return value is false.  If we can't find the method, then obviously
                //there is no annotation, so just use the default return value.
            }
        }

        return false;
    }

    private boolean isAuthzAnnotationPresent(Method method) {
        for( Class<? extends Annotation> annClass : AUTHZ_ANNOTATION_CLASSES ) {
            Annotation a = AnnotationUtils.findAnnotation(method, annClass);
            if ( a != null ) {
                return true;
            }
        }
        return false;
    }

}
```
AuthorizationAttributeSourceAdvisor类中的setAdvice方法，传入一个Advice（通知）。查看AopAllianceAnnotationsAuthorizingMethodInterceptor的继承关系发现该类继承自org.aopalliance.aop.Advice和org.apache.shiro.aop.MethodInterceptor
![image](/img/inherit_aopmethod_interceptor.png)
AopAllianceAnnotationsAuthorizingMethodInterceptor的方法如下：
```java
public AopAllianceAnnotationsAuthorizingMethodInterceptor() {
    List<AuthorizingAnnotationMethodInterceptor> interceptors =
            new ArrayList<AuthorizingAnnotationMethodInterceptor>(5);

    //use a Spring-specific Annotation resolver - Spring's AnnotationUtils is nicer than the
    //raw JDK resolution process.
    AnnotationResolver resolver = new SpringAnnotationResolver();
    //we can re-use the same resolver instance - it does not retain state:
    interceptors.add(new RoleAnnotationMethodInterceptor(resolver));
    interceptors.add(new PermissionAnnotationMethodInterceptor(resolver));
    interceptors.add(new AuthenticatedAnnotationMethodInterceptor(resolver));
    interceptors.add(new UserAnnotationMethodInterceptor(resolver));
    interceptors.add(new GuestAnnotationMethodInterceptor(resolver));

    setMethodInterceptors(interceptors);
}
```
可以看到，这里添加了5个AuthorizingAnnotationMethodInterceptor，分别对应@RequiresRoles注解权限、@RequiresPermissions注解权限、@RequiresAuthentication注解权限、@RequiresUser注解权限和@RequiresGuest注解权限。这里以@RequiresPermissions为例展开讨论。
PermissionAnnotationMethodInterceptor继承自AuthorizingAnnotationMethodInterceptor，前者的内部构造器：
```java
public PermissionAnnotationMethodInterceptor(AnnotationResolver resolver) {
    super( new PermissionAnnotationHandler(), resolver);
}
```
上面代码new一个PermissionAnnotationHandler对象。然后PermissionAnnotationHandler对象的构造器设置annotationClass = RequiresPermissions.class，如下
```java
public PermissionAnnotationHandler() {
    super(RequiresPermissions.class);
}
```
这样PermissionAnnotationMethodInterceptor就设置好了，回到AopAllianceAnnotationsAuthorizingMethodInterceptor方法拦截器中，代理对象的方法被调用时触发回调方法invoke()，调用间接父类AuthorizingMethodInterceptor的invoke()方法，如下：
```java
public Object invoke(MethodInvocation methodInvocation) throws Throwable {
        assertAuthorized(methodInvocation);
        return methodInvocation.proceed();
    }
```
调用assertAuthorized(methodInvocation)方法，即调用AopAllianceAnnotationsAuthorizingMethodInterceptor的父类AnnotationsAuthorizingMethodInterceptor的assertAuthorized(methodInvocation)方法，如下：
```java
/**
 * Iterates over the internal {@link #getMethodInterceptors() methodInterceptors} collection, and for each one,
 * ensures that if the interceptor
 * {@link AuthorizingAnnotationMethodInterceptor#supports(org.apache.shiro.aop.MethodInvocation) supports}
 * the invocation, that the interceptor
 * {@link AuthorizingAnnotationMethodInterceptor#assertAuthorized(org.apache.shiro.aop.MethodInvocation) asserts}
 * that the invocation is authorized to proceed.
 */
protected void assertAuthorized(MethodInvocation methodInvocation) throws AuthorizationException {
    //default implementation just ensures no deny votes are cast:
    Collection<AuthorizingAnnotationMethodInterceptor> aamis = getMethodInterceptors();
    if (aamis != null && !aamis.isEmpty()) {
        for (AuthorizingAnnotationMethodInterceptor aami : aamis) {
            if (aami.supports(methodInvocation)) {
                aami.assertAuthorized(methodInvocation);
            }
        }
    }
}
```
上面根据传入的AuthorizingAnnotationMethodInterceptor的不同，调用子类的assertAuthorized方法。因为这里传入的是PermissionAnnotationMethodInterceptor，故执行PermissionAnnotationMethodInterceptor的assertAuthorized方法，由于PermissionAnnotationMethodInterceptor没有重写assertAuthorized方法，故调用其父类AuthorizingAnnotationMethodInterceptor的assertAuthorized方法，如下：
```java
public void assertAuthorized(MethodInvocation mi) throws AuthorizationException {
    try {
        ((AuthorizingAnnotationHandler)getHandler()).assertAuthorized(getAnnotation(mi));
    }
    catch(AuthorizationException ae) {
        // Annotation handler doesn't know why it was called, so add the information here if possible. 
        // Don't wrap the exception here since we don't want to mask the specific exception, such as 
        // UnauthenticatedException etc. 
        if (ae.getCause() == null) ae.initCause(new AuthorizationException("Not authorized to invoke method: " + mi.getMethod()));
        throw ae;
    }         
}
```
PermissionAnnotationMethodInterceptor类的继承关系如图：
![image](/img/permission_annotation_method_interceptor.png)
((AuthorizingAnnotationHandler)getHandler()).assertAuthorized(getAnnotation(mi))调用即执行PermissionAnnotationHandler中的assertAuthorized方法，如下：
```java
    /**
     * Ensures that the calling <code>Subject</code> has the Annotation's specified permissions, and if not, throws an
     * <code>AuthorizingException</code> indicating access is denied.
     *
     * @param a the RequiresPermission annotation being inspected to check for one or more permissions
     * @throws org.apache.shiro.authz.AuthorizationException
     *          if the calling <code>Subject</code> does not have the permission(s) necessary to
     *          continue access or execution.
     */
    public void assertAuthorized(Annotation a) throws AuthorizationException {
        if (!(a instanceof RequiresPermissions)) return;

        RequiresPermissions rpAnnotation = (RequiresPermissions) a;
        String[] perms = getAnnotationValue(a);
        Subject subject = getSubject();

        if (perms.length == 1) {
            subject.checkPermission(perms[0]);
            return;
        }
        if (Logical.AND.equals(rpAnnotation.logical())) {
            getSubject().checkPermissions(perms);
            return;
        }
        if (Logical.OR.equals(rpAnnotation.logical())) {
            // Avoid processing exceptions unnecessarily - "delay" throwing the exception by calling hasRole first
            boolean hasAtLeastOnePermission = false;
            for (String permission : perms) if (getSubject().isPermitted(permission)) hasAtLeastOnePermission = true;
            // Cause the exception if none of the role match, note that the exception message will be a bit misleading
            if (!hasAtLeastOnePermission) getSubject().checkPermission(perms[0]);
            
        }
    }
```
上述代码通过getSubject()方法获取用户对象，通过subject.checkPermission方法进行授权验证，进入到DelegatingSubject类中，查看checkPermission方法，如下：
```java
    public void checkPermission(String permission) throws AuthorizationException {
        assertAuthzCheckPossible();
        securityManager.checkPermission(getPrincipals(), permission);
    }
```
执行securityManager.checkPermission(getPrincipals(), permission)，进入AuthorizingSecurityManager中，执行方法如下：
```java
    public void checkPermission(PrincipalCollection principals, String permission) throws AuthorizationException {
        this.authorizer.checkPermission(principals, permission);
    }
    //构造函数
    public AuthorizingSecurityManager() {
        super();
        this.authorizer = new ModularRealmAuthorizer();
    }
```
可以看出，上面调用ModularRealmAuthorizer.checkPermission方法，该方法如下：
```java
    public void checkPermission(PrincipalCollection principals, String permission) throws AuthorizationException {
        assertRealmsConfigured();
        if (!isPermitted(principals, permission)) {
            throw new UnauthorizedException("Subject does not have permission [" + permission + "]");
        }
    }
```
该方法首先检查realm是否被设置，然后调用isPermitted方法，如下：
```java
    public boolean isPermitted(PrincipalCollection principals, String permission) {
        assertRealmsConfigured();
        for (Realm realm : getRealms()) {
            if (!(realm instanceof Authorizer)) continue;
            if (((Authorizer) realm).isPermitted(principals, permission)) {
                return true;
            }
        }
        return false;
    }
```
上面就调用到了Realm的isPermitted方法，查看Realm的注解：
```java
<p>Most users will not implement the <tt>Realm</tt> interface directly, but will extend one of the subclasses,
 * {@link org.apache.shiro.realm.AuthenticatingRealm AuthenticatingRealm} or {@link org.apache.shiro.realm.AuthorizingRealm}, greatly reducing the effort requird
 * to implement a <tt>Realm</tt> from scratch.</p>
 *
 * @see org.apache.shiro.realm.CachingRealm CachingRealm
 * @see org.apache.shiro.realm.AuthenticatingRealm AuthenticatingRealm
 * @see org.apache.shiro.realm.AuthorizingRealm AuthorizingRealm
 * @see org.apache.shiro.authc.pam.ModularRealmAuthenticator ModularRealmAuthenticator
```
上面解释了大部分开发者不会直接实现Realm类，而是选择实现其几个子类。这里便使用了AuthorizingRealm的isPermitted方法，如下：
```java
    public boolean isPermitted(PrincipalCollection principals, Permission permission) {
        AuthorizationInfo info = getAuthorizationInfo(principals);
        return isPermitted(permission, info);
    }
```
getAuthorizationInfo(principals)方法是获取对象拥有的权限信息，如下：
```java
    /*
     * @param principals the corresponding Subject's identifying principals with which to look up the Subject's
     *                   {@code AuthorizationInfo}.
     * @return the authorization information for the account associated with the specified {@code principals},
     *         or {@code null} if no account could be found.
     */
    protected AuthorizationInfo getAuthorizationInfo(PrincipalCollection principals) {

        if (principals == null) {
            return null;
        }

        AuthorizationInfo info = null;

        if (log.isTraceEnabled()) {
            log.trace("Retrieving AuthorizationInfo for principals [" + principals + "]");
        }

        //查缓存，如果有AuthorizationInfo则返回
        Cache<Object, AuthorizationInfo> cache = getAvailableAuthorizationCache();
        if (cache != null) {
            if (log.isTraceEnabled()) {
                log.trace("Attempting to retrieve the AuthorizationInfo from cache.");
            }
            Object key = getAuthorizationCacheKey(principals);
            info = cache.get(key);
            if (log.isTraceEnabled()) {
                if (info == null) {
                    log.trace("No AuthorizationInfo found in cache for principals [" + principals + "]");
                } else {
                    log.trace("AuthorizationInfo found in cache for principals [" + principals + "]");
                }
            }
        }


        if (info == null) {
            // Call template method if the info was not found in a cache
            info = doGetAuthorizationInfo(principals);
            // If the info is not null and the cache has been created, then cache the authorization info.
            if (info != null && cache != null) {
                if (log.isTraceEnabled()) {
                    log.trace("Caching authorization info for principals: [" + principals + "].");
                }
                Object key = getAuthorizationCacheKey(principals);
                cache.put(key, info);
            }
        }

        return info;
    }
```
上述代码中执行info = doGetAuthorizationInfo(principals);该方法就是我们自定义的StatelessAuthorizingRealm（继承自AuthorizingRealm）里面实现的doGetAuthorizationInfo方法，该方法返回AuthorizationInfo，然后调用重载的isPermitted方法，如下：
```
    //changed visibility from private to protected for SHIRO-332
    protected boolean isPermitted(Permission permission, AuthorizationInfo info) {
        Collection<Permission> perms = getPermissions(info);
        if (perms != null && !perms.isEmpty()) {
            for (Permission perm : perms) {
                if (perm.implies(permission)) {
                    return true;
                }
            }
        }
        return false;
    }
```
上述方法判断通过注解@RequiresPermissions传入的permission是否在AuthorizationInfo中，如果在就返回true，允许用户访问该资源。到此@RequiresPermissions的权限控制就结束了。
