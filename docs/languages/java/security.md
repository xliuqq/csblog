# 安全

## AccessController

 类java.security.AccessController提供了一个默认的安全策略执行机制，它使用栈检查来决定潜在不安全的操作是否被允许。

每一个栈帧代表了由当前线程调用的某个方法，每一个方法是在某个类中定义的，每一个类又属于某个保护域，每个保护域包含一些权限。因此，每个栈帧间接地和一些权限相关。

**Java默认不打开安全检查，如果不打开，本地程序拥有所有权限：**

例如对于File::createNewFile，获取SecurityManager，检查是否有对文件的写权限，最终通过AccessController::checkPermission检查；

- 静态方法checkPermission()，这个方法决定一个特定的操作能否被允许。

- Java SDK 给域提供了 doPrivileged 方法，让程序突破当前域权限限制，临时扩大访问权限。

```java
AccessController.doPrivileged(new PrivilegedAction() {
    public Object run() {
      try {
        Method getCleanerMethod = buffer.getClass().getMethod("cleaner", new Class[0]);
        getCleanerMethod.setAccessible(true);
        sun.misc.Cleaner cleaner = (sun.misc.Cleaner)
        getCleanerMethod.invoke(byteBuffer, new Object[0]);
        cleaner.clean();
      } catch (Exception e) {
        e.printStackTrace();
      }
      return null;
    }
});
```



## JAAS

JAAS是Java Authentication and Authorization Service的缩写，提供了**认证与授权相关的服务框架与接口定义**:

- 认证：认证主要是验证一个用户的身份。
- 授权：授权用户访问或操作某些敏感的资源。

### Subject

**Subject**来描述这个**资源请求主体**与安全访问相关的信息，Subject通常是指一个对象实体，或者一个服务。
Subject中所关联的信息：

- 身份信息
- 密码信息
- 加密密钥/凭据信息

一个Subject可能拥有一个或多个身份，一个身份被称之为**Principal**，也就是说，一个Subject可能关联一个或多个Principals。

一个Subject可能涉及与安全有关的**凭据**信息(密钥/票据)，称之为**Credentials**。

### LoginContext

认证上下文信息，提供了针对Subject对象进行认证的基础方法，每一个LoginContext都关联一个Context Name。

- login 登录/认证，该过程由具体的LoginModule代理完成。

- logout 登出

- getSubject 获取认证之后所创建的Subject对象信息

初始化一个LoginContext对象时，可以传入如下一些参数：

- Context Name[必选]
- Subject[可选]
- CallbackHandler[可选]
- Configuration[可选]

### LoginModule

LoginModule提供了登录/认证的**基础接口定义**，所有的认证服务都需要实现该接口。一些典型的实现模块包括：

- `Krb5LoginModule` 基于Kerberos的登录/认证服务模块

- `JndiLoginModule` 基于用户名和密码的登录/认证服务模块

- `KeyStoreLoginModule` 基于KeyStore的登录/认证服务模块

主要方法为`login`与`commit`：

#### login

对Subject进行认证。

这个过程中主要涉及到用户名和密码信息提示，校验用户密码。认证结果将会在LoginModule层面暂时保存。

#### commit

Commit过程首先确认LoginModule中保存的认证结果。认证成功之后，Commit方法将Subject内对应的Principals以及Credentials关联起来。如果Login方法认证失败的话，则该方法将会清理在LoginModule中保存的认证结果信息。

