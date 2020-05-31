---
title:  "Securing your application with Spring Security"
date:   2017-07-09 15:45:00 +0100
categories: spring
excerpt: We'll explore a simple scenario of securing demo showcase for your project.
---

It has been about 2 months since last post, and Project 1 is ready for demo showcase! As you may remember, it's meant to be commercialized, but it's also a statement of my technical skills, passion for programming and constant drive to self-development. Therefore, I'd like to limit the possibility of using the webapp to people, which contacted me in order to hire me and would like to see the proof of my skills. Spring Security is a perfect tool to do the job and I'll cover such scenario in this post.

Spring Security, as its name suggests, is all about securing your application. It's one of the framework's most known project, implemented using AOP and servlet filters. You can do a whole lot of things with it, just to name a few:
* authorize users with sources ranging from in-memory storage, throughout LDAP, to relational and non-relational databases
* secure pages or specific elements which builds up the page (think of a button which allows to delete users comments and is only visible to logged-in admin)
* secure application endpoints by using advanced filtering options (like allow to perform POST operations from controller A only to logged-in users)
* automatically add CSRF token to forms.

Enough talking, let's skip to the meat. To start, we need to add Spring Security to the classpath. The most convenient way to do so is throughout extending Maven's dependencies with Spring Boot Security Starter:

{% highlight xml %}
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
  </dependency>
{% endhighlight %}

If you run the application now, every request (be it a page or endpoint) will be intercepted and you'll be asked to enter your credentials to proceed. The funny things is, there's no users source provided which implies there is no such login data that allows to proceed. That's how the default implementation of `WebSecurityConfigurerAdapter`'s `configure(HttpSecurity)` method works. There're some things we have to do to make use of it. Let's start with providing user's database.

While in-memory authentication is sufficient in case of development environment, in production you'll mostly lookup database. If you've read my [previous post](https://przemyslawwozniak.github.io/programming/spring-data-mongodb/), you know that I've choosen non-relational database to persist P1's data. In case of users I'll do the same. It's also a little bit more challenging as Spring Security provides support for relational databases as well as LDAP out of the box. In case of non-relational database, you'll have to implement `UserDetailsService` interface yourself. Before we'll reach it, there are several classes which have to be coded.

The first one is `User` which is a MongoDB's document representing persisted user entities. In my use case, my goal is to only store the basic informations that allows for user identification, that is user's name and password. If you take a look at `UserDetailsService` interface [documentation](https://docs.spring.io/spring-security/site/docs/3.2.7.RELEASE/apidocs/org/springframework/security/core/userdetails/UserDetailsService.html), you'll notice that it's one and only method, which is `loadUserByUsername(String)` returns objects of type `UserDetails`, which is also an interface. To avoid wrapping up, we'd like our `User` class to implement this interface upfront. We consult ourself with [docs](https://docs.spring.io/spring-security/site/docs/3.2.7.RELEASE/apidocs/org/springframework/security/core/userdetails/UserDetails.html) once again and write something like this:

{% highlight java %}
@Document
public class User implements UserDetails {

    @Id
    private String id;
    private String username;
    private String password;
    private boolean isAccountExpired;
    private boolean isAccountLocked;
    private boolean isCredentialsExpired;
    private boolean isEnabled;

    private List<String> grantedAuthoritiesList;

    public User() {}

    public User(String username, String password) {
        this.username = username;
        this.password = password;

        this.isAccountExpired = false;
        this.isAccountLocked = false;
        this.isCredentialsExpired = false;
        this.isEnabled = true;
        grantedAuthoritiesList = new ArrayList<>();
        grantedAuthoritiesList.add(ConstProvider.USER_ROLE);
    }

    public User(String username, String password, List<String> grantedAuthoritiesList) {
        this.username = username;
        this.password = password;
        this.grantedAuthoritiesList = grantedAuthoritiesList;

        this.isAccountExpired = false;
        this.isAccountLocked = false;
        this.isCredentialsExpired = false;
        this.isEnabled = true;
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        List<GrantedAuthority> authorities = new ArrayList<>();

        for(String authorityStr : grantedAuthoritiesList) {
            authorities.add(new SimpleGrantedAuthority(authorityStr));
        }

        return authorities;
    }

    @Override
    public boolean isAccountNonExpired() {
        return !isAccountExpired;
    }

    @Override
    public boolean isAccountNonLocked() {
        return !isAccountLocked;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return !isCredentialsExpired;
    }

    //...boilerplate code below
{% endhighlight %}

`User` class is self-explanatory but you'd probably like to have a look at `getAuthorities()` implementation. While we store authorities (roles, in other words, like `ROLE_USER`, `ROLE_ADMIN` and so on) as a list of `String`s, we have to return a class that extends `GrantedAuthority`, which is achieved with `SimpleGrantedAuthority` constructor.

Next: we have to somehow retrieve our `User`s from MongoDB. That's achieved with this simple `UserRepository` implementation:

{% highlight java %}
@Repository
public interface UserRepository extends MongoRepository<User, String> {

    User findOneByUsername(String username);

}
{% endhighlight %}

Once again, if you find any of these topics unclear, I encourage you to read [this post](https://przemyslawwozniak.github.io/programming/spring-data-mongodb/). The next step is providing a controller, which allows admin to add new users, alongside with DTO class for data transfer; it's very simple, consisting of only three fields (`username`, `password`, `grantedAuthoritiesList`), so I won't attach it. Here's code of `UserRepositoryController`:

{% highlight java %}
@Controller
public class UserRepositoryController {

    private static final Logger LOG = LoggerFactory.getLogger(UserRepositoryController.class);

    @Autowired
    private UserRepository userRepository;

    @PostMapping("/addUser")
    public ResponseEntity<?> addUser(@RequestBody UserDTO user) {
        try {
            userRepository.save(new User(user.getUsername(), user.getPassword()));
        }
        catch(Exception ex) {
            String errorMsg = "Exception occured when adding new user to the database: " + ex.getMessage();
            LOG.error(errorMsg);

            return new ResponseEntity<>(errorMsg, HttpStatus.INTERNAL_SERVER_ERROR);
        }

        return new ResponseEntity<>(HttpStatus.CREATED);
    }

    @PostMapping("/addUserWithRoles")
    public ResponseEntity<?> addUserWithRoles(@RequestBody UserDTO user) {
        try {
            userRepository.save(new User(user.getUsername(), user.getPassword(), user.getGrantedAuthoritiesList()));
        }
        catch(Exception ex) {
            String errorMsg = "Exception occured when adding new user to the database: " + ex.getMessage();
            LOG.error(errorMsg);

            return new ResponseEntity<>(errorMsg, HttpStatus.INTERNAL_SERVER_ERROR);
        }

        return new ResponseEntity<>(HttpStatus.CREATED);
    }

}
{% endhighlight %}

Nothing surprising here. Basically, we provide two endpoints, one capable of adding to the database a `User` identified by `username` and `password` (we don't do hashing here, but you should in case of production environment), with a `ROLE_USER` (see two-param constructor of `User` class above) and another which allows us to additionaly define its authorities.

Now, the most important class, which provides Spring Security configuration. That's the single point of app's behaviour definition. Let's take a look on a `SecurityConfig` class:

{% highlight java linenos %}
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    UserDetailsService userService;

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                .authorizeRequests()
                    .antMatchers("/welcome.html").permitAll()
                    .antMatchers("/deleteAll/\*\*", "/addUser", "addUserWithRoles").hasAuthority(ConstProvider.ADMIN_ROLE)
                    .anyRequest().authenticated()
                    .and()
                .formLogin().and()
                .httpBasic().and()
                .csrf().disable();
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userService);
    }
}
{% endhighlight %}

Similar to any other Spring configuration file, this one is annoted with `@Configuration`. There is also another annotation, namely `@EnableWebSecurity`. Some older sources may tell you to use `@EnableWebMvcSecurity` if you'd like Spring Security to integrate with Spring MVC, but as of Spring Security 4.0, `@EnableWebMvcSecurity` is deprecated and `@EnableWebSecurity` is the single one which will also handle MVC.

6th line is responsible for injecting `UserDetailsService`. We're programming on an interface here, which benefits us in loose coupling of beans: the huge advantage here is application's flexibility, achieved by replacing concrete bean implementation with another one. `@Autowired` resolves bean by name - the default is class name started with lowercase letter. Of course, it's up to programmer to make the bean "visible" to the application context - we've achieved this with `@Service` annotation on top of `UserService` class. To make our configuration benefit from users source, we override the implementaion of `configure(AuthenticationManagerBuilder)` (line 23).

The crucial part of the code is overriden method `configure(HttpSecurity)`. In 12th line it says that we allow anyone to display our welcome page. Next line states that three endpoints can only be reached by a user with `ROLE_ADMIN` authority. Following line says that remaining requests can be done only by authenticated (logged-in) users (without defining their exact role). I've also decided to switch off CSRF for the conveniency of development - you can see it in line number 18.

That's basically all you need to know to allow to your webapp only those you'd like to. It's highly possible that you'd like to tune the visuals of login page to meet your expectations. You can start by taking a look at its default implementation and adding your HTML and CSS to it. It's mapped to `/login`, so make sure your custom file is resolved correctly.

Hope you liked this post and see you next time!
