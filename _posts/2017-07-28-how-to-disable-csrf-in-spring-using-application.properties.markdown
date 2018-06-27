---
layout: single
classes: wide
title:  "How to disable CRSF in Spring Using an application property"
date:   2017-07-28 22:15 -0500
tags:
  - spring
  - security
  - CRSF
  - Java
  - properties
categories: java spring security 
excerpt: "How to disable CRSF protection in Spring by setting `security.enable_csrf: false` in the application.properties"
comments: true
---

## Problem 
For most of web developers <abr title="Cross-site request forgery">CSRF</abr> is a well known security exploit, on which non expected but allowed commands could be sent to a website by a "trusted user" with malicious intentions. In the Spring documentation about Web Application Security it explain how to [configure the CRSF Protection][5]. 

You may have noticed that the Spring boot property `security.enable-csrf` would take care of enabling and disabling this feature. Nonetheless its meant to be *on* by default and to disable it you must do it by Java or xml code.

The property alternative could be a great way so you can, for instance create a profile that disable this security protection, so you can focus in the actual functionality

 
 **Property working in newer versions:** Based on a [comment of a Spring Boot member][1] this [issue][3] is fixed on new versions of Spring: I had it on version `1.5.2.RELEASE` but it seems that in version [1.5.9.RELEASE][2] (the latest stable one to the date before version 2) its already fixed and by default **csrf** is disabled and it can be enabled with `security.enable_csrf: true`. Therefore a possible solution could be just upgrading to version `1.5.9.RELEASE`, before making a major one to version 2 where the architecture might be quite more different. The solution that will be presented is compatible with any version. 
 {: .notice--warning}

## My solution
As the `WebSecurityConfigurerAdapter` uses an imperative approach you can inject the value of the `security.enable-csrf` variable and disable CSRF  when it be false. You are right, I think this should work out of the box.

{% highlight java %}
@Configuration
public class AuthConfig extends WebSecurityConfigurerAdapter {
    @Autowired
    private UserDetailsService userDetailsService;

    @Value("${security.enable-csrf}")
    private boolean csrfEnabled;

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userDetailsService).passwordEncoder(new BCryptPasswordEncoder());
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        super.configure(http);

        if(!csrfEnabled)
        {
            http.csrf().disable();
        }
    }
}
{% endhighlight %} 

What I did was to set that variable to false in my application.yml for when I had a *dev* spring profile active, although you could create a profile called *nosecurity* for such purposes too. It eases this process a lot:

{% highlight yml %}
--- application.yml ---
#Production configuration
server:
    port: ${server.web.port}
admin:
    email: ${admin.email}
#etc
---
spring:
    profiles: dev

security.enable-csrf: false

#Other Development configurations
{% endhighlight %}    

I hope it suits your needs

## See more

* [How to disable csrf in Spring using application.properties?][4] in StackOverflow.
* Closed [issue][1] in Github.
* [CSRF Configuration][5] in Spring Security.
* [Cross-site request forgery](https://en.wikipedia.org/wiki/Cross-site_request_forgery) in Wikipedia.

  [1]: https://github.com/spring-projects/spring-boot/issues/11170#issuecomment-350805732
  [2]: https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-dependencies/1.5.9.RELEASE
  [3]:https://github.com/spring-projects/spring-boot/issues/11170
  [4]: https://stackoverflow.com/questions/44824382/how-to-disable-csrf-in-spring-using-application-properties/45383128#45383128
  [5]: https://docs.spring.io/spring-security/site/docs/current/reference/html/csrf.html#csrf-configure