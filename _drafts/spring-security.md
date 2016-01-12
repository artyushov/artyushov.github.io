---
layout: post
title:  "Setting up Spring Security step by step"
categories: blog
---

In this post I want to describe how to add <a class="underline" href="http://projects.spring.io/spring-security/">Spring Security</a>
to your application and customize it step by step. I assume that you already have a running web application.
All configurations will be implemented in Java (not xml).

The first thing you need to do it to add a spring-security dependencies. In gradle it will look like

{% highlight groovy %}
compile('org.springframework.security:spring-security-web:4.0.3.RELEASE')
compile('org.springframework.security:spring-security-config:4.0.3.RELEASE')
{% endhighlight %}

Let's add a controller which will serve us as a test for our security configuration. Mapping names should say for themselves.
{% highlight java %}
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

{% highlight java %}
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
