---
layout: post
title:  "Setting up Spring Security step by step"
categories: blog
---

Just before starting this blog I was trying to implement a quick demo application which had different privileges for
different users. I already knew about spring security and wanted to play with it. However, I wasn't satisfied with the
quality of the tutorials I managed to find, so I decided to provide my own, which covers all the cases I was interested in.

In this post I want to describe how to add <a class="underline" href="http://projects.spring.io/spring-security/">Spring Security</a>
to your application and customize it step by step. I assume that you already have a running web application.
All configurations will be implemented in Java (not xml).

The first thing you need to do it to add a spring-security dependencies. In gradle it will look like

{% highlight groovy %}
compile('org.springframework.security:spring-security-web:4.0.3.RELEASE')
compile('org.springframework.security:spring-security-config:4.0.3.RELEASE')
{% endhighlight %}

Let's add a controller which will serve us as a test for our security configuration. Mapping names should say for themselves.
{% highlight java linenos %}
@RestController
public class IndexController {

    @RequestMapping("/open")
    public String hello() {
        return "This link is available to everyone";
    }

    @RequestMapping("/onlyForUsers")
    public String onlyForUsers() {
        return "This link is available for every authenticated user";
    }

    @RequestMapping("/admin")
    public String admin() {
        return "This link is available only to admins";
    }
}
{% endhighlight %}

Next, you should implement the security configuration itself. I'll give you a minimal working example right away,
and then explain how to modify it.

{% highlight java linenos %}
@Configuration
@EnableWebSecurity
@ComponentScan("me.artyushov.blog")
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    public void configureGlobal(AuthenticationManagerBuilder auth) throws Exception {
        auth.inMemoryAuthentication()
                .withUser("regularUser").password("123").roles("USER")
                .and()
                .withUser("admin").password("123abc").roles("USER", "ADMIN");
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .antMatchers("/admin/**").hasRole("ADMIN")
                .antMatchers("/onlyForUsers").authenticated()
                .antMatchers("/**").permitAll()
                .and()
                .formLogin();
    }
}
{% endhighlight %}

Here you can see that in <code>configureGlobal</code> method I just hardcoded all the users and their passwords. Later
I'll show how to use database as the source of user details.

Method <code>configure</code> is actually the place where the security policies are implemented. It specifies how every
request that matches the given <a class="underline" href="http://ant.apache.org/manual/dirtasks.html#patterns">ant pattern</a>
should be handled. Note, that the order in which matchers are specified is important. Spring javadoc quote:
<blockquote>
Note that the matchers are considered in order. Therefore, the following is invalid because the first matcher matches
every request and will never get to the second mapping:
{% highlight java %}
http.authorizeRequests().antMatchers("/**").hasRole("USER").antMatchers("/admin/**")
        .hasRole("ADMIN")
{% endhighlight %}
</blockquote>

Specifying <code>formLogin()</code> will create a default login form with <code>/login</code> mapping. You can set the
link for login form by adding <code>.loginPage("myLoginPage")</code>.

Now, suppose you have a table with user information in your database. So let's change our code so that it uses this
information.

{% highlight java linenos %}
    @Autowired
    public void configureGlobal(AuthenticationManagerBuilder auth) throws Exception {
        JdbcDaoImpl userDetailsService = new JdbcDaoImpl();
        userDetailsService.setJdbcTemplate(jdbcTemplate);
        userDetailsService.setUsersByUsernameQuery(
                "select name, password, 1 from User where name = ?");
        userDetailsService.setAuthoritiesByUsernameQuery(
                "select name, authority FROM UserAuthority where name = ?");

        DaoAuthenticationProvider authenticationProvider = new DaoAuthenticationProvider();
        authenticationProvider.setUserDetailsService(userDetailsService);

        auth.authenticationProvider(authenticationProvider);
    }
{% endhighlight %}

I hope that the code here is self explanatory. Note that <code>JdbcDaoImpl</code> and <code>DaoAuthenticationProvider</code>
are provided by Spring, but you can implement your own <code>UserDetailsService</code> as you wish.

One more useful feature I'd like to mention is the possibility to specify password encoder for authentication provider.
Spring security provides several default implementations for it which should be enough for the start.