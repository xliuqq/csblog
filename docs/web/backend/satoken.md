# SaToken

> ##### 轻量级 Java 权限认证框架，基于 RBAC 权限模型。
>
> 主要解决：**登录认证**、**权限认证**、**单点登录**、**OAuth2.0**、**分布式Session会话**、**微服务网关鉴权** 等一系列权限相关问题

## 登录与注销

Sa-Token 为这个账号创建了一个Token凭证，且通过 Cookie 上下文返回给了前端：

```java
// 会话登录：参数填写要登录的账号id，建议的数据类型：long | int | String， 不可以传入复杂类型，如：User、Admin 等等
StpUtil.login(Object id); 
// 当前会话注销登录
StpUtil.logout();

// 获取当前会话账号id, 如果未登录，则抛出异常：`NotLoginException`
StpUtil.getLoginId();
```



### 多账号认证

> https://sa-token.com/doc.html#/up/many-account

在一个项目中设计两套账号体系，比如一个电商系统的 `user表` 和 `admin表`：

- 复制另写`StpUtil.java`，更改字段`LoginType`；
- 对`@SaCheckLogin`（检验是否登录）、`@SaCheckRole`、`@SaCheckPermission`，指定`type`为上面修改的`LoginType`；

#### 同端多登陆

假设不仅需要在后台同时集成两套账号，还需要**在一个客户端同时登陆两套账号**（业务场景举例：一个APP中可以同时登陆商家账号和用户账号）。在客户端会发生`token覆盖`，新登录的token会覆盖掉旧登录的token从而导致旧登录失效；

- 针对新写的`StpUserUtil`，重写其`TokenName`；

```java
public class StpUserUtil {
    // 使用匿名子类 重写`stpLogic对象`的一些方法 
    public static StpLogic stpLogic = new StpLogic("user") {
        // 重写 StpLogic 类下的 `splicingKeyTokenName` 函数，返回一个与 `StpUtil` 不同的token名称, 防止冲突 
        @Override
        public String splicingKeyTokenName() {
            return super.splicingKeyTokenName() + "-user";
        }
        // 同理你可以按需重写一些其它方法 ... 
    }; 
    // ... 
}
```



### 多类型/设备登录

强制注销 和 踢人下线 的区别在于：

- 强制注销等价于对方主动调用了注销方法，再次访问会提示：Token无效。
- 踢人下线不会清除Token信息，而是将其打上特定标记，再次访问会提示：Token已被踢下线。

#### 踢人下线

```java
StpUtil.kickout(10001);                    // 将指定账号踢下线 
StpUtil.kickout(10001, "PC");              // 将指定账号指定端踢下线
StpUtil.kickoutByTokenValue("token");      // 将指定 Token 踢下线
```

#### 强制注销

```java
// 指定`账号id`和`设备类型`进行强制注销 
StpUtil.logout(10001, "PC");    
```

#### 同设备互斥登录

在配置文件中，将 `isConcurrent` 配置为false，然后调用登录等相关接口时声明设备类型

```java
// 指定`账号id`和`设备类型`进行登录
StpUtil.login(10001, "PC");    
```

调用此方法登录后，**同设备的会被顶下线**（不同设备不受影响），再次访问系统时会抛出 `NotLoginException` 异常(场景值=`-4`)

### 账号封禁

#### 账号封禁

对于正在登录的账号，将其封禁并不会使它立即掉线，如果我们需要它即刻下线，可采用**先踢再封禁**的策略

-  校验封禁 和 登录（`StpUtil.login`）两个动作分离成两个方法，登录时不再自动校验是否封禁

```java
// 封禁指定账号1天时间，-1表示永久封禁
StpUtil.disable(10001, 86400); 
// 校验指定账号是否已被封禁，如果被封禁则抛出异常 `DisableServiceException`
StpUtil.checkDisable(10001); 
// 解除封禁
StpUtil.untieDisable(10001); 
```

#### 分类封禁

只禁止其访问部分服务，即禁止部分接口的调用。

```java
// 封禁指定 用户 评论 能力，期限为 1天
StpUtil.disable(10001, "comment", 86400);

// 在评论接口，校验一下，会抛出异常：`DisableServiceException`，使用 e.getService() 可获取业务标识 `comment` 
StpUtil.checkDisable(10001, "comment");
```

#### 阶梯封禁

多次违规，封禁时间不一样，或者不同的违规行为，处罚力度不一样：

```java
// 阶梯封禁，参数：封禁账号、封禁级别、封禁时间 
StpUtil.disableLevel(10001, 3, 10000);

// 获取：指定账号封禁的级别 （如果此账号未被封禁则返回 -2）
StpUtil.getDisableLevel(10001);

// 判断：指定账号是否已被封禁到指定级别，返回 true 或 false
StpUtil.isDisableLevel(10001, 3);

// 校验：指定账号是否已被封禁到指定级别，如果已达到此级别（例如已被3级封禁，这里校验是否达到2级），则抛出异常 `DisableServiceException`
StpUtil.checkDisableLevel(10001, 2);
```

### 无Cookie模式

> 无 Cookie 模式：特指不支持 Cookie 功能的终端，通俗来讲就是我们常说的 —— **前后台分离模式**，如**app、小程序**。
>
> - 支持配置从`body`、`header`、`cookie`中获取 token value；

解决方案：

- 不能后端控制写入了，就前端自己写入。（难点在**后端如何将 Token 传递到前端**）
- 每次请求不能自动提交了，那就手动提交。（难点在**前端如何将 Token 传递到后端**，同时**后端将其读取出来**）

#### 后端如何将 Token 传递到前端

1. 调用 `StpUtil.login(id)` 进行登录。
2. 调用 `StpUtil.getTokenInfo()` 返回当前会话的 token 详细参数，前端人员将`tokenName`和`tokenValue`这两个值保存到本地。

#### 前端将 token 提交到后端

将 token 塞到请求`header`里 ，格式为：`{tokenName: tokenValue}`：

- 使用 统一拦截器，注入该header字段；



### 模拟他人和身份切换

#### 操作其它账号的api

`StpUtil`提供一系列方法指定 loginId 进行操作。

```java
// 获取指定账号10001的`tokenValue`值 
StpUtil.getTokenValueByLoginId(10001);
```



### 临时身份切换

```java
// 将当前会话[身份临时切换]为其它账号（本次请求内有效）
StpUtil.switchTo(10044);

// 此时再调用此方法会返回 10044 (我们临时切换到的账号id)
StpUtil.getLoginId();

// 结束 [身份临时切换]
StpUtil.endSwitch();
```



### 临时Token令牌认证

**[sa-token-temp临时认证模块]** 已内嵌到核心包，无需引入其它依赖即可使用

- SaTempUtil 创建**临时token后登陆的用户id与SaToken框架中StpUtil登陆的用户id并不关联**，两者相互独立同时存在

```java
// 根据 value 创建一个 token 
String token = SaTempUtil.createToken("10014", 200);

// 解析 token 获取 value，并转换为指定类型 
String value = SaTempUtil.parseToken(token, String.class);

// 获取指定 token 的剩余有效期，单位：秒 
SaTempUtil.getTimeout(token);

// 删除指定 token
SaTempUtil.deleteToken(token);
```



## 数据存储

Sa-Token 数据存储有三大作用域，分别是：

- `SaStorage` - 请求作用域：存储的数据只在一次请求内有效，无需处于登录状态；
- `SaSession` - 会话作用域：存储的数据在一次会话范围内有效，必须登录后；
- `SaApplication` - 全局作用域：存储的数据在全局范围内有效，无需处于登录状态。
  - 在 SaApplication 存储的数据在全局范围内有效，应用关闭后数据自动清除（如果集成了 Redis 那则是 Redis 关闭后数据自动清除）。

```java
SaStorage storage = SaHolder.getStorage();
SaSession session = StpUtil.getSession();
SaApplication application = SaHolder.getApplication();
```

### SaStorage

在 SaStorage 中存储的数据只**在一次请求范围内有效**，请求结束后数据自动清除。使用 SaStorage 时无需处于登录状态：

- 通过 HttpServletRequest 的 `get/set`方法；

### Session会话

Session是会话中专业的数据缓存组件，通过 Session 我们可以很方便的**缓存一些高频读写数据**，提高程序性能，例如：

```java
// 在登录时缓存user对象 
StpUtil.getSession().set("user", user);

// 然后我们就可以在任意处使用这个user对象
SysUser user = (SysUser) StpUtil.getSession().get("user");
```

在 Sa-Token 中，Session 分为三种，分别是：

- `User-Session`: 指的是框架为每个 账号id 分配的 Session

- `Token-Session`: 指的是框架为每个 token 分配的 Session

- `Custom-Session`: 指的是以一个 特定的值 作为SessionId，来分配的 Session

  session的操作

```java
session.get("name");
session.set("name", "zhang"); 
// 返回此 Session 会话上的底层数据对象（如果更新map里的值，请调用session.update()方法避免产生脏数据）
session.getDataMap();   
// 将这个 Session 从持久库更新一下
session.update();  
// 注销此 Session 会话 (从持久库删除此Session)
session.logout();      
```



#### 与HTTP Session区别

> SaSession 与 HttpSession 没有任何关系，在`HttpSession`上写入的值，在`SaSession`中无法取出

Servlet HttpSession的原理：由相应的 Servlet Container 实现（如 Tomcat等）

- 客户端每次与服务器第一次握手时，会被**强制分配一个 `[唯一id]` 作为身份标识**，注入到 Cookie 之中， 之后每次发起请求时，客户端都要将它提交到后台，服务器根据 `[唯一id]` 找到每个请求专属的Session对象，维持会话；

  Servlet HttpSession的缺点：

- 同一账号分别在PC、APP登录，会被识别为**两个不相干的会话**

- 一个设备难以**同时登录两个账号**

- 每次一个新的客户端访问服务器时，都会**产生一个新的Session对象**，即使这个客户端只访问了一次页面

- 在**不支持Cookie**的客户端下，这种机制会失效



> 为账号id分配的Session，叫做 `User-Session`

**Sa-Token Session可以理解为 HttpSession 的升级版：**

- 只在调用`StpUtil.login(id)`**登录会话**时才会产生Session，节省性能

- **Session 是分配给账号id**的，而不是分配给指定客户端的，也就是说在PC、APP上登录的同一账号所得到的Session也是同一个；

- Sa-Token支持Cookie、Header、body三个途径提交Token

- 做到**一个客户端同时登录多个账号**

  会遇到一些需要数据隔离的场景：

> 指定客户端超过两小时无操作就自动下线，如果两小时内有操作，就再续期两小时，直到新的两小时无操作。
>
> - 放在 User-Session，则用户在PC端一直无操作，只要手机上用户还在不间断的操作，那PC端也不会过期！

Sa-Token针对会话登录，不仅为账号id分配了`User-Session`，同时还为每个token分配了不同的`Token-Session`

- 虽然两个设备登录的是同一账号，但是两个它们得到的token是不一样

#### User-Session

```java
// 获取当前账号id的Session (必须是登录后才能调用)
StpUtil.getSession();
// 获取账号id为10001的Session
StpUtil.getSessionByLoginId(10001);
```



#### Token Session

> 默认场景下，**只有登录后才能获取**，在未登录场景下获取 Token-Session ，有两种方法：
>
> 方法一：将全局配置项 tokenSessionCheckLogin 改为 false；
> 方法二：使用匿名 Token-Session；

```java
// 获取当前 Token 的 Token-Session 对象
StpUtil.getTokenSession();
// 获取指定 Token 的 Token-Session 对象
StpUtil.getTokenSessionByToken(token);
// 获取当前 Token 的匿名 Token-Session （可在未登录情况下使用的 Token-Session）
StpUtil.getAnonTokenSession();
```



#### 自定义Session

以一个`特定的值`作为SessionId来分配的`Session`

```java
// 查询指定key的Session是否存在
SaSessionCustomUtil.isExists("goods-10001");
// 获取指定key的Session，如果没有，则新建并返回
SaSessionCustomUtil.getSessionById("goods-10001");
// 删除指定key的Session
SaSessionCustomUtil.deleteSessionById("goods-10001");

```



#### 分布式Session

> Sa-Token 默认将数据保存在内存中，但重启后数据会丢失，且无法在分布式环境中共享数据。



目前的主流方案有四种：

1. **Session同步**：只要一个节点的数据发生了改变，就强制同步到其它所有节点（性能消耗太大，不太考虑）

2. **Session粘滞**：通过一定的算法，保证一个用户的所有请求都稳定的落在一个节点之上（从网关处动手，与框架无关）

3. **建立会话中心**：将**Session存储在专业的缓存中间件**上，使每个节点都变成了无状态服务，例如：`Redis`

4. **颁发无状态token**：放弃Session机制，**将用户数据直接写入到令牌本身上，使会话数据做到令牌自解释**，例如：`jwt`

   由于**`jwt`模式不在服务端存储数据**，对于比较复杂的业务可能会功能受限，因此更加推荐使用方案三

   集成Redis：

```xml
<!-- Sa-Token 整合 Redis （使用 jdk 默认序列化方式， 2选1 ） -->
<dependency>
    <groupId>cn.dev33</groupId>
    <artifactId>sa-token-dao-redis</artifactId>
    <version>1.33.0</version>
</dependency>

<!-- Sa-Token 整合 Redis （使用 jackson 序列化方式， 2选1 ） -->
<dependency>
    <groupId>cn.dev33</groupId>
    <artifactId>sa-token-dao-redis-jackson</artifactId>
    <version>1.33.0</version>
</dependency>

<!-- 提供Redis连接池 -->
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
</dependency>
```

application.yml 配置

```yaml
spring: 
    # redis配置 
    redis:
        # Redis数据库索引（默认为0）
        database: 1
        # Redis服务器地址
        host: 127.0.0.1
        # Redis服务器连接端口
        port: 6379
        # Redis服务器连接密码（默认为空）
        # password: 
        # 连接超时时间
        timeout: 10s
        lettuce:
            pool:
                # 连接池最大连接数
                max-active: 200
                # 连接池最大阻塞等待时间（使用负值表示没有限制）
                max-wait: -1ms
                # 连接池中的最大空闲连接
                max-idle: 10
                # 连接池中的最小空闲连接
                min-idle: 0
```



### SaApplication

在 SaApplication 存储的数据在全局范围内有效，应用关闭后数据自动清除（如果集成了 Redis 那则是 Redis 关闭后数据自动清除）。使用 SaApplication 时无需处于登录状态。

```java
SaApplication application = SaHolder.getApplication();
application.get("key");   // 取值
application.set("key", "value");   // 写值 
application.delete("key");   // 删值 
```



### SaTokenDao的实现

SaTokenDao 是数据持久层接口，负责所有会话数据的底层写入和读取，存储的内容

- Session 本身的 set 的值，不推荐使用，不同的微服务不建议使用其进行数据共享；

- `tokenValue`到`loginId`的映射关系；

  - 标记该token是否注销、强制下线等场景；

    Sa-Token默认的Redis集成方式会把**权限数据和业务缓存（spring redis template）**放在一起，Sa-Token-Alone-Redis 仅对以下插件有 Redis 分离效果：

- sa-token-dao-redis ：Redis集成包，使用 jdk 默认序列化方式。

- sa-token-dao-redis-jackson ：Redis集成包，使用 jackson 序列化方式。

- sa-token-dao-redis-fastjson ：Redis集成包，使用 fastjson 序列化方式。

- sa-token-dao-redis-fastjson2 ：Redis集成包，使用 fastjson2 序列化方式。

```xml
<!-- Sa-Token插件：权限缓存与业务缓存分离 -->
<dependency>
    <groupId>cn.dev33</groupId>
    <artifactId>sa-token-alone-redis</artifactId>
    <version>1.33.0</version>
</dependency>
```

application.yml中增加配置

```yaml
# Sa-Token 配置
sa-token: 
    # Token名称
    token-name: satoken
    # Token有效期
    timeout: 2592000
    # Token风格
    token-style: uuid

    # 配置 Sa-Token 单独使用的 Redis 连接 
    alone-redis: 
        # Redis数据库索引（默认为0）
        database: 2
        # Redis服务器地址
        host: 127.0.0.1
        # Redis服务器连接端口
        port: 6379
        # Redis服务器连接密码（默认为空）
        password: 
        # 连接超时时间
        timeout: 10s
spring: 
    # 配置业务使用的 Redis 连接 
    redis: 
        # Redis数据库索引（默认为0）
        database: 0
        # Redis服务器地址
        host: 127.0.0.1
        # Redis服务器连接端口
        port: 6379
        # Redis服务器连接密码（默认为空）
        password: 
        # 连接超时时间
        timeout: 10s
```



### JWT 集成

```xml
<!-- Sa-Token 整合 jwt 显式依赖 hutool-jwt 5.7.14 版本 -->
<dependency>
    <groupId>cn.dev33</groupId>
    <artifactId>sa-token-jwt</artifactId>
    <version>1.33.0</version>
</dependency>
```

在 `application.yml` 配置文件中配置 jwt 生成秘钥：

```yaml
sa-token:
    # jwt秘钥 
    jwt-secret-key: asdasdasifhueuiwyurfewbfjsdafjk
```

注入：

```java
@Configuration
public class SaTokenConfigure {
    @Bean
    public StpLogic getStpLogicJwt() {
    	// Sa-Token 整合 jwt (Simple 简单模式)
        return new StpLogicJwtForSimple();
    	// Sa-Token 整合 jwt (Mixin 混入模式)
        return new StpLogicJwtForMixin();
    	// Sa-Token 整合 jwt (Stateless 无状态模式)
        return new StpLogicJwtForStateless();
    }
}
```

#### 注入不同模式的对比

注入不同模式会让框架具有不同的行为策略，以下是三种模式的差异点（为方便叙述，以下比较以**同时引入 jwt 与 Redis** 作为前提）：

| 功能点                  | Simple 简单模式 | Mixin 混入模式        | Stateless 无状态模式   |
| ----------------------- | --------------- | --------------------- | ---------------------- |
| Token风格               | jwt风格         | jwt风格               | jwt风格                |
| **登录数据存储**        | Redis中         | Token中               | Token中                |
| Session存储             | Redis中         | Redis中               | **无Session**          |
| 注销下线                | 前后端双清数据  | 前后端双清数据        | **前端清除数据**       |
| 踢人下线API             | 支持            | **不支持**            | **不支持**             |
| 顶人下线API             | 支持            | **不支持**            | **不支持**             |
| 登录认证                | 支持            | 支持                  | 支持                   |
| 角色认证                | 支持            | 支持                  | 支持                   |
| 权限认证                | 支持            | 支持                  | 支持                   |
| timeout 有效期          | 支持            | 支持                  | 支持                   |
| activity-timeout 有效期 | 支持            | 支持                  | **不支持**             |
| id反查Token             | 支持            | 支持                  | **不支持**             |
| 会话管理                | 支持            | **部分支持**          | **不支持**             |
| 注解鉴权                | 支持            | 支持                  | 支持                   |
| 路由拦截鉴权            | 支持            | 支持                  | 支持                   |
| 账号封禁                | 支持            | 支持                  | **不支持**             |
| 身份切换                | 支持            | 支持                  | 支持                   |
| 二级认证                | 支持            | 支持                  | 支持                   |
| 模式总结                | Token风格替换   | jwt 与 Redis 逻辑混合 | 完全舍弃Redis，只用jwt |

#### 在多账户模式中集成 jwt

sa-token-jwt 插件默认只为 `StpUtil` 注入 `StpLogicJwtFoxXxx` 实现，自定义的 `StpUserUtil` 是不会自动注入的，我们需要帮其手动注入：

```java
/**
 * 为 StpUserUtil 注入 StpLogicJwt 实现 
 */
@Autowired
public void setUserStpLogic() {
    StpUserUtil.setStpLogic(new StpLogicJwtForSimple(StpUserUtil.TYPE));
}
```

### SaTokenContext的实现

> 目前 Sa-Token 仅对 SpringBoot、SpringMVC、WebFlux、Solon 等部分 Web 框架制作了 Starter 集成包， 如果我们使用的 Web 框架不在上述列表之中，则需要自定义 SaTokenContext 接口的实现完成整合工作。

从 `HttpServletRequest` 中读取 Token，但不是所有框架都具有 HttpServletRequest 对象，例如在 WebFlux 中，只有 `ServerHttpRequest`， 在一些其它Web框架中，可能连 `Request` 的概念都没有。

- `SaTokenContext`定义接口，`sa-token-spring-boot-starter`集成包中已经内置了`SaTokenContext`的实现：[SaTokenContextForSpring](https://gitee.com/dromara/sa-token/blob/master/sa-token-starter/sa-token-spring-boot-starter/src/main/java/cn/dev33/satoken/spring/SaTokenContextForSpring.java);
- 其它框架，需要自定义实现`SaTokenContext`。



## 权限

### 权限认证

#### 接口定义

`StpInterface` 提供获取角色和获取权限的接口定义，根据自己的业务逻辑进行重写。

- `loginType`字段见**`多账户认证`**部分；

```java
public interface StpInterface {
   /** 返回指定账号账号类型的id的所拥有的权限码集合 */
   public List<String> getPermissionList(Object loginId, String loginType);
   /** 返回指定账号账号类型的id所拥有的角色标识集合 */
   public List<String> getRoleList(Object loginId, String loginType);
}
```

#### 权限/角色校验

`StpUtil`提供工具，也可通过注解+拦截器实现；

```java
// 获取：当前账号所拥有的权限集合
StpUtil.getPermissionList();

// 判断：当前账号是否含有指定权限, 返回 true 或 false
StpUtil.hasPermission("user.add");        

// 校验：当前账号是否含有指定权限, 如果验证未通过，则抛出异常: NotPermissionException 
StpUtil.checkPermission("user.add");        

// 校验：当前账号是否含有指定权限 [指定多个，必须全部验证通过]
StpUtil.checkPermissionAnd("user.add", "user.delete", "user.get");        

// 校验：当前账号是否含有指定权限 [指定多个，只要其一验证通过即可]
StpUtil.checkPermissionOr("user.add", "user.delete", "user.get");    

// 获取：当前账号所拥有的角色集合
StpUtil.getRoleList();

// 判断：当前账号是否拥有指定角色, 返回 true 或 false
StpUtil.hasRole("super-admin");        

// 校验：当前账号是否含有指定角色标识, 如果验证未通过，则抛出异常: NotRoleException
StpUtil.checkRole("super-admin");        

// 校验：当前账号是否含有指定角色标识 [指定多个，必须全部验证通过]
StpUtil.checkRoleAnd("super-admin", "shop-admin");        

// 校验：当前账号是否含有指定角色标识 [指定多个，只要其一验证通过即可] 
StpUtil.checkRoleOr("super-admin", "shop-admin");        
```

### 二级认证

在某些敏感操作下，我们需要对已登录的会话进行二次验证。即：**在已登录会话的基础上，进行再次验证，提高会话的安全性**。

`Sa-Token`中进行二级认证非常简单，只需要使用以下API：

- 且可以指定一个**业务标识**来分辨不同的业务线：

```java
// 在当前会话 开启二级认证，业务标识为client，时间为600秒
StpUtil.openSafe("client", 600); 

// 获取：当前会话是否已完成指定业务的二级认证 
StpUtil.isSafe("client"); 

// 校验：当前会话是否已完成指定业务的二级认证 ，如未认证则抛出异常
StpUtil.checkSafe("client"); 

// 获取当前会话指定业务二级认证剩余有效时间 (单位: 秒, 返回-2代表尚未通过二级认证)
StpUtil.getSafeTime("client"); 

// 在当前会话 结束指定业务标识的二级认证
StpUtil.closeSafe("client"); 
```

调用步骤：

1. 前端调用 `deleteProject` 接口，尝试删除仓库。
2. 后端校验会话尚未完成二级认证，返回： `仓库删除失败，请完成二级认证后再次访问接口`。
3. 前端将信息提示给用户，用户输入密码，调用 `openSafe` 接口。
4. 后端比对用户输入的密码，完成二级认证，有效期为：120秒。
5. 前端在 120 秒内再次调用 `deleteProject` 接口，尝试删除仓库。
6. 后端校验会话已完成二级认证，返回：`仓库删除成功`。



### 单服务路由拦截鉴权

基于 `StpUtil.checkLogin()` 的登录校验拦截器，并且排除了`/user/doLogin`接口用来开放登录：

- 内部通过`SaHolder`获取 `request`信息，进行路由匹配；
- 更多用法参考：https://sa-token.dev33.cn/doc.html#/use/route-check

```java
@Configuration
public class SaTokenConfigure implements WebMvcConfigurer {
    // 注册拦截器
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // 注册 Sa-Token 拦截器
        registry.addInterceptor(new SaInterceptor(handle -> {
              // 指定一条 match 规则
            SaRouter
                .match("/**")    						// 拦截的 path 列表，可以写多个 */
                .notMatch("/user/doLogin")        		// 排除掉的 path 列表，可以写多个 
                .check(r -> StpUtil.checkLogin());      // 要执行的校验动作，可以写完整的 lambda 表达式

            // 根据路由划分模块，不同模块不同鉴权 
            SaRouter.match("/user/**", r -> StpUtil.checkPermission("user"));
            SaRouter.match("/admin/**", r -> StpUtil.checkPermission("admin"));
        })).addPathPatterns("/**");
    }
}
```



### 网关统一鉴权

以`[SpringCloud Gateway]`为例

```xml
<!-- Sa-Token 权限认证（Reactor响应式集成）, 在线文档：https://sa-token.cc -->
<dependency>
    <groupId>cn.dev33</groupId>
    <artifactId>sa-token-reactor-spring-boot-starter</artifactId>
    <version>1.33.0</version>
</dependency>

<!-- Sa-Token 整合 Redis （使用 jackson 序列化方式） -->
<dependency>
    <groupId>cn.dev33</groupId>
    <artifactId>sa-token-dao-redis-jackson</artifactId>
    <version>1.33.0</version>
</dependency>
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
</dependency>
```

#### 实现鉴权接口

关于数据的获取，建议以下方案三选一：

1. 在网关处集成ORM框架，直接从**数据库查询数据**
2. 先从**Redis中获取数据**，获取不到时走**ORM框架**查询数据库
3. 先从**Redis中获取缓存数据，获取不到时走RPC调用子服务 (专门的权限数据提供服务)** 获取

```java
/**
 * 自定义权限验证接口扩展 
 */
@Component   
public class StpInterfaceImpl implements StpInterface {

    @Override
    public List<String> getPermissionList(Object loginId, String loginType) {
        // 返回此 loginId 拥有的权限列表 
        return ...;
    }

    @Override
    public List<String> getRoleList(Object loginId, String loginType) {
        // 返回此 loginId 拥有的角色列表
        return ...;
    }
}
```

#### 注册全局过滤器

```java
@Configuration
public class SaTokenConfigure {
    // 注册 Sa-Token全局过滤器 
    @Bean
    public SaReactorFilter getSaReactorFilter() {
        return new SaReactorFilter()
            // 拦截全部path
            .addInclude("/**")
            // 开放地址 
            .addExclude("/favicon.ico")
            // 鉴权方法：每次访问进入 
            .setAuth(obj -> {
                // 登录校验 -- 拦截所有路由，并排除/user/doLogin 用于开放登录 
                SaRouter.match("/**", "/user/doLogin", r -> StpUtil.checkLogin());
                // 权限认证 -- 不同模块, 校验不同权限 
                SaRouter.match("/user/**", r -> StpUtil.checkPermission("user"));
                SaRouter.match("/admin/**", r -> StpUtil.checkPermission("admin"));
                SaRouter.match("/goods/**", r -> StpUtil.checkPermission("goods"));
                SaRouter.match("/orders/**", r -> StpUtil.checkPermission("orders"));

                // 更多匹配 ...  */
            })
            // 异常处理方法：每次setAuth函数出现异常时进入 
            .setError(e -> {
                return SaResult.error(e.getMessage());
            });
    }
}
```







## 单点登录（TODO）







## 其它



### 全局监听器

 提供一种侦听器机制，通过注册侦听器，你可以订阅框架的一些关键性事件，例如：用户登录、退出、被踢下线等。

- 同步调用，即在用户登录的函数处理过程中调用监听器的事件处理；
- 进程级别，不提供分布式场景下的监听处理；



### 工具类

https://sa-token.dev33.cn/doc.html#/more/common-action



## 配置



## FAQ

### 依赖引入说明

`sa-token-spring-boot-starter` 

- 内部基础服务一般使用SpringBoot默认的web模块：SpringMVC， 基于Servlet模型的，引入`sa-token-spring-boot-starter`

  `sa-token-reactor-spring-boot-starter`

- `SpringCloud Gateway、ShenYu` 等基于Reactor模型引入的是：`sa-token-reactor-spring-boot-starter`



### 反向代理 uri 丢失的问题

使用 `request.getRequestURL()` 可获取当前程序所在外网的访问地址，在 Sa-Token 中，其 `SaHolder.getRequest().getUrl()` 也正是借助此API完成， 有很多模块都用到了这个能力，比如SSO单点登录。

- 部署时使用 Nginx 做了一层反向代理后，获取到的 Url 路径可能会出问题；

  https://sa-token.dev33.cn/doc.html#/fun/curr-domain