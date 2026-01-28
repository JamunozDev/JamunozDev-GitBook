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