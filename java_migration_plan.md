# Java 21 Migration Plan — Shopizer Backend

## Current State

Confirmed from codebase:

| Item | Value |
|---|---|
| Java (Maven compiler) | 11 |
| Java (Dockerfile + GitHub Actions) | 17 ← already ahead of compiler |
| Spring Boot | 2.5.12 |
| JWT | `jjwt:0.8.0` (old monolithic artifact) |
| Swagger | `springfox-swagger2:2.9.2` (abandoned) |
| Drools | `7.32.0.Final` |
| Infinispan | `9.4.18.Final` |
| MapStruct | `1.3.0.Final` |

The Dockerfile and CI already run Java 17 JVM but Maven still compiles to Java 11 bytecode — this inconsistency is the first thing to fix.

---

## Migration Phases

```
Phase 1  →  Fix Java 11 → 17 compiler + replace broken libs
Phase 2a →  Spring Boot 2.5 → 2.7 (low risk step)
Phase 2b →  Spring Boot 2.7 → 3.x (javax → jakarta, biggest effort)
Phase 3  →  Java 17 → 21 (straightforward once on Boot 3.x)
```

---

## Phase 1 — Java 17 Compiler + Library Fixes

**Branch:** `feature/java17-upgrade`

### What changes in `pom.xml`

```xml
<java.version>17</java.version>
<org.mapstruct.version>1.5.5.Final</org.mapstruct.version>
<jwt.version>0.11.5</jwt.version>
```

Replace the single `jjwt:0.8.0` artifact with the split 0.11.5 artifacts:
```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>${jwt.version}</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>${jwt.version}</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>${jwt.version}</version>
    <scope>runtime</scope>
</dependency>
```

Replace springfox (abandoned, broken on Boot 2.6+) with springdoc:
```xml
<!-- remove springfox-swagger2 and springfox-swagger-ui -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-ui</artifactId>
    <version>1.7.0</version>
</dependency>
```

Add `--add-opens` JVM args to `sm-shop/pom.xml` Spring Boot plugin (needed for Drools + Infinispan on Java 17):
```xml
<jvmArguments>
    --add-opens java.base/java.lang=ALL-UNNAMED
    --add-opens java.base/java.util=ALL-UNNAMED
    --add-opens java.base/java.io=ALL-UNNAMED
</jvmArguments>
```

### JWT code changes (0.8.0 → 0.11.5 API diff)

```bash
grep -rn "Jwts\.\|SignatureAlgorithm" --include="*.java" .
```

```java
// before
Jwts.parser().setSigningKey(secret).parseClaimsJws(token)

// after
Jwts.parserBuilder()
    .setSigningKey(Keys.hmacShaKeyFor(secret.getBytes()))
    .build()
    .parseClaimsJws(token)
```

### Swagger config cleanup

```bash
grep -rn "Docket\|EnableSwagger2" --include="*.java" .
```

Remove all `Docket` beans and `@EnableSwagger2` — springdoc needs no replacement config.

Add to `application.properties`:
```properties
spring.mvc.pathmatch.matching-strategy=ant_path_matcher
```

Swagger UI moves from `/swagger-ui.html` → `/swagger-ui/index.html`

### Done when
```bash
mvn clean install -DskipTests
mvn test -pl sm-core,sm-shop
# POST /api/v1/private/login returns 200
# /swagger-ui/index.html loads
```

---

## Phase 2a — Spring Boot 2.5 → 2.7

**Branch:** `feature/springboot-27`

```xml
<version>2.7.18</version>
```

Two likely issues:
- Circular bean refs now fail by default → add `spring.main.allow-circular-references=true`
- H2 2.x SQL changes → add `MODE=LEGACY` to H2 JDBC URL if dev profile breaks

---

## Phase 2b — Spring Boot 2.7 → 3.2 (Jakarta)

**Branch:** `feature/springboot-3x`

This is the hardest phase. Spring Boot 3 requires Java 17+ (done), and migrates all `javax.*` to `jakarta.*`.

```xml
<version>3.2.12</version>
```

### Step 1 — Run OpenRewrite (handles ~80% automatically)
```bash
mvn org.openrewrite.maven:rewrite-maven-plugin:run \
  -Drewrite.recipeArtifactCoordinates=org.openrewrite.recipe:rewrite-spring:LATEST \
  -Drewrite.activeRecipes=org.openrewrite.java.spring.boot3.UpgradeSpringBoot_3_2
```

### Step 2 — Fix remaining javax imports manually
```bash
grep -rn "import javax\." --include="*.java" .
```

Key replacements: `javax.persistence` → `jakarta.persistence`, `javax.validation` → `jakarta.validation`, `javax.servlet` → `jakarta.servlet`

### Step 3 — Spring Security 6 (`WebSecurityConfigurerAdapter` removed)
```java
// before
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    protected void configure(HttpSecurity http) { ... }
}

// after
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) { ... }
}
```

### Step 4 — Upgrade Drools and Infinispan
```xml
<drools.version>8.44.0.Final</drools.version>
<infinispan.version>14.0.27.Final</infinispan.version>
```

Both have API changes — run full integration tests after each upgrade.

### Step 5 — Upgrade springdoc for Boot 3
```xml
<!-- remove springdoc-openapi-ui:1.7.0 -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.3.0</version>
</dependency>
```

---

## Phase 3 — Java 21

**Branch:** `feature/java21-upgrade`

```xml
<java.version>21</java.version>
```

Update Dockerfile:
```dockerfile
FROM eclipse-temurin:21-jre-alpine
CMD ["java", "-XX:+UseZGC", "-XX:MaxRAMPercentage=75.0", "-jar", "/opt/app/shopizer.jar"]
```

Update GitHub Actions:
```yaml
java-version: '21'
```

Optional — enable virtual threads (zero code changes, better throughput):
```properties
spring.threads.virtual.enabled=true
```

---

## Risk Summary

| Phase | Biggest Risk |
|---|---|
| Phase 1 | JWT API change breaks auth; springfox removal breaks startup |
| Phase 2a | Circular bean refs, H2 SQL changes |
| Phase 2b | Drools 8 API rewrite, Hibernate 6 query changes, Spring Security 6 |
| Phase 3 | Minimal — straightforward once on Boot 3.x |

## Rollback

Always tag Docker images with the git SHA alongside `latest`:
```bash
docker build \
  -t ghcr.io/sahilkapoor8989/shopizer:${{ github.sha }} \
  -t ghcr.io/sahilkapoor8989/shopizer:latest \
  sm-shop/
```

To rollback: redeploy the previous SHA-tagged image.
