---
layout: post
title: "SpringBoot integrates with Spring Security"
description: "springboot和spring security集成"
categories: [SpringBoot]
tags: [security]
redirect_from:
  - /2018/11/08/
---
本文介绍一种集成springboot和spring security的方式

在spring mvc模式下，前后端一起开发，集成spring是很容易的，登陆成功/失败都可以直接指定跳转到那个页面，但是前后端分离项目中，登陆成功/失败/权限校验成功/失败，都需要返回json给前端，前端拿到json后判断后续逻辑。

本人比较喜欢一种配置都写在一个配置文件里，所以下面会有匿名内部类。

security 的配置
>**UserDetailsService**：用户要根据自己的数据库实现其中的**loadUserByUsername**方法，返回UserDetails的实例。UserDetails可以用框架自带的User类。
>**AuthenticationEntryPoint**：服务端会对每次request进行过滤，判断是否已经登陆，这个类就是当判断当前请求未登录的情况下，进行处理。默认的实现有
>- **LoginUrlAuthenticationEntryPoint**：返回到login界面；
>-  **Http401AuthenticationEntryPoint**：返回401
>- 自定义类，返回json；

>**AuthenticationFailureHandler**：自定义登陆失败时，返回json
>**AuthenticationSuccessHandler**：自定义登陆成功时，返回json
>**AccessDeniedHandler**：自定义权限不足时，返回json

```java
/**
* @author slbai
*/
@Configuration
@EnableWebSecurity/
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    private ObjectMapper objectMapper = new ObjectMapper();

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authenticationProvider(authenticationProvider())
            .httpBasic()
                .authenticationEntryPoint((request,response,authException) -> {
                    response.setContentType("application/json;charset=utf-8");
                    response.setStatus(HttpServletResponse.SC_FORBIDDEN);
                    PrintWriter out = response.getWriter();
                    out.write(objectMapper.writeValueAsString(Result.fail("未登录")));
                    out.flush();
                    out.close();
                })
                .and()
            .authorizeRequests()
                .antMatchers("/", "/home").permitAll()
                .anyRequest().authenticated()
                .and()
            .formLogin()
                .loginPage("/login")
                .permitAll()
                .failureHandler((request,response,ex) -> {
                    response.setContentType("application/json;charset=utf-8");
                    response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                    PrintWriter out = response.getWriter();
                    if (ex instanceof UsernameNotFoundException || ex instanceof BadCredentialsException) {
                        out.write(objectMapper.writeValueAsString(Result.fail("用户名或密码错误")));
                    } else if (ex instanceof DisabledException) {
                        out.write(objectMapper.writeValueAsString(Result.fail("账户被禁用")));
                    } else {
                        out.write(objectMapper.writeValueAsString(Result.fail("登录失败!")));
                    }
                    out.flush();
                    out.close();
                })
                .successHandler((request,response,authentication) -> {
                    response.setContentType("application/json;charset=utf-8");
                    PrintWriter out = response.getWriter();
                    out.write(objectMapper.writeValueAsString(Result.succ(authentication.getPrincipal())));
                    out.flush();
                    out.close();
                })
                .and()
            .exceptionHandling()
                .accessDeniedHandler((request,response,ex) -> {
                    response.setContentType("application/json;charset=utf-8");
                    response.setStatus(HttpServletResponse.SC_FORBIDDEN);
                    PrintWriter out = response.getWriter();
                    out.write(objectMapper.writeValueAsString(Result.fail("权限不足")));
                    out.flush();
                    out.close();
                })
                .and()
            .logout()
                .permitAll();
    }

    @Override
    public void configure(WebSecurity web) {
        web.ignoring().antMatchers(HttpMethod.OPTIONS, "/**");//对于在header里面增加token等类似情况，放行所有OPTIONS请求。
    }

    @Autowired
    protected void configureGlobal(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userDetailsServiceBean());
    }

    @Bean
    @Override
    public UserDetailsService userDetailsServiceBean() {
        return new RedisUserDetailsService();
    }


    @Bean
    public BCryptPasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public AuthenticationProvider authenticationProvider() {
        DaoAuthenticationProvider authenticationProvider = new DaoAuthenticationProvider();
        authenticationProvider.setUserDetailsService(userDetailsServiceBean());
        authenticationProvider.setPasswordEncoder(passwordEncoder());
        return authenticationProvider;
    }


}
```
源码：[https://github.com/slbai/example-spring-security](https://github.com/slbai/example-spring-security)