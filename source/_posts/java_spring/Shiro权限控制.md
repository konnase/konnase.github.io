---
title: Shiro权限控制
p: java_spring/Shiro权限控制
date: 2017-11-15 19:05:10
tags: [shiro, authorization]
categories: java
---

## shiro实现访问权限控制和用户身份认证


![image](http://wiki.jikexueyuan.com/project/shiro/images/3.png)
### shiro的三大核心组件
1. Subject：即当前用户，在权限管理的应用程序里往往需要知道谁能够操作什么，谁拥有操作该程序的权利，shiro中则需要通过Subject来提供基础的当前用户信息，Subject 不仅仅代表某个用户，也可以是第三方进程、后台帐户（Daemon Account）或其他类似事物。
2. SecurityManager：即所有Subject的管理者，这是Shiro框架的核心组件，可以把他看做是一个Shiro框架的全局管理组件，用于调度各种Shiro框架的服务。
3. Realms：域，Shiro 从从 Realm 获取安全数据（如用户、角色、权限），就是说 SecurityManager 要验证用户身份，那么它需要从Realm获取相应的用户进行比较以确定用户身份是否合法；也需要从 Realm得到用户相应的角色/权限进行验证用户是否能进行操作；可以把Realm看成DataSource，即安全数据源。Realms则是用户的信息认证器和用户的权限人证器，我们需要自己来实现Realms来自定义的管理我们自己系统内部的权限规则。

<!--more-->

### 自定义Realm：
[shiro_example](https://github.com/swinglife/shiro_ex)
[原博客地址](http://blog.csdn.net/swingpyzf/article/details/46342023/#reply)

Shiro默认提供的Realm如下图所示：
![image](http://wiki.jikexueyuan.com/project/shiro/images/5.png)
开发的时候选择继承自org.apache.shiro.realm.AuthorizingRealm， 本例中，继承自AuthorizingRealm并重写方法：
```java
/*** 
     * 获取授权信息 
     */  
    @Override  
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection pc) {  
        //根据自己系统规则的需要编写获取授权信息，这里为了快速入门只获取了用户对应角色的资源url信息  
        String username = (String) pc.fromRealm(getName()).iterator().next();  
        if (username != null) {  
            List<String> pers = accountService.getPermissionsByUserName(username);  
            if (pers != null && !pers.isEmpty()) {  
                SimpleAuthorizationInfo info = new SimpleAuthorizationInfo();  
                for (String each : pers) {  
                    //将权限资源添加到用户信息中  
                    info.addStringPermission(each);  
                }  
                return info;  
            }  
        }  
        return null;  
    }  
    /*** 
     * 获取认证信息 
     */  
    @Override  
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken at) throws AuthenticationException {  
        UsernamePasswordToken token = (UsernamePasswordToken) at;  
        // 通过表单接收的用户名  
        String username = token.getUsername();  
        if (username != null && !"".equals(username)) {  
            User user = accountService.getUserByUserName(username);  
            if (user != null) {  
                return new SimpleAuthenticationInfo(user.getUsername(), user.getPassword(), getName());  
            }  
        }  
  
        return null;  
    }
```
在doGetAuthorizationInfo方法中，获取的权限为accountService.getPermissionsByUserName(username)方法返回的permission.url信息。表示该用户具有访问哪些url的权限。
在doGetAuthenticationInfo方法中，通过表单传来的用户名，从持久化层中获取对应用户名的用户信息，用于认证用户。

### 授权
1. 主体
主体，即访问应用的用户，在 Shiro 中使用 Subject 代表该用户。用户只有授权后才允许访问相应的资源。

2. 资源
在应用中用户可以访问的任何东西，比如访问JSP页面、查看/编辑某些数据、访问某个业务方法、打印文本等等都是资源。用户只要授权后才能访问。

3. 表示在应用中用户有没有操作某个资源的权力。反映在某个资源上的操作允不允许，不反映谁去执行这个操作。所以后续还需要把权限赋予给用户，即定义哪个用户允许在某个资源上做什么操作（权限），**Shiro 不会去做这件事情，而是由实现人员提供**。
4. 角色
角色代表了操作集合，可以理解为权限的集合，一般情况下我们会赋予用户角色而不是权限，即这样用户可以拥有一组权限，赋予权限时比较方便
Shiro 不会去维护用户、维护权限；这些需要我们自己去设计 / 提供；然后通过相应的接口注入给 Shiro 即可。
隐式角色：
基于角色的访问控制。直接通过角色验证用户有没有操作某个资源的权限，默认某角色拥有哪些权限，但这些系统并没有明确定义一个角色到底包含了哪些可执行的行为。所以一旦对权限的需求有稍微的变动，就可能需要重新修改代码。比如说，用户user拥有两种角色：部门经理和项目经理，对user进行角色访问控制：
```java
if(user.hasRole(DepartManager) && user.hasRole(ProjectManager))
{
    //把部门经理和项目经理的权限授予user
}
```
这时候如果说用户不再拥有项目经理的权限了，就需要修改代码，将后面一条判断去掉，重新编译系统。如果需求变动大，就需要大量的代码重构。
显示角色：
基于资源的访问控制。在程序中通过权限控制谁能访问某个资源，角色聚合一组权限集合；这样假设哪个角色不能访问某个资源，只需要从角色代表的权限集合中移除即可。基于什么是受保护的， 而不是谁可能有能力做什么

### 实现用户可访问所属组织及其子组织的文件权限控制机制
由于shiro中没有定义哪个用户允许在某个资源上做什么操作（权限），需要开发人员自行实现，设计一个实现机制：
采用基于资源的权限控制。所需访问的文件命名为report，用户为user1，所属组织为role_parent，子组织为role_child。
角色权限分配如下：
```ini
[users]
user1=123,role_parent
[roles]
role_child=report:view
role_parent=report:view
```
表示user1是属于role_parent组的，且role_parent和role_child都具有查看report的权限，将以上角色权限分配规则存入数据库中。对于user1,在自定义的realm中重写doGetAuthorizationInfo方法
```
@Override  
protected AuthorizationInfo doGetAuthorizationInfo(  
        PrincipalCollection principals)
```
根据principals.getPrimaryPrincipal()方法获取user1，然后根据user1查询user1拥有的所有权限的列表list1，然后将list1中的权限通过simpleAuthorizationInfo.addStringPermission(permission)方法添加到AuthorizationInfo中，realm返回该AuthorizationInfo。实际使用的时候（比如测试时），执行tmp=subject.isPermitted(report:view)，如果tmp==true，则表示user1具有该权限。
简单的讲，就是要求子组织下的每个report都要事先将权限赋予其父组织，这样在处理该问题的时候较容易，而且这种机制也很容易实现，每次创建一个report的时候，就在permission表里面增加两条该report对应组织和父组织的权限记录。

**Reference:**
[Shiro教程](http://wiki.jikexueyuan.com/project/shiro/)