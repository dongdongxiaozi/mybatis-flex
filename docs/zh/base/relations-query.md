# 一对多、多对一

在 MyBatis-Flex 中，我们内置了 3 种方式，帮助用户进行关联查询，比如 `一对多`、`一对一`、`多对一`、`多对多`等场景，他们分别是：

- Relations 注解
- Field Query
- Join Query

## Relations 注解

在 MyBatis-Flex 中，提供了 4 个 Relations 注解，他们分别是：

- **RelationOneToOne**：用于一对一的场景
- **RelationOneToMany**：用于一对多的场景
- **RelationManyToOne**：用于多对一的场景
- **RelationManyToMany**：用于多对多的场景

添加了以上配置的实体类，在通过 `BaseMapper` 的方法查询数据时，需要调用 select*****WithRelations**() 方法，Relations 注解才能生效。
否则 MyBatis-Flex 自动忽略 Relations 注解。

## 一对一 `@RelationOneToOne`

假设有一个账户，账户有身份证，账户和身份证的关系是一对一的关系，代码如下所示：

Account.java : 
```java 8
public class Account implements Serializable {

    @Id(keyType = KeyType.Auto)
    private Long id;

    private String userName;

    @RelationOneToOne(selfField = "id", targetField = "accountId")
    private IDCard idCard;
    
    //getter setter
}
```

IDCard.java :

```java 
@Table(value = "tb_idcard")
public class IDCard implements Serializable {

    private Long accountId;
    private String cardNo;
    private String content;
    
    //getter setter
}
```

`@RelationOneToOne` 配置描述：

- selfField 当前实体类的属性
- targetField 目标对象的关系实体类的属性


## 一对多 `@RelationOneToMany`

假设一个账户有很多本书籍，一本书只能归属一个账户所有；账户和书籍的关系是一对多的关系，代码如下：

Account.java :
```java 8
public class Account implements Serializable {

    @Id(keyType = KeyType.Auto)
    private Long id;

    private String userName;

    @RelationOneToMany(selfField = "id", targetField = "accountId")
    private List<Book> books;
    
    //getter setter
}
```

Book.java :

```java 
@Table(value = "tb_book")
public class Book implements Serializable {

    @Id(keyType = KeyType.Auto)
    private Long id;
    private Long accountId;
    private String title;
    
    //getter setter
}
```

`@RelationOneToMany` 配置描述：

- selfField 当前实体类的属性
- targetField 目标对象的关系实体类的属性



## 多对一 `@RelationManyToOne`

假设一个账户有很多本书籍，一本书只能归属一个账户所有；账户和书籍的关系是一对多的关系，书籍和账户的关系为多对一的关系，代码如下：

Account.java 一对多的配置:
```java 8
public class Account implements Serializable {

    @Id(keyType = KeyType.Auto)
    private Long id;

    private String userName;

    @RelationOneToMany(selfField = "id", targetField = "accountId")
    private List<Book> books;
    
    //getter setter
}
```

Book.java 多对一的配置:

```java 9
@Table(value = "tb_book")
public class Book implements Serializable {

    @Id(keyType = KeyType.Auto)
    private Long id;
    private Long accountId;
    private String title;
    
    @RelationManyToOne(selfField = "accountId", targetField = "id")
    private Account account;
    
    //getter setter
}
```

`@RelationOneToMany` 和  `@RelationManyToOne` 配置描述：

- selfField 当前实体类的属性
- targetField 目标对象的关系实体类的属性



## 多对多 `@RelationManyToOne`

假设一个账户可以有多个角色，一个角色也可以有多个账户，他们是多对多的关系，需要通过中间件表 `tb_role_mapping` 来维护：

`tb_role_mapping` 的表结构如下：

```sql
CREATE TABLE  `tb_role_mapping`
(
    `account_id`  INTEGER ,
    `role_id`  INTEGER
);
```

Account.java 多对多的配置:
```java {7-11}
public class Account implements Serializable {

    @Id(keyType = KeyType.Auto)
    private Long id;
    private String userName;

    @RelationManyToMany(
            joinTable = "tb_role_mapping", // 中间表
            selfField = "id", joinSelfColumn = "account_id",
            targetField = "id", joinTargetColumn = "role_id"
    )
    private List<Role> roles;
    
    //getter setter
}
```

Role.java 多对多的配置:

```java {7-11}
@Table(value = "tb_role")
public class Role implements Serializable {

    private Long id;
    private String name;

    @RelationManyToMany(
            joinTable = "tb_role_mapping",
            selfField = "id", joinSelfColumn = "role_id",
            targetField = "id", joinTargetColumn = "account_id"
    )
    private List<Account> accounts;
    
    //getter setter
}
```

`@RelationManyToMany` 配置描述：

- selfField 当前实体类的属性
- targetField 目标对象的关系实体类的属性
- joinTable 中间表
- joinSelfColumn 当前表和中间表的关系字段
- joinTargetColumn 目标表和中间表的关系字段

> 注意：selfField 和 targetField 配置的是类的属性名，joinSelfColumn 和 joinTargetColumn 配置的是中间表的字段名。



## Field Query

以下是文章的 `多对多` 示例，一篇文章可能归属于多个分类，一个分类可能有多篇文章，需要用到中间表 `article_category_mapping`。

```java
public class Article {
    private Long id;
    private String title;
    private String content;

    //文章的归属分类，可能是 1 个或者多个
    private List<Category> categories;

    //getter setter
}
```

查询代码如下：

```java {9-13}
QueryWrapper queryWrapper = QueryWrapper.create()
        .select().from(ARTICLE)
        .where(ARTICLE.id.ge(100));

List<Article> articles = mapper.selectListByQuery(queryWrapper
    , fieldQueryBuilder -> fieldQueryBuilder
        .field(Article::getCategories) // 或者 .field("categories")
        .queryWrapper(article -> QueryWrapper.create()
            .select().from(CATEGORY)
            .where(CATEGORY.id.in(
                    select("category_id").from("article_category_mapping")
                    .where("article_id = ?", article.getId())
            )
        )
    );
```

通过以上代码可以看出，`Article.categories` 字段的结果，来源于  `queryWrapper()` 方法构建的 `QueryWrapper`。

其原理是：MyBatis-Flex 的内部逻辑是先查询出 `Article` 的数据，然后再根据 `Article` 构建出新的 SQL，查询分类，并赋值给 `Article.categories` 属性，
假设 `Article` 有 10 条数据，那么最终会进行 11 次数据库查询。

查询的 SQL 大概如下：

```sql
select * from tb_article where id  >= 100;

-- 以上 SQL 得到结果后，再执行查询分类的 SQL，如下：
select * from tb_category where id in 
(select category_id from article_category_mapping where article_id = 100);

select * from tb_category where id in
(select category_id from article_category_mapping where article_id = 101);

select * from tb_category where id in
(select category_id from article_category_mapping where article_id = 102);

select * from tb_category where id in
(select category_id from article_category_mapping where article_id = 103);

select * from tb_category where id in
(select category_id from article_category_mapping where article_id = 104);

select * from tb_category where id in
(select category_id from article_category_mapping where article_id = 105);

select * from tb_category where id in
(select category_id from article_category_mapping where article_id = 106);

select * from tb_category where id in
(select category_id from article_category_mapping where article_id = 107);

select * from tb_category where id in
(select category_id from article_category_mapping where article_id = 108);

select * from tb_category where id in
(select category_id from article_category_mapping where article_id = 109);
```

## Field Query 知识点

相对 `Relations 注解` ，Field Query 的学习成本是非常低的，在构建子查询时，只需要明白为哪个字段、通过什么样的 SQL 查询就可以了，以下是示例：

```java 3,4
List<Article> articles = mapper.selectListByQuery(query
    , fieldQueryBuilder -> fieldQueryBuilder
        .field(...)        // 为哪个字段查询的？
        .queryWrapper(...) // 通过什么样的 SQL 查询的？
    );
```

因此，在 MyBatis-Flex 的设计中，无论是一对多、多对一、多对多... 还是其他任何一种场景，其逻辑都是一样的。


## Field Query 更多场景

通过以上内容看出，`Article` 的任何属性，都是可以通过传入 `FieldQueryBuilder` 来构建 `QueryWrapper` 进行再次查询，
这些不仅仅只适用于  `一对多`、`一对一`、`多对一`、`多对多`等场景。任何 `Article` 对象里的属性，需要二次查询赋值的，都是可以通过这种方式进行，比如一些统计的场景。


## Join Query 

Join Query 是通过 QueryWrapper 构建 `Left Join` 等方式进行查询，其原理是 MyBatis-Flex 自动构建了 MyBatis 的 `<resultMap>`
，我们只需要关注 MyBatis-Flex 的 SQL 构建即可。

## Join Query 代码示例

这里以 **用户** 和 **角色** 的 `多对多` 关系作为例子，用户表和角色表，分别对应着用户类和角色类：

```java
@Table("sys_user")
public class User {
    @Id
    private Integer userId;
    private String userName;
}


@Table("sys_role")
public class Role {
    @Id
    private Integer roleId;
    private String roleKey;
    private String roleName;
}
```

现在需要查询所有用户，以及每个用户对应的角色信息，并通过 UserVO 对象返回：

```java
public class UserVO {
    private String userId;
    private String userName;
    private List<Role> roleList;
}
```

这个操作只需要联表查询即可完成，对于联表查询的结果映射，MyBatis-Flex 会自动帮您完成：

```java
QueryWrapper queryWrapper = QueryWrapper.create()
        .select(USER.USER_ID, USER.USER_NAME, ROLE.ALL_COLUMNS)
        .from(USER.as("u"))
        .leftJoin(USER_ROLE).as("ur").on(USER_ROLE.USER_ID.eq(USER.USER_ID))
        .leftJoin(ROLE).as("r").on(USER_ROLE.ROLE_ID.eq(ROLE.ROLE_ID));
List<UserVO> userVOS = userMapper.selectListByQueryAs(queryWrapper, UserVO.class);
userVOS.forEach(System.err::println);
```

构建的联表查询 SQL 语句为：

```sql
SELECT `u`.`user_id`,
       `u`.`user_name`,
       `r`.*
FROM `sys_user` AS `u`
LEFT JOIN `sys_user_role` AS `ur` ON `ur`.`user_id` = `u`.`user_id`
LEFT JOIN `sys_role` AS `r` ON `ur`.`role_id` = `r`.`role_id`;
```

最终自动映射的结果为：

```txt
UserVO{userId='1', userName='admin', roleList=[Role{roleId=1, roleKey='admin', roleName='超级管理员'}]}
UserVO{userId='2', userName='ry', roleList=[Role{roleId=2, roleKey='common', roleName='普通角色'}]}
UserVO{userId='3', userName='test', roleList=[Role{roleId=1, roleKey='admin', roleName='超级管理员'}, Role{roleId=2, roleKey='common', roleName='普通角色'}]}
```