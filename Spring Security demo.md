# Spring Security登录注册demo原理

在本例中，前端代码使用vue.js框架，后端使用spring boot框架，并通过spring security框架实现用户授权与角色控制。



### 前端的页面跳转

#### 1.定义默认路径

首先在router中设置根路径为登录界面。

```js
routes: [
    {
		//router中定义了根路径为登录界面
      	path: '/',
      	name: 'Login',
      	component: Login,    //表示使用的组件为Login，对应的是Login.vue文件
      	hidden: true
    },
    {
      	path: '/home',
      	name: '主页',
      	component: Home,
      	mata: {
        	requiredAuth: true
      	},
      	children: [
        	{
          		path: '/chat',
          		name: '消息',
          		component: Chat,
          		hidden: true,
          		meta: {
            		keepAlive: false,
            		requiredAuth: true
          		}
        	}
      	]
    }
]
```

#### 2.定义前端登录请求

前端代码中的Login.vue文件中定义点击登录按钮所执行的function

```js
submitClick: function () {
	var _this = this;
    this.loading = true;
    this.postRequest('/login', {
    	username: this.loginForm.username,
        password: this.loginForm.password
    }).then(resp=> {
        _this.loading = false;
        if (resp && resp.status == 200) {
        	var data = resp.data;
            _this.$store.commit('login', data.obj);
            var path = _this.$route.query.redirect;
            _this.$router
                .replace({path: path == '/' || path == undefined ? '/home' : path});
        }
    });
}
```

后台登录操作的controller为/login，必须要把username和password用POST的方法传至后台，这与spring security框架的使用有关。





### 后端security框架进行用户授权(即登录)的原理

#### 1.接口的定义

在后台代码中不需要定义/login接口，spring security有内置的login接口，可以在class UsernamePasswordAuthenticationFilter 中查看到相关代码。

因此在自定义的controller中没有/login资源。

```java
public UsernamePasswordAuthenticationFilter() {
    //内置一个/login接口，用于处理用户登录操作
	super(new AntPathRequestMatcher("/login", "POST"));
}
```



#### 2.Spring Security框架内置类中的关系

当一个登陆请求从前端发送到后端以后，过滤器**UsernamePasswordAuthenticationFilter**首先将这个请求拦截。

```java
public class UsernamePasswordAuthenticationFilter extends
		AbstractAuthenticationProcessingFilter
```

可以看出该过滤器继承了父类**AbstractAuthenticationProcessingFilter**，所以先执行父类的doFilter方法。

```java
public abstract class AbstractAuthenticationProcessingFilter extends GenericFilterBean
		implements ApplicationEventPublisherAware, MessageSourceAware {
	
	public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
			throws IOException, ServletException {

		HttpServletRequest request = (HttpServletRequest) req;
		HttpServletResponse response = (HttpServletResponse) res;

		if (!requiresAuthentication(request, response)) {
			chain.doFilter(request, response);
			return;
		}

		if (logger.isDebugEnabled()) {
			logger.debug("Request is to process authentication");
		}

		Authentication authResult;

		try {
			authResult = attemptAuthentication(request, response);
			if (authResult == null) {
				return;
			}
			sessionStrategy.onAuthentication(authResult, request, response);
		}
		catch (InternalAuthenticationServiceException failed) {
			logger.error(
					"An internal error occurred while trying to authenticate the user.",
					failed);
			unsuccessfulAuthentication(request, response, failed);

			return;
		}
		catch (AuthenticationException failed) {
			// Authentication failed
			unsuccessfulAuthentication(request, response, failed);

			return;
		}
		// Authentication success
		if (continueChainBeforeSuccessfulAuthentication) {
			chain.doFilter(request, response);
		}
		successfulAuthentication(request, response, chain, authResult);
	}

}
```

这段代码的解释如下：

1. 首先判断filter是否可以处理当前请求，如果不行，则交个下一个filter去处理。

   ```java
   if (!requiresAuthentication(request, response)) {
   	chain.doFilter(request, response);
   	return;
   }
   ```

2. 然后尝试对该请求进行认证，执行**attemptAuthentication**函数

   ```java
   try {
   	authResult = attemptAuthentication(request, response);
   	if (authResult == null) {
   		return;
   	}
   	sessionStrategy.onAuthentication(authResult, request, response);
   }
   ```

   查看源码可知函数**attemptAuthentication**是**AbstractAuthenticationProcessingFilter**中定义的一个虚函数，在子类**UsernamePasswordAuthenticationFilter**中实现，所以执行子类中的**attemptAuthentication**函数。

   ```java
   public Authentication attemptAuthentication(HttpServletRequest request,
   			HttpServletResponse response) throws AuthenticationException {
   	if (postOnly && !request.getMethod().equals("POST")) {
   		throw new AuthenticationServiceException(
   			"Authentication method not supported: " + request.getMethod());
   	}
   	String username = obtainUsername(request);
   	String password = obtainPassword(request);
   	if (username == null) {
   		username = "";
   	}
   	if (password == null) {
   		password = "";
   	}
   	username = username.trim();
   	UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(
   				username, password);
   	setDetails(request, authRequest);
   	return this.getAuthenticationManager().authenticate(authRequest);
   }
   ```

   在这个方法中，首先需要判断请求方式是不是POST，然后从请求中获取到username和password，并加入一些细节-。-

   然后创建一个**UsernamePasswordAuthenticationToken**的实例，触发该类的构造函数。

   ```java
   public class UsernamePasswordAuthenticationToken extends AbstractAuthenticationToken {
   	public UsernamePasswordAuthenticationToken(Object principal, Object credentials) {
   		super(null);
   		this.principal = principal;
   		this.credentials = credentials;
   		setAuthenticated(false);
   	}
   }
   ```

   由于只是刚刚登陆过来，并没有验证username和password是否正确，只是创建一个对象，但是还没有进行验证，所以setAuthenticated(false)。

3. 回到**attemptAuthentication**方法

   ```java
   return this.getAuthenticationManager().authenticate(authRequest);
   ```

   调用**this（UsernamePasswordAuthenticationFilter）**的**getAuthenticationManager**方法，其实就是父类**AbstractAuthenticationProcessingFilter**的方法。

   ```java
   protected AuthenticationManager getAuthenticationManager() {
   	return authenticationManager;
   }
   ```

4. 调用**AuthenticationManager**

   可以发现AuthenticationManager只是一个接口，它的实现类是**ProviderManager**，查看该类的**authenticate**方法。

   ```java
   public class ProviderManager implements AuthenticationManager, MessageSourceAware,InitializingBean {
       public Authentication authenticate(Authentication authentication)
   			throws AuthenticationException {
   		Class<? extends Authentication> toTest = authentication.getClass();
   		AuthenticationException lastException = null;
   		AuthenticationException parentException = null;
   		Authentication result = null;
   		Authentication parentResult = null;
   		boolean debug = logger.isDebugEnabled();
   
   		for (AuthenticationProvider provider : getProviders()) {
   			if (!provider.supports(toTest)) {
   				continue;
   			}
   
   			if (debug) {
   				logger.debug("Authentication attempt using "
   						+ provider.getClass().getName());
   			}
   
   			try {
                   //这句很重要！！！！
   				result = provider.authenticate(authentication);
   
   				if (result != null) {
   					copyDetails(authentication, result);
   					break;
   				}
   			}
   			catch (AccountStatusException e) {
   				prepareException(e, authentication);
   				throw e;
   			}
   			catch (InternalAuthenticationServiceException e) {
   				prepareException(e, authentication);
   				throw e;
   			}
   			catch (AuthenticationException e) {
   				lastException = e;
   			}
   		}
   
   		if (result == null && parent != null) {
   			try {
   				result = parentResult = parent.authenticate(authentication);
   			}
   			catch (ProviderNotFoundException e) {
   			}
   			catch (AuthenticationException e) {
   				lastException = parentException = e;
   			}
   		}
   
   		if (result != null) {
   			if (eraseCredentialsAfterAuthentication
   					&& (result instanceof CredentialsContainer)) {
   				((CredentialsContainer) result).eraseCredentials();
   			}
   			if (parentResult == null) {
   				eventPublisher.publishAuthenticationSuccess(result);
   			}
   			return result;
   		}
   		if (lastException == null) {
   			lastException = new ProviderNotFoundException(messages.getMessage(
   					"ProviderManager.providerNotFound",
   					new Object[] { toTest.getName() },
   					"No AuthenticationProvider found for {0}"));
   		}
   
   		if (parentException == null) {
   			prepareException(lastException, authentication);
   		}
   
   		throw lastException;
   	}
   }
   ```

   在这个方法中使用到了**AuthenticationProvider**，而AuthenticationProvider只是一个接口，这个接口的实现类为**DaoAuthenticationProvider**

5. **DaoAuthenticationProvider**的调用

   可以发现**DaoAuthenticationProvider**继承了**AbstractUserDetailsAuthenticationProvider**，查看这个类中的**authenticate**方法。

   ```java
   public abstract class AbstractUserDetailsAuthenticationProvider implements
   		AuthenticationProvider, InitializingBean, MessageSourceAware {
       public Authentication authenticate(Authentication authentication)
   			throws AuthenticationException {
   		Assert.isInstanceOf(UsernamePasswordAuthenticationToken.class, authentication,
   				() -> messages.getMessage(
   						"AbstractUserDetailsAuthenticationProvider.onlySupports",
   						"Only UsernamePasswordAuthenticationToken is supported"));
   
   		String username = (authentication.getPrincipal() == null) ? "NONE_PROVIDED"
   				: authentication.getName();
   
   		boolean cacheWasUsed = true;
   		UserDetails user = this.userCache.getUserFromCache(username);
   
   		if (user == null) {
   			cacheWasUsed = false;
   
   			try {
                   //重要！！！
   				user = retrieveUser(username,
   						(UsernamePasswordAuthenticationToken) authentication);
   			}
   			catch (UsernameNotFoundException notFound) {
   				logger.debug("User '" + username + "' not found");
   
   				if (hideUserNotFoundExceptions) {
   					throw new BadCredentialsException(messages.getMessage(
   							"AbstractUserDetailsAuthenticationProvider.badCredentials",
   							"Bad credentials"));
   				}
   				else {
   					throw notFound;
   				}
   			}
   
   			Assert.notNull(user,
   					"retrieveUser returned null - a violation of the interface contract");
   		}
   
   		try {
               //重要！！！！！
   			preAuthenticationChecks.check(user);
   			additionalAuthenticationChecks(user,
   					(UsernamePasswordAuthenticationToken) authentication);
   		}
   		catch (AuthenticationException exception) {
   			if (cacheWasUsed) {
   				cacheWasUsed = false;
   				user = retrieveUser(username,
   						(UsernamePasswordAuthenticationToken) authentication);
   				preAuthenticationChecks.check(user);
   				additionalAuthenticationChecks(user,
   						(UsernamePasswordAuthenticationToken) authentication);
   			}
   			else {
   				throw exception;
   			}
   		}
   
   		postAuthenticationChecks.check(user);
   
   		if (!cacheWasUsed) {
   			this.userCache.putUserInCache(user);
   		}
   
   		Object principalToReturn = user;
   
   		if (forcePrincipalAsString) {
   			principalToReturn = user.getUsername();
   		}
   
   		return createSuccessAuthentication(principalToReturn, authentication, user);
   	}
   }
   ```

   方法中使用到了user = retrieveUser(username, (UsernamePasswordAuthenticationToken) authentication);

   这个方法定义在子类**DaoAuthenticationProvider**中，源代码如下：

   ```java
   protected final UserDetails retrieveUser(String username,
   			UsernamePasswordAuthenticationToken authentication)
   			throws AuthenticationException {
   	prepareTimingAttackProtection();
   	try {
   		UserDetails loadedUser = this.getUserDetailsService().loadUserByUsername(username);
   		if (loadedUser == null) {
   			throw new InternalAuthenticationServiceException(
   				"UserDetailsService returned null, which is an interface contract violation");
   		}
   		return loadedUser;
   	}
   	catch (UsernameNotFoundException ex) {
   		mitigateAgainstTimingAttack(authentication);
   		throw ex;
   	}
   	catch (InternalAuthenticationServiceException ex) {
   		throw ex;
   	}
   	catch (Exception ex) {
   		throw new InternalAuthenticationServiceException(ex.getMessage(), ex);
   	}
   }
   ```

   这个函数会调用稍后需要自定义的userservices的loadUserByUsername方法。

6. 回到**AbstractUserDetailsAuthenticationProvider**的**authenticate**方法。

   ```java
   preAuthenticationChecks.check(user);
   additionalAuthenticationChecks(user,
   	(UsernamePasswordAuthenticationToken) authentication);
   ```

   对用户账户和密码进行验证。

7. 如果验证成功则调用**createSuccessAuthentication**方法

   ```java
   protected Authentication createSuccessAuthentication(Object principal,
   			Authentication authentication, UserDetails user) {
   		UsernamePasswordAuthenticationToken result = new UsernamePasswordAuthenticationToken(
   				principal, authentication.getCredentials(),
   				authoritiesMapper.mapAuthorities(user.getAuthorities()));
   		result.setDetails(authentication.getDetails());
   		return result;
   }
   ```

   这个方法中使用了**UsernamePasswordAuthenticationToken**的构造函数，查看其源码

   ```java
   public UsernamePasswordAuthenticationToken(Object principal, Object credentials,
   			Collection<? extends GrantedAuthority> authorities) {
   	super(authorities);
   	this.principal = principal;
   	this.credentials = credentials;
   	super.setAuthenticated(true); // must use super, as we override
   }
   ```

   在这里会将状态设置为true。

以上是Spring Security默认的用户登录步骤，而我们通常使用的时候可以不使用**DaoAuthenticationProvider**，比如在本例中，如果使用默认的登录机制，则无法加载该用户所拥有的角色，所以我们需要自定义一个**MyAuthenticationProvider**实现**AuthenticationProvider**接口，然后实现**authenticate**方法，代码如下：

```java
@Component
public class MyAuthenticationProvider implements AuthenticationProvider {

	@Autowired
	private HrService hrsvc;
	
	@Override
	public Authentication authenticate(Authentication authentication) throws AuthenticationException {
		
		String username = authentication.getName();
		String password = (String) authentication.getCredentials();
		
		UserDetails loadedhr = (UserDetails) hrsvc.loadUserByUsername(username);
		if(loadedhr == null)
			throw new InternalAuthenticationServiceException("用户不存在！");
		//如果用户存在，则检查前端提供的密码是否正确
		BCryptPasswordEncoder pencoder = new BCryptPasswordEncoder();
		boolean ispwdcorrect = pencoder.matches(password, loadedhr.getPassword());
		if(!ispwdcorrect)
			throw new BadCredentialsException("密码错误");
		
		Collection<? extends GrantedAuthority> authorities = loadedhr.getAuthorities();
        // 构建返回的用户登录成功的token
        return new UsernamePasswordAuthenticationToken(loadedhr, password, authorities);
	}

	@Override
	public boolean supports(Class<?> authentication) {
		// TODO Auto-generated method stub
		return true;
	}

}
```

然后在主配置文件**SecurityConfig**中做以下配置：

```java
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
	
    @Autowired
    MyAuthenticationProvider mp;  //自定义的AuthenticateProvider
	
	@Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
		auth.authenticationProvider(mp); //表明使用这个自定义类的对象进行用户的认证
    }
```



### 后端需要自定义配置的部分

#### 1.用户类的配置

自定义的用户类需要实现**UserDetails**接口，并实现几个该接口的虚函数，在本例中，代码如下

```java
public class Hr implements Serializable, UserDetails {
    /**获取用户的权限信息，并将其封装在一个GrantedAuthority类型的列表中，但具体权限是如何获得的可以根据实际
	   *情况编写，在本例中该用户的角色列表已经加载进了对象成员roles中，而roles为Role类型的列表，role的name属性即为
	   *权限的名称。
	 */
	@JsonIgnore
    @Override
	public Collection<? extends GrantedAuthority> getAuthorities() {
		List<GrantedAuthority> auths = new ArrayList<>();
		for(Role r : this.roles) {
			auths.add(new SimpleGrantedAuthority(r.getName()));
		}
		return auths;
	}
	//账号是否已过期
	@JsonIgnore
	@Override
	public boolean isAccountNonExpired() {
		// TODO Auto-generated method stub
		return true;
	}
	//账户是否被冻结
	@JsonIgnore
	@Override
	public boolean isAccountNonLocked() {
		// TODO Auto-generated method stub
		return true;
	}
	//账号密码是否过期，一般有的密码要求性高的系统会使用到，比如每隔一段时间就要求密码重置
	@JsonIgnore
	@Override
	public boolean isCredentialsNonExpired() {
		// TODO Auto-generated method stub
		return true;
	}
	//账号是否可用
	@JsonIgnore
	@Override
	public boolean isEnabled() {
		// TODO Auto-generated method stub
		return enabled;
	}
}
```

#### 2.服务类的配置

再自定义一个Service类，并实现**UserDetailsService**接口，需要实现该接口中的虚函数**loadUserByUsername**，这个函数会在DaoAuthenticationProvider的retrieveUser方法中被调用，源码即原理见上文。

```java
@Service
@Transactional
public class HrService implements UserDetailsService {
    @Autowired
	private HrMapper hrdao;
	
	@Override
	public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
		Hr hr = hrdao.loadUserByUsername(username);
		if(hr == null) {
			throw new UsernameNotFoundException("我的故事里没有你");
		}
		//返回一个UserDetails类型的对象，里面封装了该登录用户的信息。
		return hr;
	}
}
```

loadUserByUsername这个方法的方法体自定义，在本例中，既是在dao层定义了一个同名函数用作加载用户信息（源码省略）。**<font color="red">仅使用username从数据库中查找用户信息并加载到用户类中，并没有使用到password，即不论密码是否正确，如果存在这个username的用户，用户信息将被加载出来，在后续的步骤中再使用credential（前端传过来的密码）和用户信息中的真实密码去验证。这其中还会用到BCryptPasswordEncoder作为密码的加密。</font>**

#### 3.Spring Security框架的主配置类

新建**WebSecurityConfig**类继承**WebSecurityConfigurerAdapter**，本例中代码如下

```java
@Configuration
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
	
	//@Autowired
    //private HrService hrsvc;
	
	@Autowired
	SecurityMetadataSourceService ms;
    @Autowired
    AccessDecision ad;
    @Autowired
    MyAccessDeniedHandler adh;
    @Autowired
    MyAuthenticationProvider mp;
	
	@Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
		auth.authenticationProvider(mp);
		/*
		 * auth.userDetailsService(hrsvc) .passwordEncoder(new BCryptPasswordEncoder());
		 */
    }
	
	@Override
    public void configure(WebSecurity web) throws Exception {
        web.ignoring().antMatchers("/index.html", "/static/**", "/login_p", "/favicon.ico");
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
    	http.authorizeRequests()
        .withObjectPostProcessor(new ObjectPostProcessor<FilterSecurityInterceptor>() {
            @Override
            public <O extends FilterSecurityInterceptor> O postProcess(O o) {
                o.setSecurityMetadataSource(ms);
                o.setAccessDecisionManager(ad);
                return o;
            }
        })
        .and()
        .formLogin()  //定义当需要用户登录的时候，转到登录界面
        .loginPage("/login_p") //设置登录页面，每当访问到被保护的接口的时候就跳转至这个页面
        .loginProcessingUrl("/login") //登录请求拦截的url,也就是form表单提交时指定的action 
        .usernameParameter("username").passwordParameter("password")
        .failureHandler(new AuthenticationFailureHandler() {
            @Override
            public void onAuthenticationFailure(HttpServletRequest req,
                                                HttpServletResponse resp,
                                                AuthenticationException e) throws IOException {
                resp.setContentType("application/json;charset=utf-8");
                RespBean respBean = null;
                if (e instanceof BadCredentialsException ||
                        e instanceof UsernameNotFoundException) {
                    respBean = RespBean.error("账户名或者密码输入错误!");
                } else if (e instanceof LockedException) {
                    respBean = RespBean.error("账户被锁定，请联系管理员!");
                } else if (e instanceof CredentialsExpiredException) {
                    respBean = RespBean.error("密码过期，请联系管理员!");
                } else if (e instanceof AccountExpiredException) {
                    respBean = RespBean.error("账户过期，请联系管理员!");
                } else if (e instanceof DisabledException) {
                    respBean = RespBean.error("账户被禁用，请联系管理员!");
                } else {
                    respBean = RespBean.error("登录失败!");
                }
                resp.setStatus(401);
                ObjectMapper om = new ObjectMapper();
                PrintWriter out = resp.getWriter();
                out.write(om.writeValueAsString(respBean));
                out.flush();
                out.close();
            }
        })
        .successHandler(new AuthenticationSuccessHandler() {
            @Override
            public void onAuthenticationSuccess(HttpServletRequest req,
                                                HttpServletResponse resp,
                                                Authentication auth) throws IOException {
                resp.setContentType("application/json;charset=utf-8");
                RespBean respBean = RespBean.ok("登录成功!", HrUtils.getCurrentHr());
                ObjectMapper om = new ObjectMapper();
                PrintWriter out = resp.getWriter();
                out.write(om.writeValueAsString(respBean));
                out.flush();
                out.close();
            }
        })
        .permitAll()
        .and()
        .logout()
        .logoutUrl("/logout")
        .logoutSuccessHandler(new LogoutSuccessHandler() {
            @Override
            public void onLogoutSuccess(HttpServletRequest req, HttpServletResponse resp, Authentication authentication) throws IOException, ServletException {
                resp.setContentType("application/json;charset=utf-8");
                RespBean respBean = RespBean.ok("注销成功!");
                ObjectMapper om = new ObjectMapper();
                PrintWriter out = resp.getWriter();
                out.write(om.writeValueAsString(respBean));
                out.flush();
                out.close();
            }
        })
        .permitAll()
        .and().csrf().disable()
        .exceptionHandling().accessDeniedHandler(adh);
    }
}
```

代码的含义见注释。

另外需要说明，对于权限的管理更简单的方法是直接在主配置文件（SecurityConfig）中写死权限的配置，范例代码如下：

```java
.antMatchers("/decision/**","/govern/**","/employee/*").hasAnyRole("EMPLOYEE","ADMIN")//对decision和govern 下的接口 需要 USER 或者 ADMIN 权限
	.antMatchers("/employee/login").permitAll()///employee/login 不限定
	.antMatchers("/admin/**").hasRole("ADMIN")//对admin下的接口 需要ADMIN权限
	.antMatchers("/oauth/**").permitAll()//不拦截 oauth 开放的资源
	.anyRequest().permitAll()//其他没有限定的请求，允许访问
	.and().anonymous()//对于没有配置权限的其他请求允许匿名访问
	.and().formLogin()//使用 spring security 默认登录页面
	.and().httpBasic();//启用http 基础验证

```

同样也可以在controller的接口上方用注解@PreAuthorize("hasRole('ROLE_xxx')")来实现，但是使用这种方法的时候记得在主配置文件的类定义上使用注解@EnableGlobalMethodSecurity(prePostEnabled=true)，示例如下：

```java
@Configuration
@EnableGlobalMethodSecurity(prePostEnabled=true)
public class WebSecurityConfig extends WebSecurityConfigurerAdapter{
    ......
}
```

在本例中，使用的是用数据库维护用户的角色以及资源的权限信息，详细原理见下章。

#### 4.权限管理的原理

Spring Boot对于权限的管控的基本原理是：左边为该资源的配置信息，右边为已登录用户的用户信息，中间投票器决定该用户是否有权限访问该资源。

所以**第一步**需要获取该资源（url）的配置信息。Spring Boot中需要程序员自定义一个类实现**FilterInvocationSecurityMetadataSource**接口，本例的源代码如下：

```java
//这个类用于通过请求的地址，用于获取访问该资源所需要的用户角色

@Component
public class SecurityMetadataSourceService implements FilterInvocationSecurityMetadataSource {

	@Autowired
    private MenuService menusvc;

    AntPathMatcher ant = new AntPathMatcher();

	@Override
	public Collection<ConfigAttribute> getAttributes(Object object) throws IllegalArgumentException {
		//获得请求的地址
		String requestedUrl = ((FilterInvocation) object).getRequestUrl();
		//加载出所有的menu以及这些menu所对应需要的role
		List<Menu> allmenu = menusvc.getAllMenu();
		for (Menu m : allmenu) {
            //match函数为正则匹配，因为数据库中存储的url信息为/salary/table/**，而真实的url可能为/salary/table/aaa
            //如果数据库中存储的url地址为精确地址，则可以直接搜索该地址所需要的role，不需要加载全表再进行检索了
			if (ant.match(m.getUrl(), requestedUrl) && m.getRoles().size() > 0)
            {
				List<Role> rs = m.getRoles();
				int size = rs.size();
				String[] rolenames = new String[size];
				for (int i = 0; i < size; i++) {
					rolenames[i] = rs.get(i).getName();
				}
				return SecurityConfig.createList(rolenames);
			}	
		}
		//如果路径没有匹配上，则说明这个路径是登录后就可访问的
		return SecurityConfig.createList("ROLE_LOGIN");
	}

	@Override
	public Collection<ConfigAttribute> getAllConfigAttributes() {
		return null;
	}

	@Override
	public boolean supports(Class<?> clazz) {
		return FilterInvocation.class.isAssignableFrom(clazz);
	}

}
```

**第二步**为获取当前已登录用户，用户一旦登录成功，所有的用户信息（包括这个用户所拥有的所有角色）将会被缓存在**SecurityContextHolder**中，此时可以直接取出用户信息。

**第三步**为使用投票器去决定该用户是否可以方位该资源。从底层来讲，真正完成授权功能的是一组**AccessDecisionVoter**，而**AccessDecisionManager**维护着一个AccessDecisionVoter列表参与授权的投票。

投票器在Spring Security中有三个不同的实现：

1. UnanimousBased（全票通过）：素有投票器都通过才允许访问资源。
2. ConsensusBased（少数服从多数）：超过一半的投票器通过才允许访问。
3. AffirmativeBased（一票通过）：只要有一个投票器通过，就允许访问资源。（<font color="red">这个为默认实现</font>）

因此程序员通常只需自定义一个类实现**AccessDecisionManager**接口即可，其中decide方法为做决策的主要方法。

```java
@Component
public class AccessDecision implements AccessDecisionManager {

	/*
	 * decide函数由于判断是否让当前用户访问该请求
	 * 第一个参数保存了当前用户的角色信息
	 * 第三个参数保存了SecurityMetadataSource的getAttributes传来的值，表示当前请求需要的角色。
	 * */
	@Override
	public void decide(Authentication authentication, Object object, Collection<ConfigAttribute> configAttributes)
			throws AccessDeniedException, InsufficientAuthenticationException {
		if(null == configAttributes || configAttributes.size() <= 0)
			return;
		Iterator<ConfigAttribute> iterator = configAttributes.iterator();
        while (iterator.hasNext()) {
            ConfigAttribute ca = iterator.next();
            //当前请求需要的权限
            String needRole = ca.getAttribute();
            if ("ROLE_LOGIN".equals(needRole)) {
                if (authentication instanceof AnonymousAuthenticationToken) {
                    throw new BadCredentialsException("没登录就想访问的吗？？？");
                } else
                    return;
            }
            //当前用户所具有的权限
            Collection<? extends GrantedAuthority> authorities = authentication.getAuthorities();
            for (GrantedAuthority authority : authorities) {
                if (authority.getAuthority().equals(needRole)) {
                    return;
                }
            }
        }
        throw new AccessDeniedException("恕我直言你似乎没有权限！");
	}

	@Override
	public boolean supports(ConfigAttribute attribute) {
		// TODO Auto-generated method stub
		return true;
	}

	@Override
	public boolean supports(Class<?> clazz) {
		// TODO Auto-generated method stub
		return true;
	}

}
```

定义好**AccessDecision**和**SecurityMetadataSourceService**后，在主配置文件中做以下配置：

```java
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
	
	//@Autowired
    //private HrService hrsvc;
	
	@Autowired
	SecurityMetadataSourceService ms;    //自定义的FilterInvocationSecurityMetadataSource
    @Autowired
    AccessDecision ad;   //自定义的AccessDecisionManager
    @Autowired
    MyAccessDeniedHandler adh;
    @Autowired
    MyAuthenticationProvider mp;
    
    //在configure方法中使用这两个类做权限的管理
    @Override
    protected void configure(HttpSecurity http) throws Exception {
    	http.authorizeRequests()
        .withObjectPostProcessor(new ObjectPostProcessor<FilterSecurityInterceptor>() {
            @Override
            public <O extends FilterSecurityInterceptor> O postProcess(O o) {
                o.setSecurityMetadataSource(ms);
                o.setAccessDecisionManager(ad);
                return o;
            }
        })
        ......
    }
```