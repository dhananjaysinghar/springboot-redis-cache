# springboot-redis-cache


### Auto config by spring

~~~
spring: 
  redis:
    host: localhost
    password: password
    port: 6379
  cache:
    redis:
      time-to-live: 60s
      
      
      
@Configuration
@EnableCaching
public class RedisConfig {

    @Bean
    public CacheManager cacheManager(LettuceConnectionFactory redisConnectionFactory) {
        return RedisCacheManager
                .builder(redisConnectionFactory)
                .cacheDefaults(RedisCacheConfiguration
                        .defaultCacheConfig()
                        .entryTtl(Duration.ofMinutes(1)))
                .build();
    }
}

~~~

~~~
@Cacheable(value = Constants.USER_HASH_KEY, key = "#userId") 
@CachePut(cacheNames = Constants.USER_HASH_KEY, key = "#userId")
@CacheEvict(cacheNames = Constants.USER_HASH_KEY, key = "#userId")
~~~



###
Manual Config

~~~
@Configuration
public class RedisConfig {

    @Value("${spring.redis.host}")
    private String host;

    @Value("${spring.redis.port}")
    private int port;

    @Value("${spring.redis.password}")
    private String password;

    @Bean
    public JedisConnectionFactory connectionFactory() {
        RedisStandaloneConfiguration configuration = new RedisStandaloneConfiguration();
        configuration.setHostName(host);
        configuration.setPort(port);
        configuration.setPassword(password);
        JedisConnectionFactory factory = new JedisConnectionFactory(configuration);
        factory.afterPropertiesSet();
        return factory;
    }

    @Bean
    public RedisTemplate<String, User> userTemplate() {
        RedisTemplate<String, User> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory());
        template.setKeySerializer(new StringRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setEnableTransactionSupport(true);
        template.afterPropertiesSet();
        return template;
    }
}

~~~


~~~

@Repository
@Slf4j
@RequiredArgsConstructor
public class UserRepository {

    private final RedisTemplate<String, User> redisUserTemplate;

    public User getUserDetails(String userId) {
        Optional<User> userOptional = Optional.empty();
        try {
            log.info("Trying to get user data from redis cache for id : {} and hashKey: {}", userId, Constants.USER_HASH_KEY);
            userOptional = Optional.ofNullable(redisUserTemplate.opsForValue().get(key));
        } catch (RuntimeException ex) {
            log.error("Failed to get data from redis cache");
        }
        return userOptional.orElseGet(() -> getUserFromDB(userId));
    }

    private User getUserFromDB(String userId) {
        log.info("Getting data from DB for id : {}", userId);
        User user = // get from DB
        try {
            log.info("Storing user info in redis cache for id : {} with hashKey : {}", userId, Constants.USER_HASH_KEY);
             redisUserTemplate.opsForValue().set(key, user, Duration.ofMinutes(1));  // TTL
        } catch (RuntimeException exception) {
            log.error("Failed to store data in cache");
        }
        return user;
    }
}

~~~


Hybrid:


~~~
@Configuration
@EnableCaching
public class RedisConfig {

    @Bean
    public RedisTemplate<String, User> userTemplate(LettuceConnectionFactory redisConnectionFactory) {
        RedisTemplate<String, User> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new Jackson2JsonRedisSerializer<>(User.class));
        template.setEnableTransactionSupport(true);
        template.afterPropertiesSet();
        return template;
    }
}

spring:
  redis:
    host: ${REDIS_HOST}
    password: ${REDIS_PASSWORD}
    port: ${REDIS_PORT}
    ssl: true
  cache:
    redis:
      time-to-live: 60s




@Repository
@Slf4j
@RequiredArgsConstructor
public class UserRepositoryImpl implements UserRepository {

    @Value("${spring.cache.redis.time-to-live:PT24H}")
    private Duration timeToLive;

    private final RedisTemplate<String, User> redisUserTemplate;

    private final DBService dbService;

    public User getUserDetails(String userId) {
        String key = Constants.USER_HASH_KEY + userId;
        Optional<User> userOptional = Optional.empty();
        try {
            log.info("Trying to get user data from redis cache for id : {} with key : {}", userId, key);
            userOptional = Optional.ofNullable(redisUserTemplate.opsForValue().get(key));
        } catch (RuntimeException ex) {
            log.error("Failed to get data from redis cache : {}", ex.getMessage());
        }
        return userOptional.orElseGet(() -> getUserFromDB(userId));
    }

    private User getUserFromDB(String userId) {
        String key = Constants.USER_HASH_KEY + userId;
        log.info("Getting data from DB for id : {}", userId);

        User user = service.getUserDetails(userId);
        try {
            log.info("Storing user info in redis cache for id : {} with key : {}", userId, key);
            redisUserTemplate.opsForValue().set(key, user, timeToLive);
        } catch (RuntimeException exception) {
            log.error("Failed to store data in cache : {}", exception.getMessage());
        }
        return user;
    }
}
~~~
