# Laboratorio de Seguridad · OficioYa protegido con Spring Security y JWT

**Escuela Colombiana de Ingeniería Julio Garavito**  
**Asignatura:** Desarrollo y Operaciones de Software (DOSW)  
**Caso de estudio:** OficioYa  
**Hito del proyecto:** Sprint 3 · Construcción de seguridad + pruebas  
**Material base:** presentación *Seguridad en API REST con Spring Security* (semana 10)  
**Modalidad:** Desarrollo por Squads del proyecto · **duración máxima: 5 horas**

---

## 1. Descripción del reto

En OficioYa, trabajadores y contratantes deben autenticarse con usuario y contraseña, y el enunciado del proyecto exige que el **correo y el teléfono permanezcan ocultos** porque son información para la administración del sistema. Hasta ahora su API acepta cualquier petición de cualquier persona.

En este laboratorio construirán un **corte vertical de OficioYa** (cuentas, perfiles y solicitudes de servicio) y lo protegerán con **autenticación, autorización y JWT**, aplicando lo visto en clase. Después llevarán lo aprendido al dominio de su squad mediante una matriz de acceso (sección 12).

### Resultado esperado

- Registro e inicio de sesión de trabajadores y contratantes.
- Contraseñas almacenadas con hash (BCrypt), nunca en texto plano.
- JWT firmado, con expiración, validado en cada petición protegida.
- Autorización por **rol** y por **propiedad** de los recursos.
- Datos de contacto ocultos en los perfiles públicos.
- Errores de seguridad claros y consistentes (`400`, `401`, `403`, `404`, `409`).
- Pruebas automatizadas que demuestren la seguridad, no solo el funcionamiento correcto.

## 2. Resultados de aprendizaje

Al finalizar, el equipo podrá:

- Diferenciar codificar, cifrar y hashear, y aplicar BCrypt a las contraseñas.
- Diferenciar autenticación de autorización y responder **401**, **403** o **404** según corresponda.
- Generar y validar un JWT firmado, y explicar por qué no se puede falsificar.
- Configurar una `SecurityFilterChain` stateless con un filtro JWT.
- Proteger recursos por rol y por propiedad (evitar que un usuario acceda a datos de otro).
- Relacionar los controles implementados con los riesgos del OWASP Top 10.

## 3. Organización del equipo y del tiempo

El laboratorio está pensado para **5 horas de trabajo del equipo**. La clave es que, después de una base común, los cuatro integrantes trabajen **en paralelo** sin esperarse.

### 3.1 Responsabilidades en la fase 1

| Integrante | Bloques | Pruebas a su cargo | Depende de |
| --- | --- | --- | --- |
| **A · Cuentas** | 2 (registro) y la parte de login del 3 | PT-01 a PT-06 | Solo la base común |
| **B · Tokens** | `JwtService` (3) y Spring Security (4) | PT-07 a PT-09 | Solo la base común |
| **C · Perfiles** | 5 (perfiles, `/usuarios/me`, administración y creación del `ADMIN`) | PT-10 a PT-12 | Solo la base común |
| **D · Solicitudes** | 6 (solicitudes, propiedad y estados) | PT-13 a PT-17 | Solo la base común |

**Cómo evitar bloqueos:**

1. La base común (fase 0) deja listas las entidades, los repositorios, `ApiError` y una `SecurityConfig` con `@EnableMethodSecurity`. Con eso C y D pueden trabajar desde el minuto 45.
2. C y D prueban la autorización con `@WithMockUser` (ver bloque 7): **no necesitan** que el JWT esté terminado.
3. B publica el PR de `JwtService` lo antes posible (meta: 1:30). A lo necesita para el login.
4. Si alguien termina antes, apoya las pruebas o adelanta las matrices de documentación.

## 4. Qué van a construir

### 4.1 Roles

| Rol | Cómo se obtiene | Qué puede hacer |
| --- | --- | --- |
| `TRABAJADOR` | Registro público | Consultar perfiles, ver y responder las solicitudes que le envían |
| `CONTRATANTE` | Registro público | Consultar perfiles, crear, ver y cancelar sus solicitudes |
| `ADMIN` | Solo por configuración local (perfil `dev`); **nunca** por registro | Consultar los datos de contacto de los usuarios |

### 4.2 Estados de una solicitud

    [*] --> PENDIENTE: CONTRATANTE crea
    PENDIENTE --> ACEPTADA: TRABAJADOR asignado
    PENDIENTE --> RECHAZADA: TRABAJADOR asignado
    PENDIENTE --> CANCELADA: CONTRATANTE autor
    ACEPTADA --> CANCELADA: CONTRATANTE autor

Cualquier otro cambio de estado responde `409`.

## 5. Requerimientos de seguridad

| Código | Requerimiento | Criterio de aceptación |
| --- | --- | --- |
| RS-01 | Registrar usuario | `201` con el usuario sin contraseña ni hash. `409` si el correo existe. `400` si faltan datos, la contraseña tiene menos de 8 caracteres o el rol no es `TRABAJADOR` ni `CONTRATANTE` |
| RS-02 | Proteger contraseñas | Se guarda un hash BCrypt mediante `PasswordEncoder`; la base de datos no contiene texto plano |
| RS-03 | Iniciar sesión | `200` con un JWT firmado si las credenciales son válidas; `401` si no, con un mensaje genérico |
| RS-04 | Validar el JWT en cada petición | Se verifican firma, expiración y emisor. Falta de token, token alterado o vencido responden `401` |
| RS-05 | Reglas de acceso | Solo registro y login son públicos; todo lo demás exige token; `/api/v1/admin/**` solo `ADMIN`; sesiones stateless |
| RS-06 | Ocultar datos de contacto | El perfil público no incluye correo ni teléfono; los ven solo el `ADMIN` y el propio usuario en `/usuarios/me` |
| RS-07 | Aislar solicitudes por propiedad | Un usuario solo ve y modifica las solicitudes en las que participa; una ajena responde `404` |
| RS-08 | Autorizar por rol en solicitudes | Solo `CONTRATANTE` crea y cancela; solo el `TRABAJADOR` asignado acepta o rechaza; `ADMIN` no accede a solicitudes |
| RS-09 | Errores de seguridad uniformes | Todas las respuestas de error (incluidas `401` y `403`) usan el mismo cuerpo JSON, sin trazas |
| RS-10 | Proteger secretos y logs | La clave del JWT y las credenciales no están en el repositorio; los logs no contienen tokens ni contraseñas |
| RS-11 | Validar con pruebas | Se cubre la matriz de pruebas del bloque 7 |

### 5.1 Reglas de seguridad obligatorias

1. El correo se normaliza (sin espacios y en minúsculas) y es único.
2. La identidad del usuario sale **siempre** del contexto de seguridad; nunca de un id enviado en el cuerpo o en la URL.
3. El JWT usa **HS256**, tiene los claims `sub` (correo), `role`, `iss` (`oficioya`), `iat` y `exp`, y dura **30 minutos**.
4. La clave de firma tiene al menos 32 bytes y llega por la variable de entorno `JWT_SECRET`.
5. El mensaje de un login fallido es el mismo si el correo no existe o si la contraseña es incorrecta.
6. El `ADMIN` se crea al arrancar **solo** en el perfil `dev`, con los valores de las variables `ADMIN_CORREO` y `ADMIN_PASSWORD`.
7. El payload del JWT es legible por cualquiera: no lleva contraseñas, teléfono ni datos sensibles.

**No se permite:** contraseñas en texto plano, `NoOpPasswordEncoder`, `anyRequest().permitAll()`, un secreto dentro del código o del repositorio, aceptar `ADMIN` en el registro público, ni devolver entidades JPA directamente en las respuestas.

## 6. Contrato REST

Las rutas llevan el prefijo `/api/v1`.

| Método | Endpoint | Acceso | Éxito | Errores |
| --- | --- | --- | --- | --- |
| `POST` | `/auth/registro` | Público | `201` | `400`, `409` |
| `POST` | `/auth/login` | Público | `200` con JWT | `400`, `401` |
| `GET` | `/usuarios/me` | Autenticado | `200` con datos propios (incluye contacto) | `401` |
| `GET` | `/trabajadores` | Autenticado | `200` con perfiles públicos | `401` |
| `GET` | `/trabajadores/{id}` | Autenticado | `200` con perfil público | `401`, `404` |
| `GET` | `/admin/usuarios` | `ADMIN` | `200` con datos de contacto | `401`, `403` |
| `POST` | `/solicitudes` | `CONTRATANTE` | `201` | `400`, `401`, `403`, `404` |
| `GET` | `/solicitudes` | `CONTRATANTE`, `TRABAJADOR` | `200` solo las mías | `401`, `403` |
| `GET` | `/solicitudes/{id}` | Parte de la solicitud | `200` | `401`, `403`, `404` |
| `PATCH` | `/solicitudes/{id}/aceptar` | `TRABAJADOR` asignado | `200` | `401`, `403`, `404`, `409` |
| `PATCH` | `/solicitudes/{id}/rechazar` | `TRABAJADOR` asignado | `200` | `401`, `403`, `404`, `409` |
| `PATCH` | `/solicitudes/{id}/cancelar` | `CONTRATANTE` autor | `200` | `401`, `403`, `404`, `409` |

Dos convenciones:

- El **rol equivocado** responde `403` (sabemos quién es, pero su rol no alcanza).
- Una solicitud **ajena o inexistente** responde `404`, para no revelar que existe.

### Ejemplos

**Registro** (`POST /api/v1/auth/registro`). Los campos `oficioPrincipal` y `zona` son opcionales y solo aplican a trabajadores:

```json
{
  "nombre": "Pedro Gómez",
  "correo": "pedro@example.com",
  "telefono": "3001234567",
  "password": "ContrasenaDemo123!",
  "rol": "TRABAJADOR",
  "oficioPrincipal": "Plomero",
  "zona": "Chapinero"
}
```

Respuesta (sin secretos): `{ "id": 3, "nombre": "Pedro Gómez", "correo": "pedro@example.com", "rol": "TRABAJADOR" }`

**Login** (`POST /api/v1/auth/login`):

```json
{ "correo": "pedro@example.com", "password": "ContrasenaDemo123!" }
```

Respuesta: `{ "tipo": "Bearer", "accessToken": "<JWT_FIRMADO>", "expiraEn": 1800 }`. El valor `<JWT_FIRMADO>` es un marcador: no suban tokens reales al repositorio.

**Perfil público** (`GET /api/v1/trabajadores/3`): `{ "id": 3, "nombre": "Pedro Gómez", "oficioPrincipal": "Plomero", "zona": "Chapinero" }`

**Crear solicitud** (`POST /api/v1/solicitudes`, con `Authorization: Bearer <JWT_FIRMADO>`). El contratante **no** se envía: sale del token.

```json
{ "trabajadorId": 3, "descripcion": "Tengo una gotera en el baño" }
```

**Formato único de error:**

```json
{
  "timestamp": "2026-10-23T10:15:30",
  "status": 401,
  "error": "Unauthorized",
  "mensaje": "Token inválido o expirado",
  "ruta": "/api/v1/solicitudes"
}
```

## 7. Modelo de datos

Ids `Long`, autogenerados e incrementales (como en el laboratorio de persistencia). Los enums se guardan como texto.

```text
┌──────────────────────────┐                ┌──────────────────────────┐
│ usuarios                 │                │ solicitudes              │
├──────────────────────────┤                ├──────────────────────────┤
│ id               PK      │◄── 1 ─── N ────│ contratante_id   FK      │
│ nombre                   │◄── 1 ─── N ────│ trabajador_id    FK      │
│ correo           UNIQUE  │                │ id               PK      │
│ telefono                 │                │ descripcion              │
│ password_hash            │                │ estado                   │
│ rol                      │                │ fecha_creacion           │
│ oficio_principal (nulo)  │                └──────────────────────────┘
│ zona             (nulo)  │
│ fecha_creacion           │
└──────────────────────────┘
```

| Tabla | Reglas |
| --- | --- |
| `usuarios` | `nombre`, `correo`, `telefono`, `password_hash` y `rol` obligatorios. `correo` único y normalizado. `rol`: `TRABAJADOR`, `CONTRATANTE` o `ADMIN`. `oficio_principal` y `zona` pueden ser nulos |
| `solicitudes` | `contratante_id` y `trabajador_id` obligatorios. `estado`: `PENDIENTE`, `ACEPTADA`, `RECHAZADA` o `CANCELADA` |

## 8. Estructura del proyecto

Usen la estructura de paquetes del curso, con la base `edu.eci.dosw.oficioya`:

| Paquete | Qué debe contener |
| --- | --- |
| `config` | `SecurityConfig` (cadena de filtros, reglas, `PasswordEncoder`), propiedades del JWT y creación del `ADMIN` |
| `security` | `JwtService`, `JwtAuthenticationFilter`, manejadores de `401` y `403` |
| `controller` | `AuthController`, `UsuarioController`, `TrabajadorController`, `AdminController`, `SolicitudController` |
| `service` | Lógica de registro, login, perfiles y solicitudes |
| `repository` | `UsuarioRepository`, `SolicitudRepository` |
| `entity` | `Usuario`, `Solicitud` y los enums `Rol`, `EstadoSolicitud` |
| `dto` | Peticiones y respuestas, incluidos el perfil público, el de administración y `ApiError` |
| `mapper` | Conversión entidad ↔ DTO (MapStruct o manual) |
| `exception` | Excepciones propias y `@RestControllerAdvice` |

Configuración local mínima (`src/main/resources/application.properties`). Las variables sin valor por defecto son obligatorias:

```properties
spring.datasource.url=${DB_URL:jdbc:postgresql://localhost:5432/oficioya_security}
spring.datasource.username=${DB_USER:postgres}
spring.datasource.password=${DB_PASSWORD}
spring.jpa.hibernate.ddl-auto=update

app.jwt.secret=${JWT_SECRET}
app.jwt.issuer=oficioya
app.jwt.ttl-minutes=30
```

Base de datos con Docker (ajusten el nombre):

```bash
docker run --name postgres-oficioya -e POSTGRES_DB=oficioya_security \
  -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=<su_clave> -p 5432:5432 -d postgres:17
```

Para arrancar en local con el `ADMIN`: `SPRING_PROFILES_ACTIVE=dev mvn spring-boot:run` (con las variables del `.env` cargadas en la terminal).

## 9. Desarrollo paso a paso

Cada bloque va en su rama (`feature/...`) y se mezcla en `develop` con un PR que revisa otro integrante. Las pistas indican **qué buscar**, no la solución.

Cada bloque trae un **ejemplo con otro dominio** (pedidos de una tienda, con roles `CLIENTE`, `VENDEDOR` y `ADMIN`) para ilustrar el patrón. No es la solución de OficioYa: deben adaptarlo a sus roles, entidades y reglas.

> **Versión de Spring Boot.** Los ejemplos están probados con **Spring Boot 4.1**, que es lo que genera hoy Spring Initializr. Si su proyecto usa Spring Boot 3.5, revisen el [Anexo A](#anexo-a--diferencias-entre-spring-boot-3-y-4).

### Bloque 0 · Arranque (`feature/setup`) · Fase 0 · Todos

1. Creen el repositorio del equipo con los archivos `README` y `.gitignore`, y la rama `develop`.
2. Generen el proyecto con Spring Initializr (Maven, Java 21) con las dependencias **Spring Web**, **Spring Data JPA**, **Validation** y **PostgreSQL Driver**.

   <img width="642" height="178" alt="Dependencias en Spring Initializr" src="https://github.com/user-attachments/assets/de9f54c2-06e6-4b8b-a622-cce493507ee7" />

3. Agreguen las dependencias de Spring Security, JWT y pruebas del ejemplo de abajo.
4. Creen los paquetes de la sección 8, levanten PostgreSQL y compilen.
5. Excluyan del repositorio `.env`, `target/` y `logs/` incluyéndolos en `.gitignore`, y creen un `.env.example` sin valores secretos.

**Ejemplo · dependencias, variables y `.gitignore`:**

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
  <groupId>io.jsonwebtoken</groupId>
  <artifactId>jjwt-api</artifactId>
  <version>0.12.6</version>
</dependency>
<dependency>
  <groupId>io.jsonwebtoken</groupId>
  <artifactId>jjwt-impl</artifactId>
  <version>0.12.6</version>
  <scope>runtime</scope>
</dependency>
<dependency>
  <groupId>io.jsonwebtoken</groupId>
  <artifactId>jjwt-jackson</artifactId>
  <version>0.12.6</version>
  <scope>runtime</scope>
</dependency>

<!-- Pruebas -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-test</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-webmvc-test</artifactId>   <!-- MockMvc en Boot 4 -->
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-security-test</artifactId> <!-- @WithMockUser en Boot 4 -->
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.springframework.security</groupId>
  <artifactId>spring-security-test</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>com.h2database</groupId>
  <artifactId>h2</artifactId>
  <scope>test</scope>
</dependency>
```

```bash
# .env.example (se sube a Git, sin valores secretos)
DB_PASSWORD=
JWT_SECRET=
ADMIN_CORREO=
ADMIN_PASSWORD=

# .gitignore (agregar estas líneas)
.env
target/
logs/

# Generar localmente una clave para JWT_SECRET (no se sube a Git)
openssl rand -base64 48
```

### Bloque 1 · Persistencia y base de seguridad (`feature/persistence`) · Fase 0 · Todos

1. Creen `Usuario`, `Solicitud` y los enums, con las anotaciones JPA del laboratorio de persistencia.
2. Creen los repositorios con las consultas que necesitarán: usuario por correo y solicitudes **filtradas por participante**.
3. Configuren H2 para las pruebas (`application-test.properties`).
4. Creen el DTO `ApiError` (formato único de error) y una `SecurityConfig` inicial con `@EnableMethodSecurity`, el `PasswordEncoder` y las reglas de acceso. B la completará con el filtro y los manejadores en el bloque 4.

*Pista:* con esta base mezclada en `develop`, los cuatro integrantes pueden trabajar en paralelo.

**Ejemplo · entidad con propietario, repositorio y perfil de pruebas:**

```java
@Entity
@Table(name = "pedidos")
public class Pedido {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String descripcion;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private EstadoPedido estado;

    @ManyToOne(optional = false)
    @JoinColumn(name = "cliente_id", nullable = false)
    private Usuario cliente;

    // constructor vacío, getters y setters
}
```

```java
public interface PedidoRepository extends JpaRepository<Pedido, Long> {

    List<Pedido> findByClienteId(Long clienteId);

    Optional<Pedido> findByIdAndClienteId(Long id, Long clienteId);
}
```

```properties
# src/test/resources/application-test.properties
spring.datasource.url=jdbc:h2:mem:testdb
spring.datasource.driverClassName=org.h2.Driver
spring.jpa.hibernate.ddl-auto=create-drop

# Clave solo para pruebas (no es la de producción)
app.jwt.secret=clave-solo-para-pruebas-de-al-menos-32-bytes
app.jwt.issuer=oficioya
app.jwt.ttl-minutes=30
```

**Ejemplo · formato único de error:**

```java
public record ApiError(
        LocalDateTime timestamp,
        int status,
        String error,
        String mensaje,
        String ruta) {

    public static ApiError of(int status, String error, String mensaje, String ruta) {
        return new ApiError(LocalDateTime.now(), status, error, mensaje, ruta);
    }
}
```

> No lo llamen `ErrorResponse`: Spring ya tiene una interfaz `org.springframework.web.ErrorResponse` y el IDE puede importar la equivocada.

### Bloque 2 · Registro y contraseñas (`feature/auth-registration`) · Integrante A

1. Creen los DTO de registro y de respuesta, con validaciones (`@Valid`).
2. Usen el bean `PasswordEncoder` (BCrypt) al guardar la contraseña.
3. Implementen `POST /auth/registro` con las reglas de RS-01.
4. Creen las excepciones propias y completen el `@RestControllerAdvice` con el formato de error único.

*Pista:* la respuesta de registro es un DTO. Si devuelven la entidad, el hash viaja al cliente.

**Ejemplo · DTO validado, `PasswordEncoder`, lista de roles públicos y manejo de errores:**

```java
public enum Rol { CLIENTE, VENDEDOR, ADMIN }
```

```java
public record RegistroRequest(
        @NotBlank String nombre,
        @NotBlank @Email String correo,
        @NotBlank @Size(min = 8) String password,
        @NotNull Rol rol) {}
```

```java
@Bean
PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

```java
private static final Set<Rol> ROLES_PUBLICOS = EnumSet.of(Rol.CLIENTE, Rol.VENDEDOR);

public UsuarioResponse registrar(RegistroRequest req) {
    if (!ROLES_PUBLICOS.contains(req.rol())) {
        throw new RolNoPermitidoException("Rol no permitido en el registro");   // 400
    }
    String correo = req.correo().trim().toLowerCase();
    if (usuarioRepository.existsByCorreo(correo)) {
        throw new CorreoDuplicadoException(correo);                            // 409
    }
    Usuario u = new Usuario();
    u.setNombre(req.nombre());
    u.setCorreo(correo);
    u.setPasswordHash(passwordEncoder.encode(req.password()));
    u.setRol(req.rol());                         // el cliente propone; el servidor solo acepta roles públicos
    return mapper.toResponse(usuarioRepository.save(u));
}
```

Se usa una lista de roles **permitidos** y no un `if (rol == ADMIN)`: si mañana aparece un rol privado nuevo, queda bloqueado por defecto.

```java
@RestControllerAdvice
public class ApiExceptionHandler {

    @ExceptionHandler(CorreoDuplicadoException.class)
    ResponseEntity<ApiError> duplicado(CorreoDuplicadoException ex, HttpServletRequest req) {
        return ResponseEntity.status(HttpStatus.CONFLICT).body(
                ApiError.of(409, "Conflict", "El correo ya está registrado", req.getRequestURI()));
    }

    @ExceptionHandler(RolNoPermitidoException.class)
    ResponseEntity<ApiError> rolNoPermitido(RolNoPermitidoException ex, HttpServletRequest req) {
        return ResponseEntity.badRequest().body(
                ApiError.of(400, "Bad Request", ex.getMessage(), req.getRequestURI()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    ResponseEntity<ApiError> datosInvalidos(MethodArgumentNotValidException ex, HttpServletRequest req) {
        return ResponseEntity.badRequest().body(
                ApiError.of(400, "Bad Request", "Datos inválidos", req.getRequestURI()));
    }

    @ExceptionHandler(HttpMessageNotReadableException.class)   // p. ej. "rol": "JEFE"
    ResponseEntity<ApiError> cuerpoIlegible(HttpMessageNotReadableException ex, HttpServletRequest req) {
        return ResponseEntity.badRequest().body(
                ApiError.of(400, "Bad Request", "Cuerpo de la petición inválido", req.getRequestURI()));
    }
}
```

El último manejador es necesario porque un valor que no existe en el enum no llega a la validación: falla antes, al leer el JSON.

### Bloque 3 · JWT y login (`feature/jwt` y `feature/login`) · Integrantes B y A

1. **B:** lean la clave, el emisor y la duración desde las propiedades (la clave, de `JWT_SECRET`).
2. **B:** implementen en `JwtService` la generación (con los claims de la regla 3) y la validación (firma, expiración y emisor). Publiquen el PR cuanto antes.
3. **A:** implementen `POST /auth/login` con `PasswordEncoder.matches` y un mensaje genérico en caso de fallo.

*Pista:* hagan configurable la duración para poder generar un token **ya vencido** en las pruebas.

**Ejemplo · `JwtService` con `jjwt` 0.12 y login:**

```java
@Service
public class JwtService {

    private final SecretKey key;
    private final String issuer;
    private final long ttlMinutes;

    public JwtService(@Value("${app.jwt.secret}") String secret,
                      @Value("${app.jwt.issuer}") String issuer,
                      @Value("${app.jwt.ttl-minutes}") long ttlMinutes) {
        this.key = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
        this.issuer = issuer;
        this.ttlMinutes = ttlMinutes;
    }

    public String generar(String correo, String rol) {
        Instant ahora = Instant.now();
        return Jwts.builder()
                .subject(correo)
                .claim("role", rol)
                .issuer(issuer)
                .issuedAt(Date.from(ahora))
                .expiration(Date.from(ahora.plus(ttlMinutes, ChronoUnit.MINUTES)))
                .signWith(key, Jwts.SIG.HS256)   // fija HS256 sin importar el largo de la clave
                .compact();
    }

    public Claims validar(String token) {
        return Jwts.parser()
                .verifyWith(key)
                .requireIssuer(issuer)
                .build()
                .parseSignedClaims(token)      // lanza JwtException si la firma es inválida o venció
                .getPayload();
    }
}
```

> Sin el segundo argumento de `signWith`, jjwt elige el algoritmo según el largo de la clave: con la clave de 64 caracteres que genera `openssl rand -base64 48` firmaría con HS512 y no cumpliría la regla 3.

```java
public LoginResponse login(LoginRequest req) {
    Usuario u = usuarioRepository.findByCorreo(req.correo().trim().toLowerCase())
            .filter(x -> passwordEncoder.matches(req.password(), x.getPasswordHash()))
            .orElseThrow(CredencialesInvalidasException::new);   // mismo error si falla el correo o la clave
    String token = jwtService.generar(u.getCorreo(), u.getRol().name());
    return new LoginResponse("Bearer", token, ttlMinutes * 60);
}
```

### Bloque 4 · Spring Security (`feature/security-config`) · Integrante B

1. Completen la `SecurityFilterChain`: sesiones stateless, `csrf` desactivado (justifíquenlo en el informe), `/auth/**` público, `/admin/**` solo `ADMIN` y el resto autenticado.
2. Implementen `JwtAuthenticationFilter` (`OncePerRequestFilter`): leer `Authorization: Bearer`, validar el token y dejar el usuario en el `SecurityContext` con su autoridad (`ROLE_...`). Regístrenlo con `addFilterBefore`.
3. Implementen un `AuthenticationEntryPoint` (`401`) y un `AccessDeniedHandler` (`403`) que respondan con `ApiError`.

*Pista:* sin configurar el punto de entrada, Spring puede responder **403** a una petición sin token. Comprueben que responde **401**.

**Ejemplo · cadena de filtros, filtro JWT y punto de entrada (imports omitidos):**

```java
@Configuration
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http,
                                    JwtAuthenticationFilter jwtFilter,
                                    AuthenticationEntryPoint entryPoint,
                                    AccessDeniedHandler deniedHandler) throws Exception {
        return http
                .csrf(csrf -> csrf.disable())
                .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers("/api/v1/auth/**").permitAll()
                        .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                        .anyRequest().authenticated())
                .exceptionHandling(e -> e
                        .authenticationEntryPoint(entryPoint)      // 401
                        .accessDeniedHandler(deniedHandler))        // 403
                .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
                .build();
    }

    @Bean
    PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtService jwtService;

    public JwtAuthenticationFilter(JwtService jwtService) {
        this.jwtService = jwtService;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        String header = request.getHeader("Authorization");
        if (header != null && header.startsWith("Bearer ")) {
            try {
                Claims claims = jwtService.validar(header.substring(7));
                var autoridad = new SimpleGrantedAuthority("ROLE_" + claims.get("role", String.class));
                var auth = new UsernamePasswordAuthenticationToken(
                        claims.getSubject(), null, List.of(autoridad));
                SecurityContextHolder.getContext().setAuthentication(auth);
            } catch (JwtException | IllegalArgumentException e) {
                SecurityContextHolder.clearContext();   // sin autenticación: el punto de entrada responderá 401
            }
        }
        chain.doFilter(request, response);
    }
}
```

```java
@Component
public class JsonAuthenticationEntryPoint implements AuthenticationEntryPoint {

    private final ObjectMapper objectMapper;   // Boot 4: tools.jackson.databind.ObjectMapper

    public JsonAuthenticationEntryPoint(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    @Override
    public void commence(HttpServletRequest req, HttpServletResponse res,
                         AuthenticationException ex) throws IOException {
        res.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        res.setContentType(MediaType.APPLICATION_JSON_VALUE);
        res.setCharacterEncoding("UTF-8");
        objectMapper.writeValue(res.getWriter(),
                ApiError.of(401, "Unauthorized", "Token inválido o expirado", req.getRequestURI()));
    }
}
```

El `AccessDeniedHandler` es análogo, con `403`.

### Bloque 5 · Perfiles y administración (`feature/profiles-admin`) · Integrante C

1. Implementen `GET /usuarios/me` con los datos del usuario autenticado (incluido su contacto).
2. Implementen `GET /trabajadores` y `GET /trabajadores/{id}` con un DTO público **sin** correo ni teléfono.
3. Implementen `GET /admin/usuarios` con un DTO que sí incluya los datos de contacto, protegido por URL, por `@PreAuthorize` o por ambos.
4. Creen el `ADMIN` al arrancar, solo en el perfil `dev` (regla 6).

**Ejemplo · un DTO público, otro para administración y un endpoint por rol:**

```java
public record PerfilPublico(Long id, String nombre, String ciudad) {}

public record PerfilAdmin(Long id, String nombre, String ciudad, String correo, String telefono) {}
```

```java
@RestController
@RequestMapping("/api/v1/admin")
public class AdminController {

    private final UsuarioService usuarioService;

    public AdminController(UsuarioService usuarioService) {
        this.usuarioService = usuarioService;
    }

    @PreAuthorize("hasRole('ADMIN')")
    @GetMapping("/usuarios")
    public List<PerfilAdmin> listar() {
        return usuarioService.listarParaAdmin();
    }
}
```

**Ejemplo · crear el `ADMIN` solo en desarrollo:**

```java
@Configuration
@Profile("dev")                                   // nunca en pruebas ni en producción
public class AdminSeeder {

    @Bean
    CommandLineRunner crearAdmin(UsuarioRepository repo, PasswordEncoder encoder,
                                 @Value("${ADMIN_CORREO}") String correo,
                                 @Value("${ADMIN_PASSWORD}") String password) {
        return args -> {
            String normalizado = correo.trim().toLowerCase();
            if (repo.existsByCorreo(normalizado)) {
                return;                           // idempotente: no duplica al reiniciar
            }
            Usuario admin = new Usuario();
            admin.setNombre("Administrador");
            admin.setCorreo(normalizado);
            admin.setPasswordHash(encoder.encode(password));
            admin.setRol(Rol.ADMIN);
            repo.save(admin);
        };
    }
}
```

Ojo: si su `@RestControllerAdvice` captura `Exception` de forma genérica, también capturará `AccessDeniedException` y responderá `500` en lugar de `403`.

### Bloque 6 · Solicitudes (`feature/service-requests`) · Integrante D

1. Implementen los endpoints de solicitudes de la sección 6 y las transiciones de estado de la sección 4.2.
2. Tomen el contratante o el trabajador **del contexto de seguridad**, no del cuerpo de la petición.
3. Hagan que las consultas incluyan al participante, de modo que una solicitud ajena y una inexistente produzcan el mismo `404`.
4. Respondan `403` cuando el rol no corresponde, y `409` cuando la transición de estado no es válida.

*Advertencia:* filtrar en el servicio después de traer el registro es frágil. La restricción debe estar en la consulta o en una verificación explícita antes de devolver o modificar datos.

**Ejemplo · propiedad en la consulta, identidad del contexto y transición de estado:**

```java
@RestController
@RequestMapping("/api/v1/pedidos")
public class PedidoController {

    private final PedidoService pedidoService;

    public PedidoController(PedidoService pedidoService) {
        this.pedidoService = pedidoService;
    }

    @PreAuthorize("hasRole('CLIENTE')")                    // otro rol: 403
    @PostMapping
    public ResponseEntity<PedidoResponse> crear(@RequestBody @Valid PedidoRequest req,
                                                Authentication auth) {
        // la identidad sale del contexto de seguridad, no del cuerpo de la petición
        return ResponseEntity.status(HttpStatus.CREATED).body(pedidoService.crear(req, auth.getName()));
    }

    @PreAuthorize("hasRole('CLIENTE')")
    @GetMapping("/{id}")
    public PedidoResponse obtener(@PathVariable Long id, Authentication auth) {
        return pedidoService.obtener(id, auth.getName());
    }
}
```

```java
public PedidoResponse obtener(Long id, String correoAutenticado) {
    Usuario actual = usuarioService.porCorreo(correoAutenticado);
    return pedidoRepository.findByIdAndClienteId(id, actual.getId())    // la consulta incluye al propietario
            .map(mapper::toResponse)
            .orElseThrow(() -> new RecursoNoEncontradoException("Pedido no encontrado")); // ajeno o inexistente: mismo 404
}

public PedidoResponse cancelar(Long id, String correoAutenticado) {
    Usuario actual = usuarioService.porCorreo(correoAutenticado);
    Pedido pedido = pedidoRepository.findByIdAndClienteId(id, actual.getId())
            .orElseThrow(() -> new RecursoNoEncontradoException("Pedido no encontrado"));
    if (pedido.getEstado() != EstadoPedido.PENDIENTE) {
        throw new EstadoInvalidoException("Solo se puede cancelar un pedido pendiente");   // 409
    }
    pedido.setEstado(EstadoPedido.CANCELADO);
    return mapper.toResponse(pedidoRepository.save(pedido));
}
```

En OficioYa una solicitud tiene **dos** participantes (contratante y trabajador): piensen qué consulta necesita cada rol.

### Bloque 7 · Pruebas de seguridad · Cada integrante en su rama

Cada integrante escribe las pruebas de su bloque (ver sección 3.2). Usen JUnit 5, Mockito y MockMvc, con el perfil `test` (H2).

- Para probar el **token** (PT-07 a PT-09), generen tokens con `JwtService`.
- Para probar **roles y propiedad** (PT-10 a PT-17), pueden usar `@WithMockUser`, que simula un usuario autenticado sin necesidad de JWT.

| ID | Prueba | Resultado esperado | Responsable |
| --- | --- | --- | --- |
| PT-01 | Registro válido | `201`, rol correcto, sin contraseña ni hash en la respuesta | A |
| PT-02 | Registro con correo repetido | `409` | A |
| PT-03 | Registro con rol `ADMIN` o contraseña corta | `400` | A |
| PT-04 | Contraseña guardada | Es un hash BCrypt, distinto del texto original | A |
| PT-05 | Login correcto | `200` con token | A |
| PT-06 | Login con contraseña errónea y con correo inexistente | `401` sin token y el mismo mensaje | A |
| PT-07 | Recurso protegido sin token | `401` con el formato `ApiError` | B |
| PT-08 | Token alterado (cambiar un carácter) | `401` | B |
| PT-09 | Token vencido | `401` | B |
| PT-10 | Perfil público de un trabajador | `200` sin `correo` ni `telefono` | C |
| PT-11 | `CONTRATANTE` consulta `/admin/usuarios` | `403` | C |
| PT-12 | `ADMIN` consulta `/admin/usuarios` | `200` con datos de contacto | C |
| PT-13 | `TRABAJADOR` intenta crear una solicitud | `403` | D |
| PT-14 | Contratante lista solicitudes | Solo las propias | D |
| PT-15 | Consultar o cancelar una solicitud ajena | `404`, sin cambios en la base de datos | D |
| PT-16 | Aceptar con un trabajador no asignado | `404`; con un contratante, `403` | D |
| PT-17 | `ADMIN` consulta `/solicitudes` | `403` | D |

Comando de referencia: `mvn clean verify`.

**Ejemplo · pruebas del token con MockMvc (token alterado, vencido y recurso ajeno):**

```java
package edu.eci.dosw.oficioya;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

import edu.eci.dosw.oficioya.security.JwtService;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.webmvc.test.autoconfigure.AutoConfigureMockMvc;   // Boot 4
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.web.servlet.MockMvc;

@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("test")
class SeguridadPedidosTest {

    @Autowired MockMvc mockMvc;
    @Autowired JwtService jwtService;
    @Value("${app.jwt.secret}") String secret;

    @Test
    @DisplayName("Un token con la firma alterada debe responder 401")
    void tokenAlterado() throws Exception {
        String token = jwtService.generar("ana@example.com", "CLIENTE");
        mockMvc.perform(get("/api/v1/pedidos/1").header("Authorization", "Bearer " + alterar(token)))
               .andExpect(status().isUnauthorized());
    }

    @Test
    @DisplayName("Un token vencido debe responder 401")
    void tokenVencido() throws Exception {
        String vencido = new JwtService(secret, "oficioya", -1).generar("ana@example.com", "CLIENTE");
        mockMvc.perform(get("/api/v1/pedidos/1").header("Authorization", "Bearer " + vencido))
               .andExpect(status().isUnauthorized());
    }

    @Test
    @DisplayName("Consultar un pedido ajeno debe responder 404")
    void pedidoAjeno() throws Exception {
        Long idDeAna = crearPedidoDe("ana@example.com");          // helper que escriben ustedes
        String tokenDeLuis = jwtService.generar("luis@example.com", "CLIENTE");
        mockMvc.perform(get("/api/v1/pedidos/" + idDeAna).header("Authorization", "Bearer " + tokenDeLuis))
               .andExpect(status().isNotFound());
    }

    private String alterar(String token) {
        int i = token.length() - 10;                                // un carácter dentro de la firma
        char c = token.charAt(i) == 'A' ? 'B' : 'A';
        return token.substring(0, i) + c + token.substring(i + 1);
    }
}
```

Los usuarios `ana@example.com` y `luis@example.com` deben existir en la base de pruebas antes de ejecutar `pedidoAjeno` (por ejemplo, creándolos en un método `@BeforeEach`).

**Ejemplo · pruebas de rol y de datos ocultos con `@WithMockUser` (no necesitan JWT):**

```java
@Test
@WithMockUser(username = "vera@example.com", roles = "VENDEDOR")
@DisplayName("Un VENDEDOR no puede crear pedidos: 403")
void vendedorNoCreaPedidos() throws Exception {
    mockMvc.perform(post("/api/v1/pedidos")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("{\"descripcion\":\"Cambiar la llave del lavamanos\"}"))
           .andExpect(status().isForbidden());
}

@Test
@WithMockUser(roles = "CLIENTE")
@DisplayName("El perfil público no expone datos de contacto")
void perfilPublicoSinContacto() throws Exception {
    mockMvc.perform(get("/api/v1/perfiles/7"))
           .andExpect(status().isOk())
           .andExpect(jsonPath("$.correo").doesNotExist())
           .andExpect(jsonPath("$.telefono").doesNotExist());
}
```

`@WithMockUser` deja `username` como `auth.getName()` y agrega el prefijo `ROLE_` a los roles, igual que su filtro JWT. Recuerden que no reemplaza las pruebas PT-07 a PT-09: esas sí deben pasar por el token.

### Bloque 8 · Ataques manuales · Fase 2 · Todos, en parejas

Con `curl`, Postman o un archivo `.http`, contra la aplicación corriendo con el perfil `dev`, registren la respuesta de cada intento y compárenla con lo esperado:

| # | Ataque | Esperado |
| --- | --- | --- |
| 1 | Pedir `/solicitudes` sin token | `401` |
| 2 | Pegar su token en [jwt.io](https://jwt.io), cambiar `role` a `ADMIN` y enviarlo | `401` (la firma ya no coincide) |
| 3 | Usar un token vencido (reduzcan la duración temporalmente) | `401` |
| 4 | Con el token de un contratante B, pedir una solicitud del contratante A | `404` |
| 5 | Con un contratante, llamar a `/admin/usuarios` | `403` |
| 6 | Registrarse enviando `"rol": "ADMIN"` | `400` |
| 7 | Hacer login con un correo inexistente y con uno existente pero clave errónea | Mismo `401` y mismo mensaje |

**Ejemplo · ataques 1 y 4 con `curl`:**

```bash
# Iniciar sesión como Luis y guardar el token en una variable (no lo peguen completo en el informe)
TOKEN_LUIS=$(curl -s -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"correo":"luis@example.com","password":"ContrasenaDemo123!"}' | jq -r .accessToken)

# Ataque 1: sin token
curl -i http://localhost:8080/api/v1/pedidos
# HTTP/1.1 401

# Ataque 4: el token de Luis sobre un pedido de Ana
curl -i -H "Authorization: Bearer $TOKEN_LUIS" http://localhost:8080/api/v1/pedidos/1
# HTTP/1.1 404
```

Guarden los comandos y las respuestas obtenidas en `docs/ataques.md` (o la colección de Postman en `docs/postman/`).

### Bloque 9 · Documentación (`feature/docs`) · Fase 3 · En parejas

| Pareja | Entregable |
| --- | --- |
| A y B | Matriz OWASP (sección 11) y preguntas 1 a 3 del informe |
| C y D | Matriz de acceso de su dominio (sección 12) y preguntas 4 a 6 del informe |

Además, actualicen el `README` de su repositorio con: cómo ejecutar el proyecto, las variables de entorno (sin sus valores), el contrato de endpoints, el resultado de `mvn clean verify` y los enlaces a las evidencias, las matrices y el informe.

**Ejemplo · una fila de cada matriz y el formato de una respuesta del informe:**

| Riesgo | ¿Dónde podría ocurrir? | Control implementado | Prueba o evidencia |
| --- | --- | --- | --- |
| A01 · Control de acceso roto | Un cliente podría pedir `/pedidos/7` de otro cliente cambiando el id | La consulta incluye al propietario y responde `404` | PT-15 y ataque 4 |

| Endpoint | Roles permitidos | Regla de propiedad o datos ocultos | Código si se incumple |
| --- | --- | --- | --- |
| `GET /api/v1/historial` | `CONTRATANTE`, `TRABAJADOR` | Solo devuelve el historial del usuario autenticado | `401` sin token; `403` para `ADMIN` |

> **Pregunta 3 (ejemplo de formato, no lo copien).** Con el token de Luis pedimos el pedido 1, que es de Ana, y recibimos `404` (prueba `pedidoAjeno`). Respondemos `404` y no `403` porque un `403` confirmaría que el pedido existe. En cambio, un `ADMIN` que llama a `/pedidos` recibe `403`: sabemos quién es, pero su rol no alcanza.

## 10. Evidencias

Guárdenlas en `docs/evidence/` y refiéranlas desde el `README`:

- La columna `password_hash` en PostgreSQL, que muestre un hash y no texto plano.
- El payload decodificado de un token en jwt.io (sin mostrar el valor completo del token).
- Una respuesta `401`, una `403` y una `404` de su API.
- Los ataques 2 y 4 del bloque 8.
- El resultado de `mvn clean verify`.

## 11. Matriz OWASP

Completen la tabla con lo que implementaron (usen la numeración OWASP Top 10 2025 de la presentación):

| Riesgo | ¿Dónde podría ocurrir en OficioYa? | Control implementado | Prueba o evidencia |
| --- | --- | --- | --- |
| A01 · Control de acceso roto | | | |
| A02 · Configuración de seguridad incorrecta | | | |
| A04 · Fallas criptográficas | | | |
| A05 · Inyección | | | |
| A07 · Fallas de autenticación | | | |
| A10 · Manejo incorrecto de condiciones excepcionales | | | |

## 12. Matriz de acceso para el proyecto del curso

Elijan **al menos 6 endpoints** del dominio de su squad en el proyecto OficioYa y completen la tabla. Es el puente entre este laboratorio y el repositorio del proyecto:

| Endpoint | Roles permitidos | Regla de propiedad o datos ocultos | Código si se incumple |
| --- | --- | --- | --- |
| | | | |

Agreguen un párrafo breve: en la arquitectura del proyecto, ¿qué componente emite el JWT y cuáles lo validan? ¿Cómo comparten la clave sin ponerla en el repositorio?

## 13. Informe

Respondan en `docs/informe-seguridad.md`, con ejemplos de **su** implementación (máximo una página en total):

1. ¿Dónde aparecen en su proyecto codificar, cifrar y hashear? ¿Por qué la contraseña se hashea y no se cifra?
2. Decodificaron el payload de su token: ¿qué contiene, qué pasó cuando cambiaron el `role` a `ADMIN` y por qué el servidor lo detectó?
3. Den un ejemplo de `401`, `403` y `404` de su API. ¿Por qué una solicitud ajena responde `404` y no `403`?
4. ¿Por qué el contratante no envía su propio id al crear una solicitud?
5. ¿Por qué desactivaron CSRF? ¿Qué limitación tiene el cierre de sesión sin revocación de tokens?
6. ¿Qué pasaría si la clave del JWT se sube a GitHub, o si es corta? ¿Qué cambiarían para llevar esta API a producción?

## 14. Buenas prácticas de Git

- Ramas con los nombres indicados en cada bloque, creadas desde `develop`.
- Commits con el formato `tipo: descripción`, con `feat`, `fix`, `test`, `docs` o `chore`.
- Cada PR a `develop` lo revisa y aprueba un integrante distinto de su autor.
- Al final, un PR de `develop` a `main`.
- **Los cuatro integrantes deben tener commits y haber revisado al menos un PR.**

## 15. Evaluación

| Componente | Peso |
| --- | ---: |
| Registro y contraseñas (RS-01, RS-02) | 12 % |
| Login y JWT (RS-03, RS-04, RS-10) | 18 % |
| Configuración de seguridad y errores (RS-05, RS-09) | 15 % |
| Autorización: perfiles, administración y solicitudes (RS-06, RS-07, RS-08) | 25 % |
| Pruebas automatizadas (RS-11) | 15 % |
| Matriz OWASP, matriz de acceso e informe | 10 % |
| Proceso: Git, PR y README | 5 % |
| **Total** | **100 %** |

### Definition of Done

- [ ] Es posible registrarse e iniciar sesión, y las contraseñas están hasheadas.
- [ ] El JWT está firmado con HS256, vence y se valida en el servidor.
- [ ] Sin token, o con uno alterado o vencido, la API responde `401`.
- [ ] Ningún usuario ve ni modifica solicitudes ajenas.
- [ ] Un rol sin permiso recibe `403`; el perfil público no revela correo ni teléfono.
- [ ] Ningún secreto está en el repositorio ni en los logs.
- [ ] `mvn clean verify` pasa y la matriz de pruebas está cubierta.
- [ ] Las evidencias, las dos matrices y el informe están en `docs/`.

### Descuentos por errores críticos

Contraseñas en texto plano, clave del JWT en el repositorio, `ADMIN` aceptado en el registro público o rutas protegidas abiertas con `permitAll()` restan puntos aunque el resto funcione.

## 16. Referencias

- Presentación del curso: *Seguridad en API REST con Spring Security* (semana 10).
- [Spring Security · documentación oficial](https://docs.spring.io/spring-security/reference/)
- [Spring Security · almacenamiento de contraseñas](https://docs.spring.io/spring-security/reference/features/authentication/password-storage.html)
- [Spring Security · pruebas con `@WithMockUser`](https://docs.spring.io/spring-security/reference/servlet/test/method.html)
- [jjwt · documentación](https://github.com/jwtk/jjwt)
- [OWASP Top 10](https://owasp.org/Top10/)
- [OWASP · API Security](https://owasp.org/www-project-api-security/)
- [jwt.io](https://jwt.io) para inspeccionar tokens

---

## Anexo A · Diferencias entre Spring Boot 3 y 4

Los ejemplos funcionan en ambas versiones con estos cambios:

| Tema | Spring Boot 4.x | Spring Boot 3.5.x |
| --- | --- | --- |
| Starter web | `spring-boot-starter-webmvc` | `spring-boot-starter-web` |
| Dependencias de prueba | `spring-boot-starter-test`, `spring-boot-starter-webmvc-test`, `spring-boot-starter-security-test` y `spring-security-test` | `spring-boot-starter-test` y `spring-security-test` |
| Import de `@AutoConfigureMockMvc` | `org.springframework.boot.webmvc.test.autoconfigure.AutoConfigureMockMvc` | `org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc` |
| `ObjectMapper` (Jackson) | `tools.jackson.databind.ObjectMapper` | `com.fasterxml.jackson.databind.ObjectMapper` |
| Simular un bean en pruebas | `@MockitoBean` | `@MockitoBean` (o el antiguo `@MockBean`) |

> En Spring Boot 4, si falta `spring-boot-starter-security-test`, las pruebas con `@WithMockUser` responden `401` aunque el código esté bien.
