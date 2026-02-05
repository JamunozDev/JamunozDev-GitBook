# Login
Como añadir un login a la aplicación
## Dependencias Maven
Agregar las dependencias necesarias en el pom.xml para Spring Security y Thymeleaf en Spring Boot 3.5.10. Incluir spring-boot-starter-security para autenticación y thymeleaf-extras-springsecurity6 para integración con templates.
```console
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
<dependency>
    <groupId>org.thymeleaf.extras</groupId>
    <artifactId>thymeleaf-extras-springsecurity6</artifactId>
</dependency>
```
Ejecutar mvn clean install para actualizar.

## Configuración de Seguridad
Crear una clase @Configuration con @EnableWebSecurity y @EnableMethodSecurity. Definir un SecurityFilterChain para form login, habilitar CSRF y configurar logout.
```console
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        return PasswordEncoderFactories.createDelegatingPasswordEncoder();
    }

    @Bean
    public AuthenticationManager authManager(AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.csrf(Customizer.withDefaults())
            .authorizeHttpRequests(auth -> auth.anyRequest().permitAll())
            .formLogin(form -> form.loginPage("/login")
                                  .usernameParameter("email")
                                  .permitAll())
            .logout(logout -> logout.logoutSuccessUrl("/login?logout"))
            .sessionManagement(session -> session.maximumSessions(1));
        return http.build();
    }
}
```
Esto permite acceso público inicial; protegeremos endpoints con @PreAuthorize después.
## Usuarios In-Memory (Opcional)
Para pruebas rápidas, configuraremos usuarios en memoria en SecurityConfig.
```console
@Bean
public UserDetailsService userDetailsService(PasswordEncoder encoder) {
    UserDetails user = User.withUsername("user")
                           .password(encoder.encode("password"))
                           .roles("USER")
                           .build();
    return new InMemoryUserDetailsManager(user);
}
```
## Controlador de Login
Crear un controlador para mostrar la página de login con mensajes de error/éxito. Mapear /login y redirigir según parámetros de Spring Security.
```console
@Controller
public class LoginController {
    @GetMapping("/login")
    public String login(String error, String logout, Model model) {
        if (error != null) model.addAttribute("message", "Error de login");
        if (logout != null) model.addAttribute("message", "Logout exitoso");
        model.addAttribute("requestUri", "/login");
        return "login";
    }
}
```
Proporciona requestUri para Thymeleaf.
## Plantilla de Login (Thymeleaf)
Crear src/main/resources/templates/login.html con formulario POST a /login. Usar sec:authorize y CSRF automático.
```console
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org" xmlns:sec="http://www.thymeleaf.org/extras/spring-security">
<head><title>Login</title></head>
<body>
    <div sec:authorize="!isAuthenticated()">
        <form th:action="@{/login}" method="post">
            <input type="hidden" th:name="${_csrf.parameterName}" th:value="${_csrf.token}"/>
            <label>Email: <input type="email" name="email"/></label>
            <label>Password: <input type="password" name="password"/></label>
            <button type="submit">Login</button>
        </form>
    </div>
    <p th:if="${message}" th:text="${message}"></p>
</body>
</html>
```
## Proteger Endpoints
En controladores existentes, usar @PreAuthorize("hasRole('USER')") en métodos. Por ejemplo, en un controlador protegido.
```console
@GetMapping("/protected")
@PreAuthorize("hasRole('USER')")
public String protectedPage(Principal principal) {
    return "protected";
}
```
Redirige a /login si no autenticado.
## Layout Compartido
En layout.html (o fragments), mostrar links condicionales con sec:authorize.
```console
<div sec:authorize="isAuthenticated()">
    Bienvenido <span sec:authentication="name"></span> |
    <form th:action="@{/logout}" method="post">
        <input type="hidden" th:name="${_csrf.parameterName}" th:value="${_csrf.token}"/>
        <button>Logout</button>
    </form>
</div>
<div sec:authorize="!isAuthenticated()">
    <a th:href="@{/login}">Login</a>
</div>
```
## Añadir persistencia al login
Para añadir persistencia al login (en lugar de usuarios en memoria) hay que leer los usuarios desde la base de datos mediante JPA + UserDetailsService.
### Entidad Usuario y Roles
Definir una entidad User que implemente UserDetails o que pueda adaptar a UserDetails.
```console
@Entity
@Table(name = "users")
public class UserEntity {   // o implementar UserDetails si se desea
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String username;   // o email

    @Column(nullable = false)
    private String password;   // BCrypt

    private boolean enabled = true;

    // relación con roles
    @ManyToMany(fetch = FetchType.EAGER)
    @JoinTable(
        name = "users_roles",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id"))
    private Set<RoleEntity> roles = new HashSet<>();
}
```
```console
@Entity
@Table(name = "roles")
public class RoleEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true)
    private String name;  // "ROLE_USER", "ROLE_ADMIN"
}
```
### Repositorios JPA
Usar Spring Data JPA para consultar usuarios por username/email.
```console
public interface UserRepository extends JpaRepository<UserEntity, Long> {
    Optional<UserEntity> findByUsername(String username); // o findByEmail
}
```
### Adaptar Usuario a UserDetails
Implementar UserDetails en la entidad o crear un wrapper. Ejemplo wrapper:
```console
public class CustomUserDetails implements UserDetails {

    private final UserEntity user;

    public CustomUserDetails(UserEntity user) {
        this.user = user;
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return user.getRoles().stream()
            .map(r -> new SimpleGrantedAuthority(r.getName()))
            .toList();
    }

    @Override
    public String getPassword() { return user.getPassword(); }

    @Override
    public String getUsername() { return user.getUsername(); }

    @Override
    public boolean isAccountNonExpired() { return true; }

    @Override
    public boolean isAccountNonLocked() { return true; }

    @Override
    public boolean isCredentialsNonExpired() { return true; }

    @Override
    public boolean isEnabled() { return user.isEnabled(); }
}
```
### Implementar UserDetailsService con JPA
Esta clase carga el usuario desde la BD para que Spring Security lo use durante el login.
```console
@Service
public class JpaUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    public JpaUserDetailsService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public UserDetails loadUserByUsername(String username)
            throws UsernameNotFoundException {

        UserEntity user = userRepository.findByUsername(username)
            .orElseThrow(() ->
                new UsernameNotFoundException("Usuario no encontrado: " + username));

        return new CustomUserDetails(user);
    }
}
```
### Configurar Security para usar la BD
En SecurityConfig, inyectar UserDetailsService y PasswordEncoder (BCrypt) y dejar que DaoAuthenticationProvider se configure automáticamente.
```console
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    private final UserDetailsService userDetailsService;

    public SecurityConfig(JpaUserDetailsService userDetailsService) {
        this.userDetailsService = userDetailsService;
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(); // ya lo estabas usando
    }

    @Bean
    public AuthenticationManager authManager(AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())  // según necesidad
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/login", "/register", "/css/**", "/js/**").permitAll()
                .anyRequest().authenticated()
            )
            .userDetailsService(userDetailsService)
            .formLogin(form -> form
                .loginPage("/login")
                .defaultSuccessUrl("/", true)
                .permitAll()
            )
            .logout(logout -> logout.logoutSuccessUrl("/login?logout"));

        return http.build();
    }
}
```
Con esto, cuando el usuario se loguea, Spring Security consulta la BD vía JpaUserDetailsService en lugar de InMemoryUserDetailsManager.
### Registro de Usuarios (guardar con BCrypt)
En el flujo de registro, codificar la contraseña antes de guardar el usuario.
```console
@Service
public class UserService {

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final RoleRepository roleRepository;

    public UserService(UserRepository userRepository,
                       PasswordEncoder passwordEncoder,
                       RoleRepository roleRepository) {
        this.userRepository = userRepository;
        this.passwordEncoder = passwordEncoder;
        this.roleRepository = roleRepository;
    }

    public UserEntity register(String username, String rawPassword) {
        UserEntity user = new UserEntity();
        user.setUsername(username);
        user.setPassword(passwordEncoder.encode(rawPassword));
        user.setEnabled(true);

        RoleEntity userRole = roleRepository.findByName("ROLE_USER")
            .orElseThrow();
        user.getRoles().add(userRole);

        return userRepository.save(user);
    }
}
```
El formulario de login Thymeleaf no cambia: sigue enviando username y password, pero ahora se validan contra la BD.