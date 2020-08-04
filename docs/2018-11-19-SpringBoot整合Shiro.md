# SpringBoot整合Shiro

## 1. Shiro核心API

* Subject：用户主体(把操作交给SercurityManager)
* SecurityManager：安全管理器(关联Realm)
* Realm：Shiro连接数据的桥梁

---

## 2. 整合

> 1. 添加依赖
>
> 2. 自定义Realm类
>
>    ```java
>    public class CustomRealm extends AuthorizingRealm {
>        // 授权
>        @Override
>        protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection p) {
>            return null;
>        }
>        // 认证
>        @Override
>        protected AuthenticationInfo doGetAuthentication(AuthenticationToken token)		{
>            System.out.println("认证");
>            
>            return null;
>        }
>    }
>    ```
>
> 3. 新增shiro配置类
>
>    ```java
>    @Configuration
>    public class ShiroConfig {
>        // 创建ShiroFilterFactoryBean
>        @Bean
>        public ShiroFilterFactoryBean getShiroFilterFactoryBean(@Qualifier("securityManager") SecurityManager securityManager) {
>            ShiroFilterFactoryBean bean = new ShiroFilterFactoryBean();
>            // 设置安全管理器
>            bean.setSecurityManager(securityManager);
>            return bean;
>        }
>        
>        // 创建DefaultWebSecurityManager
>        @Bean(name="securityManager")
>        public DefaultWebSercurityManager getDefaultWebSercurityManager(@Qualifier("customRealm") CustomRealm customRealm) {
>            DefaultWebSercurityManager sercurity = new DefaultWebSercurityManager();
>            // 关联realm
>            sercurity.setRealm(customRealm);
>            return security;
>        }
>        
>        // 创建Realm
>        @Bean(name="customRealm")
>        public CustomRealm getRealm() {
>            return new CustomRealm();
>        }
>    }
>    ```
>
>    4. 使用内置过滤器实现拦截
>
>       ```java
>       ShiroFilterFactoryBean bean = new ShiroFilterFactoryBean();
>       // 设置安全管理器
>       bean.setSecurityManager(securityManager);
>       // 在此处添加内置过滤器
>       /**
>        * 可以实现权限相关的拦截
>        * anno：无需认证(登陆)就可访问
>        * authc：必须认证才可访问
>        * user：如果使用rememberMe可直接访问
>        * perms：该资源必须得到资源权限才可以访问
>        * role：有角色才可以访问
>        */
>       Map<String, String> map = new LinkedHashMap();
>       map.put("/add", "authc");
>       map.put("/update", "authc");
>       // 通配形式 map.put("/*", "authc");
>       // 修改未登陆的页面
>       bean.setLoginUrl("/login");// 写对应跳转俄controller
>       bean.setFilterChainDefinitionMap(map);
>       return bean;
>       ```
>
>    5. 认证登陆操作
>
>       ```java
>       @RequestMapping("/login")
>       public String login(String name, String password, Model model) {
>           // 使用shiro编写认证操作
>           // 获取subject
>           Subject subject = SecurityUtils.getSubject();
>           // 封装用户数据
>           UsernamePasswordToken token = new UsernamePasswordToken(name,password);
>           try {
>               subject.login(token);
>               return "r:index";
>           } catch (UnknownAccountException e) {
>               model.addAttribute("msg","用户不存在！");
>               return "login";
>           } catch (IncorrectCredentialsException e) {
>               model.addAttribute("msg","用户名或密码错误！");
>               return "login";
>           }
>       }
>       ```
>

