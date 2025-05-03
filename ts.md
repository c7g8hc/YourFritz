To store both the **Snowflake token** and its **expiry timestamp** in Redis, you can wrap them in a custom Java object, serialize it, and store it in Redis as a single entry. Here's how you can do this professionally in a Spring Boot application using Java 17.

---

## ✅ **Step-by-Step Implementation**

### 1. **Define Token Cache Object**

```java
import java.io.Serializable;
import java.time.Instant;

public class SnowflakeTokenCache implements Serializable {
    private final String token;
    private final Instant expiryTime;

    public SnowflakeTokenCache(String token, Instant expiryTime) {
        this.token = token;
        this.expiryTime = expiryTime;
    }

    public String getToken() {
        return token;
    }

    public Instant getExpiryTime() {
        return expiryTime;
    }

    public boolean isExpired() {
        return Instant.now().isAfter(expiryTime.minusSeconds(30)); // consider refresh window
    }
}
```

---

### 2. **Configure Redis to Serialize Java Objects**

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;

@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, SnowflakeTokenCache> redisTemplate(RedisConnectionFactory connectionFactory) {
        RedisTemplate<String, SnowflakeTokenCache> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }
}
```

---

### 3. **Token Service Logic with Redis Cache**

```java
import org.springframework.beans.factory.annotation.Value;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.time.Instant;

@Service
public class SnowflakeTokenService {

    private static final String CACHE_KEY = "snowflake:token";

    @Value("${snowflake.token.expiration.time:3600}") // in seconds
    private long tokenValiditySeconds;

    private final SnowflakeTokenProvider tokenProvider;
    private final RedisTemplate<String, SnowflakeTokenCache> redisTemplate;

    public SnowflakeTokenService(SnowflakeTokenProvider tokenProvider,
                                 RedisTemplate<String, SnowflakeTokenCache> redisTemplate) {
        this.tokenProvider = tokenProvider;
        this.redisTemplate = redisTemplate;
    }

    public String getToken() {
        SnowflakeTokenCache cached = redisTemplate.opsForValue().get(CACHE_KEY);
        if (cached == null || cached.isExpired()) {
            return refreshToken();
        }
        return cached.getToken();
    }

    public String refreshToken() {
        String token = tokenProvider.fetchToken();
        Instant expiry = Instant.now().plusSeconds(tokenValiditySeconds);
        SnowflakeTokenCache tokenCache = new SnowflakeTokenCache(token, expiry);

        redisTemplate.opsForValue().set(CACHE_KEY, tokenCache, Duration.ofSeconds(tokenValiditySeconds));
        return token;
    }

    // Proactively refresh before expiry (e.g., every 5 mins or before expiry)
    @Scheduled(fixedDelay = 300000) // every 5 minutes
    public void scheduledRefresh() {
        SnowflakeTokenCache cached = redisTemplate.opsForValue().get(CACHE_KEY);
        if (cached == null || cached.isExpired()) {
            refreshToken();
        }
    }
}
```

---

### 4. **Token Provider Stub**

Replace with your actual Snowflake token generation logic:

```java
import org.springframework.stereotype.Component;

@Component
public class SnowflakeTokenProvider {

    public String fetchToken() {
        // Call to Snowflake authentication API
        return "mocked-snowflake-token-" + System.currentTimeMillis(); // Replace with actual call
    }
}
```

---

### ✅ Features Covered

* ✅ Stores both token and expiry timestamp in Redis.
* ✅ Automatically refreshes token 30 seconds before it expires.
* ✅ Avoids multiple calls by caching the full object.
* ✅ Uses proper Redis serialization for custom object storage.

Would you like me to also generate unit tests or a diagram of this flow?
