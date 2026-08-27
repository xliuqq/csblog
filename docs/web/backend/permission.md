# 认证授权

## RBAC 实践

> [[译\] 基于角色的访问控制（RBAC）：演进历史、设计理念及简洁实现（Tailscale, 2021） (arthurchiao.art)](https://arthurchiao.art/blog/rbac-as-it-meant-to-be-zh/)

根据**用户类型**（user types）而非 **文件类型**（file types）来创建 user groups

**还没有谁实现过完整的 RBAC 模型**：

1. 每个人都是一个 User (subject)。

2. 每个 user 都有一个或多个 Roles。

3. 每个 object（resource） 都有一个或多个 Tags。

4. 一条 “security policy” **定义一个**将 `(Role, Tag)` 转换成 Entitlements 的 **公式**。

5. 一个执行层（enforcement layer）负责 enforce security policy，并为每个 object 生成有效 entitlements 列表（ACL）。

   最终的产品在**理念设计上很简洁**：

6. Device 或 container 的 owner 可以设置 tag；

7. 安全团队决定谁 own 哪些 tag、每个 tag 关联了哪些 permissions、tags 会授权给哪些 roles；

8. Identity/HR 团队决定哪些 users 应该属于哪些 roles。



## 鉴权配置

如何在接口信息上，生成**接口（数据）权限**信息

- 接口权限**通过注解**，在Controller的方法上，每个服务共用同样的拦截器统一鉴权：
- 接口权限**通过数据库**，**仅在网关**上单独拦截器统一鉴权：
  - 启动时，从数据库获取所有接口权限；



## 菜单权限

页面从形式上，分为 `目录`，`菜单`，`按钮`三种

- 目录：一般来说，目录不关联具体的页面（即无法点击到页面），包含菜单或者子目录；
  - 目录也可以跳转到相应的路由地址（外链情形）；
- 菜单：可点击跳转到对应的路由页面
  - 可对应**权限标识**
- 按钮
  - 按钮对应相关的权限标识

接口对应相应的按钮

- 查询：可能不提供按钮，点击页面时默认查询；（一般来说，查询不做接口权限验证）
- 修改和删除等：具体的按钮操作；

可能会出现接口 A 在内部调用其他服务 接口，因此服务内部间不需要接口权限的验证：

- 具备接口A的权限，则默认其所有调用的子接口都应该具备访问的权限；
- 每个服务单独进行接口鉴权：
  - 服务对外接口采用REST并配置拦截鉴权；
  - 服务间调用使用Dubbo等定义新的接口，不进行鉴权；
- 统一网关处进行接口鉴权：
  - 服务内部不需要配置拦截鉴权，内部通信直连；

## DB 用户/组织/菜单 

### 部门（组织）

```sql
-- ----------------------------
-- 1、部门表
-- ----------------------------
drop table if exists sys_dept;
create table sys_dept (
  dept_id           bigint(20)      not null                   comment '部门id',
  parent_id         bigint(20)      default 0                  comment '父部门id',
  ancestors         varchar(500)    default ''                 comment '祖级列表',
  dept_name         varchar(30)     default ''                 comment '部门名称',
  order_num         int(4)          default 0                  comment '显示顺序',
  leader            varchar(20)     default null               comment '负责人',
  phone             varchar(11)     default null               comment '联系电话',
  email             varchar(50)     default null               comment '邮箱',
  status            char(1)         default '0'                comment '部门状态（0正常 1停用）',
  del_flag          char(1)         default '0'                comment '删除标志（0代表存在 2代表删除）',
  create_by         varchar(64)     default ''                 comment '创建者',
  create_time 	    datetime                                   comment '创建时间',
  update_by         varchar(64)     default ''                 comment '更新者',
  update_time       datetime                                   comment '更新时间',
  primary key (dept_id)
) engine=innodb comment = '部门表';
```

### 用户

```sql
-- ----------------------------
-- 2、用户信息表
-- ----------------------------
create table sys_user (
  user_id           bigint(20)      not null                   comment '用户ID',
  dept_id           bigint(20)      default null               comment '部门ID',
  user_name         varchar(30)     not null                   comment '用户账号',
  nick_name         varchar(30)     not null                   comment '用户昵称',
  user_type         varchar(10)     default 'sys_user'         comment '用户类型（sys_user系统用户）',
  email             varchar(50)     default ''                 comment '用户邮箱',
  phone_number       varchar(11)     default ''                comment '手机号码',
  sex               char(1)         default '0'                comment '用户性别（0男 1女 2未知）',
  avatar            varchar(100)    default ''                 comment '头像地址',
  password          varchar(100)    default ''                 comment '密码',
  status            char(1)         default '0'                comment '帐号状态（0正常 1停用）',
  login_date        datetime                                   comment '最后登录时间',
  create_by         varchar(64)     default ''                 comment '创建者',
  create_time       datetime                                   comment '创建时间',
  update_by         varchar(64)     default ''                 comment '更新者',
  update_time       datetime                                   comment '更新时间',
  remark            varchar(500)    default null               comment '备注',
  primary key (user_id)
) engine=innodb comment = '用户信息表';
```

### 岗位（可选）

```sql
-- ----------------------------
-- 3、岗位信息表
-- ----------------------------
drop table if exists sys_post;
create table sys_post
(
  post_id       bigint(20)      not null                   comment '岗位ID',
  post_code     varchar(64)     not null                   comment '岗位编码',
  post_name     varchar(50)     not null                   comment '岗位名称',
  post_sort     int(4)          not null                   comment '显示顺序',
  status        char(1)         not null                   comment '状态（0正常 1停用）',
  create_by     varchar(64)     default ''                 comment '创建者',
  create_time   datetime                                   comment '创建时间',
  update_by     varchar(64)     default ''			       comment '更新者',
  update_time   datetime                                   comment '更新时间',
  remark        varchar(500)    default null               comment '备注',
  primary key (post_id)
) engine=innodb comment = '岗位信息表';
```

### 角色

```sql
-- ----------------------------
-- 4、角色信息表
-- ----------------------------
drop table if exists sys_role;
create table sys_role (
  role_id              bigint(20)      not null                   comment '角色ID',
  role_name            varchar(30)     not null                   comment '角色名称',
  role_key             varchar(100)    not null                   comment '角色权限字符串',
  role_sort            int(4)          not null                   comment '显示顺序',
  data_scope           char(1)         default '1'                comment '数据范围（1：全部数据权限 2：自定数据权限 3：本部门数据权限 4：本部门及以下数据权限）',
  menu_check_strictly  tinyint(1)      default 1                  comment '菜单树选择项是否关联显示',
  dept_check_strictly  tinyint(1)      default 1                  comment '部门树选择项是否关联显示',
  status               char(1)         not null                   comment '角色状态（0正常 1停用）',
  create_by            varchar(64)     default ''                 comment '创建者',
  create_time          datetime                                   comment '创建时间',
  update_by            varchar(64)     default ''                 comment '更新者',
  update_time          datetime                                   comment '更新时间',
  remark               varchar(500)    default null               comment '备注',
  primary key (role_id)
) engine=innodb comment = '角色信息表';
```

### 菜单权限

```sql
-- ----------------------------
-- 5、菜单权限表
-- ----------------------------
create table sys_menu (
  menu_id           bigint(20)      not null                   comment '菜单ID',
  menu_name         varchar(50)     not null                   comment '菜单名称',
  parent_id         bigint(20)      default 0                  comment '父菜单ID',
  order_num         int(4)          default 0                  comment '显示顺序',
  path              varchar(200)    default ''                 comment '路由地址',
  component         varchar(255)    default null               comment '组件路径',
  query_param       varchar(255)    default null               comment '路由参数',
  is_frame          int(1)          default 1                  comment '是否为外链（0是 1否）',
  is_cache          int(1)          default 0                  comment '是否缓存（0缓存 1不缓存）',
  menu_type         char(1)         default ''                 comment '菜单类型（M目录 C菜单 F按钮）',
  visible           char(1)         default 0                  comment '显示状态（0显示 1隐藏）',
  status            char(1)         default 0                  comment '菜单状态（0正常 1停用）',
  perms             varchar(100)    default null               comment '权限标识',
  icon              varchar(100)    default '#'                comment '菜单图标',
  create_by         varchar(64)     default ''                 comment '创建者',
  create_time       datetime                                   comment '创建时间',
  update_by         varchar(64)     default ''                 comment '更新者',
  update_time       datetime                                   comment '更新时间',
  remark            varchar(500)    default ''                 comment '备注',
  primary key (menu_id)
) engine=innodb comment = '菜单权限表';
```

### 角色菜单关联表

```sql
-- ----------------------------
-- 7、角色和菜单关联表  角色1-N菜单
-- ----------------------------
create table sys_role_menu (
  role_id   bigint(20) not null comment '角色ID',
  menu_id   bigint(20) not null comment '菜单ID',
  primary key(role_id, menu_id)
) engine=innodb comment = '角色和菜单关联表';
```

### 用户角色关联表

```sql
-- ----------------------------
-- 6、用户和角色关联表  用户N-1角色
-- ----------------------------
create table sys_user_role (
  user_id   bigint(20) not null comment '用户ID',
  role_id   bigint(20) not null comment '角色ID',
  primary key(user_id, role_id)
) engine=innodb comment = '用户和角色关联表';
```

### 角色部门关联表

```sql
-- ----------------------------
-- 8、角色和部门关联表  角色1-N部门
-- ----------------------------
create table sys_role_dept (
  role_id   bigint(20) not null comment '角色ID',
  dept_id   bigint(20) not null comment '部门ID',
  primary key(role_id, dept_id)
) engine=innodb comment = '角色和部门关联表';
```

### 用户岗位关联表（可选）

```sql
-- ----------------------------
-- 9、用户与岗位关联表  用户1-N岗位
-- ----------------------------
create table sys_user_post
(
  user_id   bigint(20) not null comment '用户ID',
  post_id   bigint(20) not null comment '岗位ID',
  primary key (user_id, post_id)
) engine=innodb comment = '用户与岗位关联表';
```



## 数据权限

> - 用户能够查看组织下的用户，并进行相应的修改；
> - 当前用户创建的资源，其它人是否能够看到？

数据权限分为：

- 是否能够操作具体的数据；（事前校验）

- 能够查看哪些具体的数据；（事后过滤）

  基于 Mybatis plus interceptor + 自定义注解实现。

- 在进行`Select/Update/Delete`时，对 `where` 语句进行扩充，实现数据权限；

### MybatisPlus 数据权限

mybatis plus 3.5 版本提供`DataPermissionInterceptor`，但官网未提及如何使用，以下是使用示例：

数据权限**主拦截器**：

- 拦截 `query/update/delete` 类型 SQL
- 调用 SQL 解析器解析语句
- 调用 `DataPermissionHandler` 获取权限规则
- 动态拼接权限条件、改写 SQL

```java
@Bean
public MybatisPlusInterceptor mybatisPlusInterceptor() {
    MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
   
    // 添加数据权限插件
    DataPermissionInterceptor dataPermissionInterceptor=new DataPermissionInterceptor();
    MyDataPermissionHandler myDataPermissionHandler=new MyDataPermissionHandler();
    // 添加自定义的数据权限处理器
    dataPermissionInterceptor.setDataPermissionHandler(myDataPermissionHandler);
    interceptor.addInnerInterceptor(dataPermissionInterceptor);
    
    return interceptor;
}

// 继承 DataPermissionHandler，实现 getSqlSegment
// MultiDataPermissionHandler (支持多表的数据权限处理器)
public class MyDataPermissionHandler implements DataPermissionHandler {

    /**
     * @param where             原SQL Where 条件表达式
     * @param mappedStatementId Mapper接口方法ID
     * @return
     */
    @SneakyThrows
    @Override
    public Expression getSqlSegment(Expression where, String mappedStatementId) {
        log.info("=========================== start MyDataPermissionHandler");
        // 1. 模拟获取登录用户，从用户信息中获取部门ID
        Random random = new Random();
        int userDeptId = random.nextInt(9) + 1; // 随机部门ID 1-10 随机数
        Expression expression = null;
        log.info("=============== userDeptId:{}", userDeptId);
        if (userDeptId == DeptEnum.BOOS.getType()) {
            // 2.userDeptId为1，说明是老总，可查看所有数据无需处理
            return where;

        } else if (userDeptId == DeptEnum.MANAGER_02.getType()) {
            // 3. userDeptId为2，说明是02部门经理，可查看02部门及下属部门所有数据
            // 创建IN 表达式
            Set<String> deptIds = Sets.newLinkedHashSet(); // 创建IN范围的元素集合
            deptIds.add("2");
            deptIds.add("3");
            deptIds.add("4");
            deptIds.add("5");
            ItemsList itemsList = new ExpressionList(deptIds.stream().map(StringValue::new).collect(Collectors.toList())); // 把集合转变为JSQLParser需要的元素列表
            InExpression inExpression = new InExpression(new Column("order_tbl.dept_id"), itemsList); //  order_tbl.dept_id IN ('2', '3', '4', '5')
            return new AndExpression(where, inExpression);
        } else if (userDeptId == DeptEnum.MANAGER_06.getType()) {
            // 4. userDeptId为6，说明是06部门经理，可查看06部门及下属部门所有数据
            // 创建IN 表达式
            Set<String> deptIds = Sets.newLinkedHashSet(); // 创建IN范围的元素集合
            deptIds.add("6");
            deptIds.add("7");
            deptIds.add("8");
            deptIds.add("9");
            ItemsList itemsList = new ExpressionList(deptIds.stream().map(StringValue::new).collect(Collectors.toList())); // 把集合转变为JSQLParser需要的元素列表
            InExpression inExpression = new InExpression(new Column("order_tbl.dept_id"), itemsList);
            return new AndExpression(where, inExpression);
        } else {
            // 5. userDeptId为其他时，表示为员工级别没有下属机构，只能查看当前部门的数据
            //  = 表达式
            EqualsTo equalsTo = new EqualsTo(); // order_tbl.dept_id = userDeptId
            equalsTo.setLeftExpression(new Column("order_tbl.dept_id"));
            equalsTo.setRightExpression(new LongValue(userDeptId));
            // 创建 AND 表达式 拼接Where 和 = 表达式
            return new AndExpression(where, equalsTo); // WHERE user_id = 2 AND order_tbl.dept_id = 3
        }
    }
}
```



## 自定义注解实现

> 示例代码：[注解实现数据权限](https://gitee.com/oscsc/web-tech/tree/master/authz/spring-data-permission)

- （Get）查看某个/某多个资源时，是否能够访问？
- （List）列出所有资源时，能够访问哪些？

针对 *Get* 的场景，因为请求体中会携带 *资源 ids*，因此可以进行***PreCheck***。

针对 *List* 的场景，请求体中不包含 *资源 ids*

- **数据库无侵入（*PostFilter*）**：查询出所有资源，再进行过滤；
  - 如果资源很多时，性能有问题；
  - 对于分页查询，无法确定查询数据量，只能全查，否则过滤时的数据不准确；
- **数据库侵入（SqlWhere）**：查询该用户可查看的*资源 ids*，拼接到 SQL 语句后面；

网上的一些数据权限（Ruoyi等），通过内置几种策略来实现，不具备通用性：

- 本人，本部门，部门及所有子部门；
- 针对分享，很难做处理；

因为资源类型有很多，如**”用户“，”用户组“，”项目“和各种业务资源**；

针对管理员：提供特殊权限，不需要将所有*资源 ids* 赋值

-  ***AUTH_ALL***，即 ***"\*"***：指定任意资源 ids，都可以查询；

### 方案

> 由于对于 *List* 的场景，采用 SQL 侵入式的方案。如果没有请求体，则无法将用户具备的资源 ids 通过 SPEL 修改并返回（不打算采用 ThreadLocal 等方式，防止多线程的本地变量继承等问题）。
>
> - 将 多个*Get* 和 *List* 的场景，统一成一个接口；
>   - 如果带*资源 ids* ，则是  多个*Get*；
>   - 如果不带 *资源 ids*（即为 null），则为 *List*；

针对单个*Get* 的场景，因为请求体中会携带 *资源 id*，因此可以进行***PreCheck***：

- 获取用户的所有资源，查看是否具备权限；

- （或）直接查询用户是否具备该资源的权限；

  针对 多个*Get* 的场景，因为请求体中会携带 *资源 id*，因此可以进行***PreCheck***：

- 获取用户的所有资源，查看是否具备权限并过滤有权限的资源；

- （或）直接查询用户是否具备这一批资源的权限；

- 最后，将有权限的*资源ids*，通过 SPEL 回写参数，拼接到SQL；

  针对 *List* 的场景，因为请求体中会携带 *资源 id*，因此可以进行***PreCheck***：

- 获取用户的所有资源，通过 SPEL 回写参数，拼接到SQL；

