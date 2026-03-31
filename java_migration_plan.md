# Java Upgrade Migration Plan — Shopizer

## Current State

| Item | Current Value |
|---|---|
| Java | 11 (`pom.xml` `<java.version>11</java.version>`) |
| Spring Boot | 2.5.12 |
| Dockerfile base | `eclipse-temurin:17-jre-alpine` |
| GitHub Actions | `java-version: '17'` |
| JWT | `io.jsonwebtoken:jjwt:0.8.0` |
| Swagger | `io.springfox:springfox-swagger2:2.9.2` |
| Drools | `7.32.0.Final` |
| Infinispan | `9.4.18.Final` |
| MapStruct | `1.3.0.Final` |

> **Key Insight:** Dockerfile and GitHub Actions already target Java 17, but Maven compiler still targets Java 11 bytecode. This inconsistency is fixed in Phase 1.

---

## Upgrade Strategy

```
Phase 1:  Java 11 → Java 17          (fix the latent inconsistency)
Phase 2a: Spring Boot 2.5.12 → 2.7.x (safe incremental step)
Phase 2b: Spring Boot 2.7 → 3.x      (Jakarta namespace migration)
Phase 3:  Java 17 → Java 21          (latest LTS, ZGC, virtual threads)
```

---

## Phase 1 — Java 11 → Java 17

### pom.xml Changes

```xml
<!-- java version -->
<java.version>17</java.version>
<maven.compiler.source>17</maven.compiler.source>
<maven.compiler.target>17</maven.compiler.target>

<!-- upgrade MapStruct -->
<org.mapstruct.version>1.5.5.Final</org.mapstruct.version>

<!-- upgrade JWT -->
<jwt.version>0.11.5</jwt.version>
```

**Replace single `jjwt` with split artifacts in `dependencyManagement`:**
```xml
<!-- REMOVE -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt</artifactId>
    <version>${jwt.version}</version>
</dependency>

<!-- ADD -->
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

**Replace springfox with springdoc:**
```xml
<!-- REMOVE -->
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>${swagger.version}</version>
</dependency>
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>${swagger.version}</version>
</dependency>

<!-- ADD -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-ui</artifactId>
    <version>1.7.0</version>
</dependency>
```

**Add JVM flags to `sm-shop/pom.xml` Spring Boot plugin:**
```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <jvmArguments>
            --add-opens java.base/java.lang=ALL-UNNAMED
            --add-opens java.base/java.util=ALL-UNNAMED
            --add-opens java.base/java.io=ALL-UNNAMED
        </jvmArguments>
    </configuration>
</plugin>
```

### JWT Code Migration (0.8.0 → 0.11.5)

```bash
grep -rn "Jwts\.\|SignatureAlgorithm\|Claims" --include="*.java" .
```

| Old (0.8.0) | New (0.11.5) |
|---|---|
| `Jwts.parser()` | `Jwts.parserBuilder()` |
| `.setSigningKey(String)` | `.setSigningKey(Keys.hmacShaKeyFor(bytes))` |

```java
// OLD
Jwts.parser().setSigningKey(secret).parseClaimsJws(token)

// NEW
Jwts.parserBuilder()
    .setSigningKey(Keys.hmacShaKeyFor(secret.getBytes()))
    .build()
    .parseClaimsJws(token)
```

### Springfox → Springdoc Migration

```bash
grep -rn "Docket\|EnableSwagger2" --include="*.java" .
```

- Remove all `@EnableSwagger2` annotations and `Docket` beans
- Springdoc auto-configures — no replacement bean needed
- Add to `application.properties`:
```properties
spring.mvc.pathmatch.matching-strategy=ant_path_matcher
```
> Swagger UI URL changes: `/swagger-ui.html` → `/swagger-ui/index.html`

### Validation
```bash
mvn clean install -DskipTests
mvn test -pl sm-core,sm-shop
mvn spring-boot:run -pl sm-shop
# Verify: http://localhost:8080/swagger-ui/index.html
# Verify: POST /api/v1/private/login
```

### Risks

| Risk | Severity | Mitigation |
|---|---|---|
| jjwt API breaking change | MEDIUM | Grep all JWT usages; test all auth flows |
| Springfox config breaks startup | MEDIUM | Remove all `Docket` + `@EnableSwagger2` |
| Drools/Infinispan reflection warnings | LOW | Add `--add-opens` flags |

---

## Phase 2a — Spring Boot 2.5.12 → 2.7.18

### pom.xml Change
```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.7.18</version>
</parent>
```

### Breaking Changes

**Circular bean dependencies** — add if startup fails:
```properties
spring.main.allow-circular-references=true
```

**H2 2.x SQL compatibility** — add if H2 dev profile fails:
```properties
spring.datasource.url=jdbc:h2:file:./SALESMANAGER;MODE=LEGACY;DB_CLOSE_ON_EXIT=FALSE
```

### Risks

| Risk | Severity | Mitigation |
|---|---|---|
| Circular bean dependency failures | MEDIUM | Add `allow-circular-references=true` |
| H2 2.x SQL syntax changes | LOW | Add `MODE=LEGACY` to H2 URL |

---

## Phase 2b — Spring Boot 2.7 → 3.2.12 (Jakarta Migration)

Highest-risk phase. Requires `javax.*` → `jakarta.*` across all source files.

### pom.xml Change
```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.12</version>
</parent>
```

### Run OpenRewrite (automates ~80%)
```bash
mvn org.openrewrite.maven:rewrite-maven-plugin:run \
  -Drewrite.recipeArtifactCoordinates=org.openrewrite.recipe:rewrite-spring:LATEST \
  -Drewrite.activeRecipes=org.openrewrite.java.spring.boot3.UpgradeSpringBoot_3_2
```

### Manual Jakarta Namespace Migration
```bash
grep -rn "import javax\." --include="*.java" .
```

| Old | New |
|---|---|
| `javax.persistence.*` | `jakarta.persistence.*` |
| `javax.validation.*` | `jakarta.validation.*` |
| `javax.servlet.*` | `jakarta.servlet.*` |
| `javax.inject.*` | `jakarta.inject.*` |
| `javax.annotation.*` | `jakarta.annotation.*` |

### Spring Security 6 Migration

`WebSecurityConfigurerAdapter` is removed. Find usages:
```bash
grep -rn "WebSecurityConfigurerAdapter" --include="*.java" .
```

```java
// OLD
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception { ... }
}

// NEW
@Configuration
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception { ... }
}
```

### Drools Upgrade (7.x → 8.x)
```xml
<drools.version>8.44.0.Final</drools.version>
```
```bash
grep -rn "KieSession\|KieContainer\|KieServices" --include="*.java" .
```

### Infinispan Upgrade (9.4 → 14.x)
```xml
<infinispan.version>14.0.27.Final</infinispan.version>
<infinispan.tree.version>14.0.27.Final</infinispan.tree.version>
```

### Springdoc Upgrade for Spring Boot 3
```xml
<!-- REMOVE -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-ui</artifactId>
    <version>1.7.0</version>
</dependency>

<!-- ADD -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.3.0</version>
</dependency>
```

### Risks

| Risk | Severity | Mitigation |
|---|---|---|
| Drools 7 → 8 API changes | HIGH | Test all pricing/discount rules |
| Hibernate 6 query behavior changes | HIGH | Run full integration test suite |
| Spring Security config rewrite | MEDIUM | Use OpenRewrite; test all auth flows |
| Jakarta namespace misses | MEDIUM | Grep after OpenRewrite to catch remaining |

---

## Phase 3 — Java 17 → Java 21

### pom.xml Change
```xml
<java.version>21</java.version>
<maven.compiler.source>21</maven.compiler.source>
<maven.compiler.target>21</maven.compiler.target>
```

### Dockerfile Update
```dockerfile
FROM eclipse-temurin:21-jre-alpine
RUN mkdir /opt/app /files
COPY target/shopizer.jar /opt/app
COPY SALESMANAGER.h2.db /
COPY ./files /files
CMD ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-XX:+UseZGC", \
  "--add-opens", "java.base/java.lang=ALL-UNNAMED", \
  "-jar", "/opt/app/shopizer.jar"]
```

### GitHub Actions Update
```yaml
- uses: actions/setup-java@v4
  with:
    java-version: '21'
    distribution: temurin
```

### Optional: Virtual Threads (Project Loom)
```properties
spring.threads.virtual.enabled=true
```

### Risks

| Risk | Severity | Mitigation |
|---|---|---|
| Virtual threads + blocking code | LOW | Monitor thread dumps |
| ZGC memory overhead | LOW | Fallback to G1GC if needed |

---

## Branching Strategy

```
main
├── feature/java17-upgrade     ← Phase 1
├── feature/springboot-27      ← Phase 2a
├── feature/springboot-3x      ← Phase 2b
└── feature/java21-upgrade     ← Phase 3
```

Each branch merges to `main` only after all CI tests pass and Docker image is smoke-tested.

---

## Rollback Strategy

```bash
# Git rollback
git revert -m 1 <merge-commit-sha>
git push origin main

# Docker image rollback — always tag with SHA
docker build \
  -t ghcr.io/sahilkapoor8989/shopizer:${{ github.sha }} \
  -t ghcr.io/sahilkapoor8989/shopizer:latest \
  sm-shop/
```

Never overwrite a SHA-tagged image. Keep at least 3 previous tags.

---

## Summary

| Phase | Risk | Effort | Priority |
|---|---|---|---|
| Phase 1: Java 17 compiler fix | LOW | Small | **Do immediately** |
| Phase 2a: Spring Boot 2.7 | LOW | Small | After Phase 1 stabilizes |
| Phase 2b: Spring Boot 3.x | HIGH | Large | Budget most time — Drools + Jakarta are hardest |
| Phase 3: Java 21 | LOW | Small | Straightforward once on Boot 3.x |
