
---
title: Blog源码学习
---

# 1.前端

### 1.路由组件

##### 1.懒加载子组件与直接加载父组件（按角色权限加载菜单组件）

### 2.App.vue

- ```java
  1.create()： 1.重新加载用户菜单 /api/admin/user/menus  2.登录的游客信息 /report
  ```

### 3.Login.vue

- ```java
  1.填写完登录信息过后发送请求：/api/login
  2.成功收到信息过后发送请求：/api/admin/user/menus 第一次请求用户菜单信息（并将用户信息存入仓库中）
  3.重定向到: / (前端路由)  
  ```

### 4.Home.vue

- ```javascript
  create():  /api/admin  :   /api/admin/users/area
  ```

### 5.article

##### 1.Article.vue

###### 1.  点击发表按钮

- ```java
  /api/admin/categories/search
  /api/admin/tags/search
  ```

###### 2.上传封面

- ```java
  上传自己的本地文件到服务器上  /admin/articles/images
  ```

###### 3.保存

- ```java
  post :/api/admin/articles  保存文章  删除本地存储的草稿文件：sessionStorage.removeItem("article");
  成功后跳转页面：/article-list 
  ```

##### 2.ArticleList.vue

###### 1.初始化（created）

- ```java
  this.listArticles(); get  /api/admin/articles 
  this.listCategories(); get
  this.listTags(); get
  ```

###### 2.编辑

- ```
  点击编辑时，带着文章参数跳转到 article.vue页面
  ```

### 6.role

##### 1.Role.vue

- ```java
  /api/admin/users/roles   
  /api/admin/role/resources   
  /api/admin/role/menus
  ```

### 7.user

##### 1.Online.vue

- ```java
  /api/admin/users/online  请求在线用户列表
      
  ```

# 2.后端

### 1.无关业务逻辑的类与注解

##### 1.handler

###### 1.WebSecurityHandler

- ```java
  public class WebSecurityHandler implements HandlerInterceptor(拦截浏览器请求)
  //拦截所有请求（对所有方法请求进行限流）  
  ```

###### 2.PageableHandlerInterceptor

- ```java
  public class PageableHandlerInterceptor implements HandlerInterceptor {...}
  PageUtils.setCurrentPage(new Page<>(Long.parseLong(currentPage), Long.parseLong(pageSize)));
  //拦截所有请求（只对含有分页查询的请求进行处理）  
  ```

###### 3.JwtAuthenticationTokenFilter

- ```java
  token过滤器  抛出异常只能由spring框架捕获，全局异常处理不会起作用
  处理方式：添加异常处理 controller 将异常转发给它处理    
  ```

###### 4.FilterInvocationSecurityMetadataSourceImpl

- ```java
  接口拦截规则（在访问接口时触发）
  public Collection<ConfigAttribute> getAttributes(Object object) throws IllegalArgumentException {...}
  //可判断是否为匿名访问
  ```

##### 2.annotation

###### 1.@Valid 

- **一般验证方法参数是否符合自己定义要求** 【通常与注解在对应类中的字段上的**@NotBlank**(message = "xxx")结合使用】

###### 2.OptLogAspect （aspect包下）

- 操作日志的切片处理

- ```java
  @Pointcut("@annotation(com.minzheng.blog.annotation.OptLog)")
  public void optLogPointCut() {}
  //切入点
  ```

- ```java
  @AfterReturning(value = "optLogPointCut()", returning = "keys")
  @SuppressWarnings("unchecked")
  public void saveOptLog(JoinPoint joinPoint, Object keys) {...}
  //方法执行完后，执行该方法
  ```

###### 3.OptLog（annotation包下）

- 操作日志的注解

##### 3.async

###### 1.CompletableFuture

1. 异步编程的接口方法
2. **CompletableFuture.supplyAsync (function）** 设置为异步函数调用

##### 4.exception

###### 1.BizException

- 自定义异常处理类

### 2.登录模块

##### *1.登录流程（过时）（采用了JWT认证模式）*

**1.**用户名和密码被过滤器获取到，封装成`Authentication`,通常情况下是`UsernamePasswordAuthenticationToken`这个实现类。

```
AuthenticationManager`身份管理器负责验证这个`Authentication
```

**2.**认证成功后，`AuthenticationManager`身份管理器返回一个被填充满了信息的（包括上面提到的权限信息，身份信息，细节信息，但密码通常会被移除）`Authentication`实例。

**3.**`SecurityContextHolder`安全上下文容器将第3步填充了信息的`Authentication`，通过`SecurityContextHolder.getContext().setAuthentication()`方法，设置到其中。

```java
//配置中直接接受了登录的url请求，因此直接进行用户信息认证
public class SecurityContextPersistenceFilter extends GenericFilterBean {...}//设置上下文（过时）
1.public class UsernamePasswordAuthenticationFilter extends AbstractAuthenticationProcessingFilter {...}
2.public class UserDetailsServiceImpl implements UserDetailsService {....}//用户信息认证
//（1）1之后底层调用SecurityContextHolder.getContext().setAuthentication()方法
//（2）UserUtils工具类会获取上下文中的用户信息
3.public class AuthenticationSuccessHandlerImpl implements AuthenticationSuccessHandler{...}//用户信息认证成功处理
4.public class AccessDecisionManagerImpl implements AccessDecisionManager{...}//根据用户权限，开放请求
```

##### *2.JWT认证模式*

###### 1.自定义过滤器

```java
public class JwtAuthenticationTokenFilter extends OncePerRequestFilter {}
```

###### 2.取消了两个过滤器

```java
SecurityContextPersistenceFilter 与 UsernamePasswordAuthenticationFilter
```

###### 3.具体的大致流程

```java
1.(自定义认证过滤器)JwtAuthenticationTokenFilter
2.(接口拦截规则)FilterInvocationSecurityMetadataSourceImpl(非白名单访问经过)
3.(用户登录认证)UserDetailsServiceImpl（登录）
4.(管理session)SessionRegistryImpl【SessionRegistry】
    4.1.调用registerNewSession(String sessionId, Object principal)(登录)
    	【根据传入的sessionId（(3)后自动生成）与注册信息注册session对象】
    4.2.调用removeSessionInformation(String sessionId)（手动注销）删除session
5.(登录成功)AuthenticationSuccessHandlerImpl(登录)
```

### 3.Redis存储模块

##### 1.出现Java序列化异常：大概率是存储的类出现无法序列的属性对象

- 例如权限对象：（将其取消序列化即可）

- ```java
  @JSONField(serialize = false)
  List<GrantedAuthority> authorities;
  ```

##### 2.各key值的含义

###### 1.【unique_visitor】：通过识别ip保存登录游客的登录信息的Set集合  每天定时存储与清除

###### 2.【blog_views_count】：用户(游客)访问博客次数（每个游客只记录一次）

###### 3.【login:id】：用户登录token

###### 4.【website_config】：网站配置

###### 5.【visitor_area】： 游客地区信息 【 哈希集合】 

###### 6.【user_area】：用户地区信息

### 4.菜单模块

##### 1.登录成功发送请求

```javascript
1. /admin/user/menus  :获取当前用户状态下的所有菜单选项
```

### 5.网站信息模块

##### 1.接口：/  

- ```json
  //获取该网站的基本信息
  getBlogHomeInfo(){...}
  {
      "alipayQRCode": "https://static.talkxj.com/photos/13d83d77cc1f7e4e0437d7feaf56879f.png",
      "gitee": "",
      "github": "",
      "isChatRoom": 1,
      "isCommentReview": 0,
      "isEmailNotice": 1,
      "isMessageReview": 0,
      "isMusicPlayer": 1,
      "isReward": 1,
      "qq": "",
      "socialLoginList": [
          "qq",
          "weibo"
      ],
      "socialUrlList": [
          "qq",
          "github",
          "gitee"
      ],
      "touristAvatar": "https://static.talkxj.com/photos/0bca52afdb2b9998132355d716390c9f.png",
      "userAvatar": "https://static.talkxj.com/config/2cd793c8744199053323546875655f32.jpg",
      "websiteAuthor": "网站作者",
      "websiteAvatar": "https://static.talkxj.com/config/43a07ac1ca201143f7b938d0791124fc.png",
      "websiteCreateTime": "2019-12-10",
      "websiteIntro": "网站简介",
      "websiteName": "个人博客",
      "websiteNotice": "请前往后台管理->系统管理->网站管理处修改信息",
      "websiteRecordNo": "备案号",
      "websocketUrl": "ws://127.0.0.1:8080/websocket",
      "weiXinQRCode": "https://static.talkxj.com/photos/4f767ef84e55ab9ad42b2d20e51deca1.png"
  }
  ```

##### 2.记录游客接口:/report

- ```
  保存用户的信息的接口
  ```

### 6.HOME模块

##### 1.会接受两个请求

- ```java
  /admin/users/area  ：获取用户或者游客登录的area信息
  /api/admin		   ：获取home界面的可视化必要数据	
  ```

### 7.文章管理模块

##### 1.article

- ```java
  搜索分类列表请求  /api/admin/categories/search 
  搜索标签列表请求  /api/admin/categories/search
  ```

- ```java
  保存封面  
  1.调用 strategy(策略)包下的UploadStrategyContext（上传策略上下文）的executeUploadStrategy方法
  2.调用了UploadStrategy（策略接口）  该接口由抽象类(AbstractUploadStrategyImpl)来实现  
  3.实现类为oss与local模式的上传方法 
  ```

- ```java
  保存文章 /api/admin/articles  （该方法会被记录下来）
  1.异步调用网站相关信息   CompletableFuture.supplyAsync(...)
  2.保存文章分类	      saveArticleCategory(articleVO)
  3.保存或修改文章	     this.saveOrUpdate(article);    		
  4.保存文章标签列表（较为复杂）       saveArticleTag(articleVO, article.getId());
  ```

##### 2.articleList

- ```java
  在后台查看文章列表  /api/admin/articles 
  public Result<PageResult<ArticleBackDTO>> listArticleBacks(ConditionVO conditionVO) {...}
  -->// 查询文章总量  
      articleDao.countArticleBacks(condition); 自定义sql
  -->// 查询后台文章
      articleDao.listArticleBacks(PageUtils.getLimitCurrent(), PageUtils.getSize(), condition); 自定义sql
      
  ```

### 8.角色模块

- ```java
              /api/admin/users/roles   查询用户角色列表    RoleController
              /api/admin/role/resources  查看角色资源选项  ResourceController
  listResourceOption() -->listResourceOption() -->模块与资源的数据整合
      /api/admin/role/menus  查看角色菜单选项  MenuController
  ```

### 9.用户模块

- ```java
  /api/admin/users/online   查询在线用户  listOnlineUsers(){...}
  @其中--->sessionRegistry(sercurity对象)==会获取之前的用户登录而保存的用户(<UserDetailDto>)信息列表（同一会话中）
  ```

# 3.数据库

### 1.菜单mapper

- ```sql
  //根据用户id和角色查询菜单
  SELECT DISTINCT
              m.id,
              `name`,
              `path`,
              component,
              icon,
              is_hidden,
              parent_id,
              order_num
           FROM
              tb_user_role ur
              JOIN tb_role_menu rm ON ur.role_id = rm.role_id
              JOIN `tb_menu` m ON rm.menu_id = m.id
           WHERE
              user_id = #{userInfoId}
  ```

- 联表查询：**tb_user_role**、**tb_role_menu**、**tb_menu**

### 2.文章mapper

- ```sql
  //按天查询文章统计量
  <select id="listArticleStatistics" resultType="com.minzheng.blog.dto.ArticleStatisticsDTO">
          SELECT
              DATE_FORMAT( create_time, "%Y-%m-%d" ) AS date,  //对时间格式化
              COUNT(1) AS count  //注意：这里与count(*)效果相差不大
          FROM
              tb_article
          GROUP BY
              date   //按时间分组计时（按天）
          ORDER BY
              date DESC
  </select>
  ```

- ```sql
  //在后台查询文章列表
  //一表内嵌查询，三表联合查询：tb_article  【tb_category 、tb_article_tag 、tb_tag】
  <select id="listArticleBacks" resultMap="articleBackResultMap">
      SELECT
      a.id,
      article_cover,
      article_title,
      type,
      is_top,
      a.is_delete,
      a.status,
      a.create_time,
      category_name,
      t.id AS tag_id,
      t.tag_name
      FROM
      (
      SELECT
      id,
      article_cover,
      article_title,
      type,
      is_top,
      is_delete,
      status,
      create_time,
      category_id
      FROM
      tb_article
      <where>
          is_delete = #{condition.isDelete}
          <if test="condition.keywords != null">
              and article_title like concat('%',#{condition.keywords},'%')
          </if>
          <if test="condition.status != null">
              and status = #{condition.status}
          </if>
          <if test="condition.categoryId != null">
              and category_id = #{condition.categoryId}
          </if>
          <if test="condition.type != null">
              and type = #{condition.type}
          </if>
          <if test="condition.tagId != null">
              and id in
               (
                SELECT
                  article_id
                FROM
                  tb_article_tag
                WHERE
                  tag_id = #{condition.tagId}
               )
          </if>
      </where>
      ORDER BY
        is_top DESC,
        id DESC
      LIMIT #{current},#{size}
      ) a
      LEFT JOIN tb_category c ON a.category_id = c.id
      LEFT JOIN tb_article_tag atg ON a.id = atg.article_id
      LEFT JOIN tb_tag t ON t.id = atg.tag_id
      ORDER BY
        is_top DESC,
        a.id DESC
  </select>
  ```

### 3.分类mapper

- ```sql
  //  按文章查询所属分类总数
  <select id="listCategoryDTO" resultType="com.minzheng.blog.dto.CategoryDTO">
  		SELECT
  		  c.id,
  		  c.category_name,
  		  COUNT( a.id ) AS article_count   //按文章主键计数
  		FROM
  		  tb_category c
  		  LEFT JOIN ( SELECT id, category_id FROM tb_article WHERE is_delete = 0 AND `status` = 1 ) a ON c.id = a.category_id
  		GROUP BY
  		  c.id //按文章主键分组
  </select>
  ```

- ```sql
  // 查询分类 【联合:文章表】
  <select id="listCategoryBackDTO" resultType="com.minzheng.blog.dto.CategoryBackDTO">
  		SELECT
  		  c.id,
  		  c.category_name,
  		  COUNT( a.id ) AS article_count,  //count会合并相同a.id的数据
  		  c.create_time
  		FROM
  		  tb_category c
  		  LEFT JOIN tb_article a ON c.id = a.category_id
  		<where>
  			<if test="condition.keywords != null">
  			     category_name like concat('%',#{condition.keywords},'%')
  			</if>
  		</where>
  		GROUP BY
  		  c.id
  		ORDER BY
  		  c.id DESC
          LIMIT #{current},#{size}
   </select>
  ```


### 4.评论mapper

- ```sql
  //后台返回所有评论信息  （左连接则会产生左表字段均会显现，但右表只显现两表连接条件相同的结果，不匹配则为null）
  <select id="listCommentBackDTO" resultType="com.minzheng.blog.dto.CommentBackDTO">
          SELECT
           c.id,
           u.avatar,
           u.nickname,
           r.nickname AS reply_nickname,
           a.article_title,
           c.comment_content,
           c.type,
           c.create_time,
           c.is_review
          FROM
           tb_comment c                                         //评论表
           LEFT JOIN tb_article a ON c.topic_id = a.id          //文章相关id
           LEFT JOIN tb_user_info u ON c.user_id = u.id         //评论用户id
           LEFT JOIN tb_user_info r ON c.reply_user_id = r.id   //回复用户id
         <where>
             <if test="condition.type != null">
                 c.type = #{condition.type}
             </if>
             <if test="condition.isReview != null">
                and c.is_review = #{condition.isReview}
             </if>
              <if test="condition.keywords != null">
                and u.nickname like concat('%',#{condition.keywords},'%')
              </if>
          </where>
          ORDER BY
            id DESC
          LIMIT #{current},#{size}
  </select>
  ```

### 5.角色mapper

- ```sql
  //三表查询   mybatis 会自动封装查询到的结果   rr.resource_id, rm.menu_id （列表集合）
  <select id="listRoles" resultMap="RoleMap">
          SELECT
          r.id,
          role_name,
          role_label,
          r.create_time,
          r.is_disable,
          rr.resource_id,
          rm.menu_id
          FROM
          (
            SELECT
              id,
              role_name,
              role_label,
              create_time,
              is_disable
            FROM
              tb_role   //角色表
          <where>
              <if test="conditionVO.keywords != null ">
                  role_name like concat('%',#{conditionVO.keywords},'%')
              </if>
          </where>
          LIMIT #{current}, #{size}
          ) r
          LEFT JOIN tb_role_resource rr ON r.id = rr.role_id  //角色资源表
          LEFT JOIN tb_role_menu rm on r.id = rm.role_id      //角色菜单表
          ORDER BY r.id
      </select>
  ```

