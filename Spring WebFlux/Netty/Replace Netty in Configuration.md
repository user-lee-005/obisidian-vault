# 1. What can you use instead of Netty?

Spring WebFlux supports:

### 1. Apache Tomcat

- Uses Servlet 3.1 async
- Not truly event-loop based like Netty
- Slight overhead

### 2. Jetty

- Better async handling than Tomcat
- Still servlet-based

### 3. Undertow

- Lightweight
- Good performance
- Still not as clean as Netty for reactive

---

# 2. How to replace Netty (actual setup)

## Step 1 — Remove Netty

If you’re using:

spring-boot-starter-webflux

Exclude Netty:

<dependency>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-webflux</artifactId>  
    <exclusions>  
        <exclusion>  
            <groupId>org.springframework.boot</groupId>  
            <artifactId>spring-boot-starter-reactor-netty</artifactId>  
        </exclusion>  
    </exclusions>  
</dependency>

---

## Step 2 — Add the server you want

### 👉 For Tomcat

<dependency>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-tomcat</artifactId>  
</dependency>

---

### 👉 For Jetty

<dependency>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-jetty</artifactId>  
</dependency>

---

### 👉 For Undertow

<dependency>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-undertow</artifactId>  
</dependency>

---

## Step 3 — Run normally

Spring Boot auto-detects the server from the classpath.

No extra config needed.

---

# 3. What people don’t tell you (important)

Switching away from Netty usually means:

### ❌ You lose:

- True event-loop efficiency
- Best performance under high concurrency
- Clean non-blocking pipeline

### ✅ You gain:

- Compatibility with servlet ecosystem
- Easier integration with legacy filters/libs

---

# 4. When switching actually makes sense

Do it ONLY if:

### ✅ You need servlet-based features

- Filters relying on Servlet API
- Legacy libs that don’t support reactive

### ✅ Your team already uses Tomcat everywhere

- Operational consistency matters

### ❌ Don’t switch if:

- You chose WebFlux for scalability
- You’re building streaming / real-time systems

---

# 5. Brutal recommendation

- If you’re serious about reactive → **stay with Netty**
- If you want servlet features → just use **Spring MVC instead of WebFlux**

👉 Mixing WebFlux + Tomcat is often a confused architecture.