# Spécifications du back-end Java/Spring Boot

Généré à partir de l'audit du front-end Angular et des spécifications techniques du projet.

---

## Table des matières

1. [Stack technique et architecture](#1-stack-technique-et-architecture)
2. [Configuration et environnement](#2-configuration-et-environnement)
3. [Schéma de base de données](#3-schéma-de-base-de-données)
4. [Sécurité et JWT](#4-sécurité-et-jwt)
5. [Cartographie complète des endpoints](#5-cartographie-complète-des-endpoints)
6. [Gestion des images](#6-gestion-des-images)
7. [Documentation Swagger / OpenAPI](#7-documentation-swagger--openapi)
8. [Points d'attention issus de l'audit front-end](#8-points-dattention-issus-de-laudit-front-end)

---

## 1. Stack technique et architecture

### Versions

| Technologie | Version minimale |
|---|---|
| Java | 11 ou 17 |
| Spring Boot | 3.x (compatible Java 17) ou 2.7.x (Java 11) |
| Spring Security | inclus dans Spring Boot |
| MySQL | 8.x |
| Maven ou Gradle | au choix |

### Architecture en couches

```
com.openclassrooms.rentals/
├── controller/          ← Couche HTTP : reçoit les requêtes, renvoie les réponses
├── service/             ← Couche métier : logique applicative
├── repository/          ← Couche données : interfaces JPA (Spring Data)
├── model/ (ou entity/)  ← Entités JPA mappées sur les tables MySQL
├── dto/                 ← Objets de transfert (request/response)
├── security/            ← Configuration Spring Security, filtres JWT
└── exception/           ← Gestion centralisée des erreurs (optionnel mais recommandé)
```

**Règle** : les contrôleurs ne contiennent aucune logique métier. Les repositories ne sont jamais appelés directement depuis les contrôleurs.

---

## 2. Configuration et environnement

### `application.properties` (ou `application.yml`)

Les identifiants de base de données et le secret JWT **ne doivent pas apparaître dans le code versionné**. Utiliser des variables d'environnement ou un fichier `.env` exclu du dépôt Git.

```properties
# Serveur
server.port=3001

# Base de données MySQL
spring.datasource.url=${DB_URL:jdbc:mysql://localhost:3306/rentals_db}
spring.datasource.username=${DB_USERNAME}
spring.datasource.password=${DB_PASSWORD}
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

# JPA / Hibernate
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=false
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQL8Dialect

# JWT
app.jwt.secret=${JWT_SECRET}
app.jwt.expiration=86400000  # 24h en millisecondes

# Upload d'images
app.upload.dir=${UPLOAD_DIR:uploads/}
app.base-url=${BASE_URL:http://localhost:3001}
```

### Proxy Angular ↔ back-end

Le front-end Angular envoie ses requêtes vers des chemins **relatifs** (`/api/auth`, `/api/rentals`, etc.) sans utiliser la `baseUrl` définie dans `environment.ts`. En développement, il faut configurer le proxy Angular (`proxy.conf.json`) pour rediriger `/api/` vers `http://localhost:3001/api/`, ou servir les deux depuis le même origin.

Le back-end doit exposer toutes ses routes sous le préfixe `/api/`.

```properties
# Préfixe global des routes API
server.servlet.context-path=/
```

Toutes les routes dans les contrôleurs doivent commencer par `/api/...`.

---

## 3. Schéma de base de données

### Table `users`

```sql
CREATE TABLE users (
    id         BIGINT AUTO_INCREMENT PRIMARY KEY,
    name       VARCHAR(255) NOT NULL,
    email      VARCHAR(255) NOT NULL UNIQUE,
    password   VARCHAR(255) NOT NULL,  -- mot de passe BCrypt hashé
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

### Table `rentals`

```sql
CREATE TABLE rentals (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    name        VARCHAR(255) NOT NULL,
    surface     DECIMAL(10, 2) NOT NULL,
    price       DECIMAL(10, 2) NOT NULL,
    picture     VARCHAR(500),           -- URL complète de l'image stockée sur le serveur
    description TEXT NOT NULL,
    owner_id    BIGINT NOT NULL,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    CONSTRAINT fk_rental_owner FOREIGN KEY (owner_id) REFERENCES users(id)
);
```

### Table `messages`

```sql
CREATE TABLE messages (
    id        BIGINT AUTO_INCREMENT PRIMARY KEY,
    rental_id BIGINT NOT NULL,
    user_id   BIGINT NOT NULL,
    message   TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    CONSTRAINT fk_message_rental FOREIGN KEY (rental_id) REFERENCES rentals(id),
    CONSTRAINT fk_message_user   FOREIGN KEY (user_id)   REFERENCES users(id)
);
```

### Entités JPA correspondantes

```java
// User.java
@Entity
@Table(name = "users")
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    @Column(unique = true)
    private String email;
    private String password;          // stocker le hash BCrypt
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    // getters/setters ou @Data Lombok
}

// Rental.java
@Entity
@Table(name = "rentals")
public class Rental {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private Double surface;
    private Double price;
    private String picture;           // URL de l'image
    @Column(columnDefinition = "TEXT")
    private String description;
    @Column(name = "owner_id")
    private Long ownerId;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}

// Message.java
@Entity
@Table(name = "messages")
public class Message {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Column(name = "rental_id")
    private Long rentalId;
    @Column(name = "user_id")
    private Long userId;
    @Column(columnDefinition = "TEXT")
    private String message;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```

---

## 4. Sécurité et JWT

### Vue d'ensemble

- Spring Security filtre toutes les requêtes entrantes
- Un filtre `JwtAuthenticationFilter` lit le header `Authorization: Bearer {token}`
- Les routes publiques (login, register, swagger) sont explicitement exclues
- Le mot de passe est hashé avec **BCrypt** — jamais stocké en clair

### Dépendances Maven

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.11.5</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.11.5</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.11.5</version>
    <scope>runtime</scope>
</dependency>
```

### Configuration Spring Security

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http, JwtAuthenticationFilter jwtFilter) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/login", "/api/auth/register").permitAll()
                .requestMatchers("/v3/api-docs/**", "/swagger-ui/**", "/swagger-ui.html").permitAll()
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### Filtre JWT

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        String header = request.getHeader("Authorization");
        if (header != null && header.startsWith("Bearer ")) {
            String token = header.substring(7);
            // valider le token, extraire l'email, charger UserDetails, peupler le SecurityContext
        }
        chain.doFilter(request, response);
    }
}
```

### Service JWT

```java
@Service
public class JwtService {
    @Value("${app.jwt.secret}")
    private String secret;

    @Value("${app.jwt.expiration}")
    private long expirationMs;

    public String generateToken(String email) { ... }
    public String extractEmail(String token) { ... }
    public boolean isTokenValid(String token, UserDetails userDetails) { ... }
}
```

### Flux d'authentification

```
[Front] POST /api/auth/login { email, password }
    → AuthController → AuthService
    → Vérifier email dans DB (UserRepository)
    → Vérifier password avec BCrypt.matches()
    → Si OK : générer token JWT avec email comme subject
    → Retourner { "token": "eyJhbGciOiJIUzI1NiJ9..." }
    → [Front] stocke token dans localStorage['token']

[Requêtes suivantes]
    → Header: Authorization: Bearer {token}
    → JwtAuthenticationFilter valide le token
    → Peuple SecurityContext avec l'utilisateur courant
    → Le contrôleur peut appeler SecurityContextHolder pour obtenir l'utilisateur connecté
```

---

## 5. Cartographie complète des endpoints

### Règles communes

- Toutes les réponses sont en `application/json`
- Le champ `Authorization: Bearer {token}` est requis sur toutes les routes sauf login, register et Swagger
- Les timestamps (`created_at`, `updated_at`) sont sérialisés en **ISO 8601** : `"2024-01-15T10:30:00.000+00:00"`
- En cas d'absence ou de token invalide sur une route protégée → **HTTP 401** (géré par Spring Security)

---

### 5.1 `POST /api/auth/register`

**Accès** : public

**Request body** (JSON) :
```json
{
  "email": "user@example.com",
  "name": "John Doe",
  "password": "secret123"
}
```

| Champ | Type | Obligatoire | Contraintes |
|---|---|---|---|
| `email` | string | oui | format email, unique en base |
| `name` | string | oui | non vide |
| `password` | string | oui | non vide |

**Réponse 200 OK** :
```json
{ "token": "eyJhbGciOiJIUzI1NiJ9..." }
```

**Réponse 400 Bad Request** : si l'email est déjà utilisé ou si un champ est manquant.

**Comportement** :
1. Hasher le mot de passe avec BCrypt
2. Persister le nouvel utilisateur
3. Générer et retourner un token JWT

---

### 5.2 `POST /api/auth/login`

**Accès** : public

**Request body** (JSON) :
```json
{
  "email": "user@example.com",
  "password": "secret123"
}
```

**Réponse 200 OK** :
```json
{ "token": "eyJhbGciOiJIUzI1NiJ9..." }
```

**Réponse 401 Unauthorized** : email ou mot de passe incorrect.

---

### 5.3 `GET /api/auth/me`

**Accès** : authentifié

**Headers** : `Authorization: Bearer {token}`

**Réponse 200 OK** :
```json
{
  "id": 1,
  "name": "John Doe",
  "email": "user@example.com",
  "created_at": "2024-01-15T10:30:00.000+00:00",
  "updated_at": "2024-01-20T14:00:00.000+00:00"
}
```

**Réponse 401** : token absent ou invalide.

**Comportement** : extraire l'email depuis le token JWT, charger l'utilisateur depuis la base, retourner ses informations.

> Appelé au démarrage de l'app Angular (`app.component.ts`) pour vérifier si le token stocké est encore valide. Si cette route répond 401, le front efface le token et déconnecte l'utilisateur.

---

### 5.4 `GET /api/rentals`

**Accès** : authentifié

**Réponse 200 OK** :
```json
{
  "rentals": [
    {
      "id": 1,
      "name": "Appartement Paris 11",
      "surface": 42.0,
      "price": 150.0,
      "picture": "http://localhost:3001/uploads/abc123.jpg",
      "description": "Bel appartement lumineux...",
      "owner_id": 3,
      "created_at": "2024-01-15T10:30:00.000+00:00",
      "updated_at": "2024-01-20T14:00:00.000+00:00"
    }
  ]
}
```

> **Important** : la réponse est un **objet avec une propriété `rentals`**, pas un tableau nu. Le front itère sur `response.rentals` (`list.component.html:10`).

**Réponse 401** : token absent ou invalide.

---

### 5.5 `GET /api/rentals/{id}`

**Accès** : authentifié

**Path variable** : `id` (Long)

**Réponse 200 OK** : objet `Rental` unique (même structure qu'un élément du tableau ci-dessus)

**Réponse 401** : token absent ou invalide.

**Réponse 404** : rental inexistant.

---

### 5.6 `POST /api/rentals`

**Accès** : authentifié

**Content-Type** : `multipart/form-data`

**Form fields** :

| Champ | Type Spring | Obligatoire | Notes |
|---|---|---|---|
| `name` | `@RequestParam String` | oui | |
| `surface` | `@RequestParam String` → convertir en Double | oui | FormData sérialise les nombres en string |
| `price` | `@RequestParam String` → convertir en Double | oui | idem |
| `picture` | `@RequestParam MultipartFile` | oui | fichier image |
| `description` | `@RequestParam String` | oui | |

> **Attention** : `surface` et `price` arrivent comme des chaînes dans le multipart. Spring peut les convertir automatiquement si déclarés en `Double` ou `Integer` avec `@RequestParam`.

**Comportement** :
1. Sauvegarder le fichier image sur le serveur (répertoire configuré)
2. Construire l'URL publique complète : `{app.base-url}/uploads/{filename}`
3. Récupérer l'`owner_id` depuis le token JWT (l'utilisateur courant)
4. Persister le rental avec l'URL de l'image

**Réponse 200 OK** :
```json
{ "message": "Rental created !" }
```

**Réponse 401** : token absent ou invalide.

---

### 5.7 `PUT /api/rentals/{id}`

**Accès** : authentifié

**Path variable** : `id` (Long)

**Content-Type** : `multipart/form-data`

**Form fields** :

| Champ | Type Spring | Obligatoire | Notes |
|---|---|---|---|
| `name` | `@RequestParam String` | oui | |
| `surface` | `@RequestParam String/Double` | oui | |
| `price` | `@RequestParam String/Double` | oui | |
| `description` | `@RequestParam String` | oui | |

> **`picture` n'est pas envoyé** lors d'une mise à jour. Le back-end doit conserver l'URL d'image existante en base. Ne pas écraser ni supprimer l'image lors d'un PUT.

**Comportement** :
1. Charger le rental depuis la base
2. Vérifier que `owner_id == id de l'utilisateur connecté` (le front le vérifie aussi dans `form.component.ts:67`, mais la vérification serveur est obligatoire)
3. Mettre à jour les champs texte uniquement
4. Mettre à jour `updated_at`

**Réponse 200 OK** :
```json
{ "message": "Rental updated !" }
```

**Réponse 401** : token absent ou invalide.

**Réponse 403** : l'utilisateur connecté n'est pas le propriétaire du rental.

---

### 5.8 `POST /api/messages`

**Accès** : authentifié

**Request body** (JSON) :
```json
{
  "rental_id": 1,
  "user_id": 3,
  "message": "Bonjour, est-ce disponible en août ?"
}
```

| Champ | Type | Obligatoire | Notes |
|---|---|---|---|
| `rental_id` | number | oui | doit référencer un rental existant |
| `user_id` | number | oui | doit référencer un user existant |
| `message` | string | oui | contenu du message |

> **Note** : le front envoie `user_id` explicitement depuis `sessionService.user?.id`. Il est recommandé de le vérifier côté serveur (l'utilisateur connecté correspond bien à `user_id`), mais cela peut être un simple enregistrement selon les exigences métier.

**Réponse 200 OK** :
```json
{ "message": "Message send with success" }
```

> Le champ `message` de la réponse est affiché tel quel dans le `MatSnackBar` du front (`detail.component.ts:48`).

**Réponse 401** : token absent ou invalide.

---

### 5.9 `GET /api/user/{id}`

**Accès** : authentifié

**Path variable** : `id` (Long)

**Réponse 200 OK** :
```json
{
  "id": 3,
  "name": "Jane Smith",
  "email": "jane@example.com",
  "created_at": "2024-01-10T08:00:00.000+00:00",
  "updated_at": "2024-01-10T08:00:00.000+00:00"
}
```

> Seul `user.name` est affiché dans le template (`owner-info.component.html`), mais le front attend l'objet `User` complet. Retourner tous les champs.

**Réponse 401** : token absent ou invalide.

**Réponse 404** : utilisateur inexistant.

> **Attention aux performances** : ce endpoint est appelé autant de fois qu'il y a de rentals affichés dans la liste. Chaque carte `<app-owner-info>` déclenche un appel GET. Envisager un cache HTTP (`Cache-Control: max-age=60`) pour réduire la charge.

---

## 6. Gestion des images

### Flux complet

```
[Front] POST /api/rentals (multipart/form-data, champ "picture" = fichier binaire)
    ↓
[RentalController] reçoit MultipartFile picture
    ↓
[RentalService] appelle ImageService.save(picture)
    ↓
[ImageService]
  1. Générer un nom de fichier unique : UUID + extension originale
     ex: "a1b2c3d4-e5f6-7890-abcd-ef1234567890.jpg"
  2. Sauvegarder dans le répertoire configuré (app.upload.dir)
  3. Retourner l'URL publique : "{app.base-url}/uploads/{filename}"
     ex: "http://localhost:3001/uploads/a1b2c3d4.jpg"
    ↓
[RentalService] persister rental.picture = URL retournée
    ↓
[Front] reçoit Rental avec picture = URL complète
[Front] <img [src]="rental.picture"> → chargement direct de l'image
```

### Configuration du répertoire de stockage

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Value("${app.upload.dir}")
    private String uploadDir;

    @Value("${app.base-url}")
    private String baseUrl;

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        // Exposer le répertoire d'uploads comme ressource statique
        registry.addResourceHandler("/uploads/**")
                .addResourceLocations("file:" + uploadDir);
    }
}
```

### Service d'upload

```java
@Service
public class ImageService {

    @Value("${app.upload.dir}")
    private String uploadDir;

    @Value("${app.base-url}")
    private String baseUrl;

    public String save(MultipartFile file) throws IOException {
        String extension = StringUtils.getFilenameExtension(file.getOriginalFilename());
        String filename = UUID.randomUUID() + "." + extension;
        Path targetPath = Paths.get(uploadDir).resolve(filename);
        Files.createDirectories(targetPath.getParent());
        Files.copy(file.getInputStream(), targetPath, StandardCopyOption.REPLACE_EXISTING);
        return baseUrl + "/uploads/" + filename;
    }
}
```

### Comportement au PUT (mise à jour)

Le front **n'envoie jamais** le champ `picture` lors d'un PUT (`form.component.ts:48`). Le contrôleur de mise à jour ne doit **pas accepter** de `MultipartFile` optionnel, et doit simplement ignorer le champ image — conserver la valeur existante en base.

---

## 7. Documentation Swagger / OpenAPI

### Dépendance

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.1.0</version>
</dependency>
```

### Configuration

```java
@Configuration
public class SwaggerConfig {

    @Bean
    public OpenAPI openAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("Rental API")
                .version("1.0")
                .description("API back-end pour l'application de location"))
            .addSecurityItem(new SecurityRequirement().addList("Bearer Authentication"))
            .components(new Components()
                .addSecuritySchemes("Bearer Authentication",
                    new SecurityScheme()
                        .type(SecurityScheme.Type.HTTP)
                        .scheme("bearer")
                        .bearerFormat("JWT")));
    }
}
```

### Routes Swagger (à exclure de l'authentification)

```java
// Dans SecurityConfig.authorizeHttpRequests() :
.requestMatchers(
    "/v3/api-docs/**",
    "/swagger-ui/**",
    "/swagger-ui.html"
).permitAll()
```

La documentation Swagger est accessible sans authentification. Pour **tester les routes protégées** via Swagger UI, utiliser le bouton "Authorize" en haut de la page et saisir le token JWT obtenu via login.

### URL Swagger en développement

```
http://localhost:3001/swagger-ui/index.html
http://localhost:3001/v3/api-docs
```

---

## 8. Points d'attention issus de l'audit front-end

### 8.1 Nommage des champs en snake_case

Le front-end attend des réponses JSON avec des champs en **snake_case** (`owner_id`, `created_at`, `updated_at`, `rental_id`, `user_id`). Configurer Jackson pour la sérialisation :

```properties
spring.jackson.property-naming-strategy=SNAKE_CASE
```

Ou via annotation sur les DTO :

```java
@JsonNaming(PropertyNamingStrategies.SnakeCaseStrategy.class)
public class RentalDto { ... }
```

Sans cette configuration, Spring Boot sérialisera `ownerId` → `"ownerId"` au lieu de `"owner_id"`, ce qui cassera l'affichage côté front.

### 8.2 Format des timestamps

Le front utilise le pipe Angular `date` directement sur `rental.created_at` et `user.created_at`. Retourner des chaînes **ISO 8601** :

```
"2024-01-15T10:30:00.000+00:00"
```

Configurer Jackson :

```properties
spring.jackson.serialization.write-dates-as-timestamps=false
spring.jackson.time-zone=UTC
```

### 8.3 FormData — `surface` et `price` arrivent en string

Dans le multipart, toutes les valeurs sont des chaînes. Spring Boot convertit automatiquement `@RequestParam Double surface` depuis une chaîne `"42.0"`. Tester ce cas explicitement pour s'assurer que la conversion fonctionne (notamment avec des valeurs entières comme `"42"` sans décimale).

### 8.4 `picture` non envoyée au PUT

Le PUT ne contient jamais le champ `picture`. Ne pas déclarer `@RequestParam(required = false) MultipartFile picture` — juste ne pas inclure de paramètre picture dans la méthode PUT. La logique de mise à jour doit charger le rental existant et conserver son URL d'image.

### 8.5 Réponse de `GET /api/rentals` — objet wrapper obligatoire

```json
// ✓ Correct — ce que le front attend
{ "rentals": [ {...}, {...} ] }

// ✗ Incorrect — tableau nu, le front ne peut pas itérer
[ {...}, {...} ]
```

Le template Angular itère sur `(rentals$ | async)?.rentals` (`list.component.html:10`).

### 8.6 `GET /api/user/{id}` — chemin singulier

Le service Angular utilise `api/user` au singulier (`user.service.ts:11`), contrairement aux autres services qui utilisent le pluriel. Le contrôleur Spring doit exposer `/api/user/{id}` (singulier), pas `/api/users/{id}`.

### 8.7 Aucune gestion du 401 côté front (sauf au démarrage)

Le front ne gère pas les erreurs 401 survenant pendant la navigation (token expiré en cours de session). Spring Security retournera 401, la requête Angular échouera, et rien ne se passera côté UI. C'est un comportement connu et accepté du front actuel — ne pas tenter de corriger côté back-end.

### 8.8 Messages de succès contrôlés par le back-end

Les messages affichés dans les `MatSnackBar` proviennent directement des champs `message` des réponses JSON :
- `RentalResponse.message` → affiché après create/update (`form.component.ts:82`)
- `MessageResponse.message` → affiché après envoi d'un message (`detail.component.ts:48`)

Exemples de valeurs attendues (conformes à Mockoon) :
```json
{ "message": "Rental created !" }
{ "message": "Rental updated !" }
{ "message": "Message send with success" }
```

### 8.9 Double navigation après login (bug front non bloquant)

`login.component.ts` appelle `router.navigate(['/rentals'])` deux fois : une immédiatement après le login (ligne 37), et une dans le callback de `/me` (ligne 35). Cela peut provoquer une navigation avant que `sessionService.user` soit peuplé. Le back-end n'a rien à faire — c'est un bug front documenté ici pour information.

### 8.10 `user_id` potentiellement absent dans `POST /api/messages`

Le front envoie `user_id: this.sessionService.user?.id` — si `user` est `undefined`, `user_id` sera `undefined` dans le body JSON. Le back-end doit valider la présence de ce champ et retourner 400 si absent.

---

## Récapitulatif des 9 endpoints

| # | Méthode | URL | Auth | Content-Type | Réponses |
|---|---|---|---|---|---|
| 1 | `POST` | `/api/auth/register` | non | JSON | 200, 400 |
| 2 | `POST` | `/api/auth/login` | non | JSON | 200, 401 |
| 3 | `GET` | `/api/auth/me` | Bearer | — | 200, 401 |
| 4 | `GET` | `/api/rentals` | Bearer | — | 200, 401 |
| 5 | `GET` | `/api/rentals/{id}` | Bearer | — | 200, 401, 404 |
| 6 | `POST` | `/api/rentals` | Bearer | multipart/form-data | 200, 401 |
| 7 | `PUT` | `/api/rentals/{id}` | Bearer | multipart/form-data | 200, 401, 403 |
| 8 | `POST` | `/api/messages` | Bearer | JSON | 200, 401 |
| 9 | `GET` | `/api/user/{id}` | Bearer | — | 200, 401, 404 |
