# rabbitmq
redis分布式锁
@Component
@Slf4j
public class RedisLockService {

    @Autowired
    private StringRedisTemplate stringRedisTemplate;

    /** 加锁超时时间 */
    private long lockTimeOut = 20000;

    /** 释放锁异常超时时间 */
    private long releaseTimeOut = 20000;
    /**
     * 加锁
     * @param lockName 锁的key
     * @return 锁标识
     */
    public String lock(String lockName){
        return lockWithTimeout(lockName);
    }

    /**
     * 获取锁
     * @param lockName 锁的key
     * @return 锁标识
     */
    private String lockWithTimeout(String lockName) {
        String retIdentifier = null;
        RedisConnectionFactory connectionFactory = stringRedisTemplate.getConnectionFactory();
        RedisConnection redisConnection = connectionFactory.getConnection();
        // 获取连接
        // 随机生成一个value
        String identifier = UUID.randomUUID().toString();
        // 锁名，即key值
        String lockKey = "lock:" + lockName;
        // 超时时间，上锁后超过此时间则自动释放锁
        int lockExpire = (int)(lockTimeOut / 1000);
        // 获取锁的超时时间，超过这个时间则放弃获取锁
        long end = System.currentTimeMillis() + lockTimeOut;
        while (System.currentTimeMillis() < end) {
            try {
                if (redisConnection.setNX(lockKey.getBytes("UTF-8"), identifier.getBytes("UTF-8"))) {
                    redisConnection.expire(lockKey.getBytes("UTF-8"), lockExpire);
                    // 返回value值，用于释放锁时间确认
                    retIdentifier = identifier;
                    RedisConnectionUtils.releaseConnection(redisConnection, connectionFactory);
                    return retIdentifier;
                }
            } catch (Exception e) {
                log.error("加锁异常", e);
            }
            log.error("正在处理，被锁中，锁的key：" + lockName);
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                log.error("获取到分布式锁：线程中断！", e);
                Thread.currentThread().interrupt();
            }
        }
        RedisConnectionUtils.releaseConnection(redisConnection, connectionFactory);
        return retIdentifier;
    }

    /**
     * 释放锁
     * @param lockName 锁的key
     * @param identifier 释放锁的标识
     * @return
     */
    public boolean releaseLock(String lockName, String identifier) {
        RedisConnectionFactory connectionFactory = stringRedisTemplate.getConnectionFactory();
        RedisConnection redisConnection = connectionFactory.getConnection();
        String lockKey = "lock:" + lockName;
        boolean releaseFlag = false;
        long end = System.currentTimeMillis() + releaseTimeOut;
        while (System.currentTimeMillis() < end) {
            try {
                // 监视lock，准备开始事务
                redisConnection.watch(lockKey.getBytes("UTF-8"));
                // 通过前面返回的value值判断是不是该锁，若是该锁，则删除，释放锁
                byte[] valueBytes = redisConnection.get(lockKey.getBytes("UTF-8"));
                if (valueBytes == null) {
                    redisConnection.unwatch();
                    releaseFlag = false;
                    break;
                }
                String identifierValue = new String(valueBytes, "UTF-8");
                if (identifier.equals(identifierValue)) {
                    redisConnection.multi();
                    redisConnection.del(lockKey.getBytes("UTF-8"));
                    List<Object> results = redisConnection.exec();
                    if (results == null) {
                        continue;
                    }
                    releaseFlag = true;
                }
                redisConnection.unwatch();
                break;
            } catch (Exception e) {
                log.error("释放锁异常", e);
                try {
                    Thread.sleep(500);
                } catch (InterruptedException e1) {
                    log.error("释放锁异常：线程中断！", e1);
                    Thread.currentThread().interrupt();
                }
            }
        }
        RedisConnectionUtils.releaseConnection(redisConnection, connectionFactory);
        return releaseFlag;
    }
}
