# springboot-redis-cache


### Auto config by spring

~~~
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
            userOptional = Optional.ofNullable((User) redisUserTemplate.opsForHash().get(Constants.USER_HASH_KEY, userId));
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
            redisUserTemplate.opsForHash().put(Constants.USER_HASH_KEY, userId, user);
        } catch (RuntimeException exception) {
            log.error("Failed to store data in cache");
        }
        return user;
    }
}

~~~
