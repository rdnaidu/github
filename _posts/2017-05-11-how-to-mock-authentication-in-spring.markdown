---
layout: single
classes: wide
title:  "How to mock authentication in Spring"
date:   2017-05-11 16:24 -0500
tags:
  - spring
  - test
  - security
  - java
  - mock
categories: java spring testing 
excerpt: "Mock a simple but flexible security hierarchy to be check in calls to Rest Controllers"
comments: true
---
Have you ever looked for a simple and flexible way to mock a security hierarchy and try it out in your rest controllers in Spring? After reading the [Spring Security Reference][1] I realized there are straightforward ways using [AOP](https://docs.spring.io/spring/docs/4.3.x/spring-framework-reference/html/aop.html), which solutions often are the greatest ones for testing. For mocking credentials in Spring you can use the annotations `@WithMockUser`, `@WithUserDetails` and `@WithSecurityContext`, included in the artifact:

{% highlight xml %}
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-test</artifactId>
    <version>4.2.2.RELEASE</version>
    <scope>test</scope>
</dependency>
{% endhighlight %}

In most cases, `@WithUserDetails` gathers the flexibility and power I need.

## How @WithUserDetails works?
Basically you just need to create a custom `UserDetailsService` with all the possible users profiles you want to test. E.g

{% highlight java %}
@TestConfiguration
public class SpringSecurityWebAuxTestConfig {

    @Bean
    @Primary
    public UserDetailsService userDetailsService() {
        User basicUser = new UserImpl("Basic User", "user@company.com", "password");
        UserActive basicActiveUser = new UserActive(basicUser, Arrays.asList(
                new SimpleGrantedAuthority("ROLE_USER"),
                new SimpleGrantedAuthority("PERM_FOO_READ")
        ));

        User managerUser = new UserImpl("Manager User", "manager@company.com", "password");
        UserActive managerActiveUser = new UserActive(managerUser, Arrays.asList(
                new SimpleGrantedAuthority("ROLE_MANAGER"),
                new SimpleGrantedAuthority("PERM_FOO_READ"),
                new SimpleGrantedAuthority("PERM_FOO_WRITE"),
                new SimpleGrantedAuthority("PERM_FOO_MANAGE")
        ));

        return new InMemoryUserDetailsManager(Arrays.asList(
                basicActiveUser, managerActiveUser
        ));
    }
}
{% endhighlight %}

Now we have our users ready, so imagine we want to test the access control to this controller function:

{% highlight java %}
@RestController
@RequestMapping("/foo")
public class FooController {

    @Secured("ROLE_MANAGER")
    @GetMapping("/salute")
    public String saluteYourManager(@AuthenticationPrincipal User activeUser)
    {
        return String.format("Hi %s. Foo salutes you!", activeUser.getUsername());
    }
}
{% endhighlight %}

Here we have a *get mapped function* to the route **/foo/salute** and we are testing a role based security with the `@Secured` annotation, although you can test `@PreAuthorize` and `@PostAuthorize` as well.
Let's create two tests, one to check if a valid user can see this salute response and the other to check if it's actually forbidden.

{% highlight java %}
@RunWith(SpringRunner.class)
@SpringBootTest(
        webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT,
        classes = SpringSecurityWebAuxTestConfig.class
)
@AutoConfigureMockMvc
public class WebApplicationSecurityTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    @WithUserDetails("manager@company.com")
    public void givenManagerUser_whenGetFooSalute_thenOk() throws Exception
    {
        mockMvc.perform(MockMvcRequestBuilders.get("/foo/salute")
                .accept(MediaType.ALL))
                .andExpect(status().isOk())
                .andExpect(content().string(containsString("manager@company.com")));
    }

    @Test
    @WithUserDetails("user@company.com")
    public void givenBasicUser_whenGetFooSalute_thenForbidden() throws Exception
    {
        mockMvc.perform(MockMvcRequestBuilders.get("/foo/salute")
                .accept(MediaType.ALL))
                .andExpect(status().isForbidden());
    }
}
{% endhighlight %}

As you see we imported `SpringSecurityWebAuxTestConfig` to provide our users for testing. Each one used on its corresponding test case just by using a straightforward annotation, reducing code and complexity.

## Better use @WithMockUser for simpler Role Based Security

As you see `@WithUserDetails` has all the flexibility you need for most of your applications. It allows you to use custom users with any GrantedAuthority, like roles or permissions. But if you are just working with roles, testing can be even easier and you could avoid constructing a custom `UserDetailsService`. In such cases, specify a simple combination of user, password and roles with [@WithMockUser][2]. 

{% highlight java %}
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
@WithSecurityContext(
    factory = WithMockUserSecurityContextFactory.class
)
public @interface WithMockUser {
    String value() default "user";

    String username() default "";

    String[] roles() default {"USER"};

    String password() default "password";
}
{% endhighlight %}

{% highlight java %}
The annotation defines default values for a very basic user. As in our case the route we are testing just requires that the authenticated user be a manager, we can quit using `SpringSecurityWebAuxTestConfig` and do this.

    @Test
    @WithMockUser(roles = "MANAGER")
    public void givenManagerUser_whenGetFooSalute_thenOk() throws Exception
    {
        mockMvc.perform(MockMvcRequestBuilders.get("/foo/salute")
                .accept(MediaType.ALL))
                .andExpect(status().isOk())
                .andExpect(content().string(containsString("user")));
    }
{% endhighlight %}

Notice that now instead of the user **manager@company.com** we are getting the default provided by `@WithMockUser`: **user**; yet it won't matter because what we really care about is his role: `ROLE_MANAGER`.

## Conclusions

As you see with annotations like `@WithUserDetails` and `@WithMockUser` we can *switch between different authenticated users scenarios* without building classes alienated from our architecture just for making simple tests. Its also recommended you to see how [@WithSecurityContext][3] works for even more flexibility. 

## See more:
* [Testing](https://docs.spring.io/spring-security/site/docs/4.0.x/reference/htmlsingle/#test) in the Spring Security Reference. 
* [Spring Test & Security: How to mock authentication?](https://stackoverflow.com/questions/15203485/spring-test-security-how-to-mock-authentication) in StackOverflow.

[1]: http://docs.spring.io/spring-security/site/docs/4.0.x/reference/htmlsingle/#test-mockmvc
[2]: http://docs.spring.io/spring-security/site/docs/4.0.x/reference/htmlsingle/#test-method-withmockuser
[3]: http://docs.spring.io/spring-security/site/docs/4.0.x/reference/htmlsingle/#test-method-withsecuritycontext