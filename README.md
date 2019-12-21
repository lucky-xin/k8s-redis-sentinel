# k8s Redis 哨兵模式，AOF数据持久化，以及SpringBoot sentinel方式连接
```text
定义了无头服务redis-svc用于k8s进群内部访问，StatefullSet的serviceName和Headless服务名称必须一致。
定义了NodePort用于外部访问，启动时第一个启动的pod为主节点。公共部分配置从config-map之中获取。
使用initContainers来初始化所有的配置文件，把配置文件都挂载到同一个volumeMounts之中initContainers来写配置文件
redis以及sentinel再从同一个volumeMounts之中获取配置文件
如果pods编号为redis-0则设置为主节点，其他节点访问主节点方式为
{pod-name}.{service-name}.{namespace}.svc.cluster.local，可以进入到pod中查看/etc/hosts
service-name为定义的Headless服务名称
```
## sentinel 必须等redis准备就绪之后再启动,使用一下脚本阻塞
```shell script
until redis-cli -p 6379 -a $REDIS_PWD info replication; do
  echo "Waiting for redis to be ready (accepting connections)"
  sleep 5
done
```

## 在当前项目的redis-sentinel目录下执行
```shell script
kubectl apply -f host-path-pv.yaml
kubectl apply -f redis-config.yaml
kubectl apply -f secret.yaml
kubectl apply -f redis-stateful-set.yaml
```

## 待服务都启动
### 查看pods信息如下：
![](https://github.com/lucky-xin/k8s-redis-sentinel/blob/master/.img/redis-stateful-set.png)

### 查看service信息如下：
![](https://github.com/lucky-xin/k8s-redis-sentinel/blob/master/.img/redis-svc.png)
## 执行
```shell script
kubectl exec -it redis-0 -c redis -- redis-cli -p 6379 -a 'Data*2019*' info replication
```
pod名称为redis-0容器中主节点信息输出如下：
```text
# Replication
role:master
connected_slaves:2
slave0:ip=10.1.3.223,port=6379,state=online,offset=638476,lag=0
slave1:ip=10.1.3.224,port=6379,state=online,offset=638198,lag=1
master_replid:e1727a2bef06cd874d6934abad415108801d148d
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:638476
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:638476
```

## 执行
```shell script
kubectl exec -it redis-1 -c redis -- redis-cli -p 6379 -a 'Data*2019*' info replication
```
pod名称为redis-1容器中的redis从节点信息输出如下：
```text
# Replication
role:slave
master_host:10.1.3.222
master_port:6379
master_link_status:up
master_last_io_seconds_ago:2
master_sync_in_progress:0
slave_repl_offset:648137
slave_priority:100
slave_read_only:1
connected_slaves:0
master_replid:e1727a2bef06cd874d6934abad415108801d148d
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:648137
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:648137
```
## 执行
```shell script
kubectl exec -it redis-2 -c redis -- redis-cli -p 6379 -a 'Data*2019*' info replication
```
pod名称为redis-2容器中的redis从节点信息输出如下：
```text
# Replication
role:slave
master_host:10.1.3.222
master_port:6379
master_link_status:up
master_last_io_seconds_ago:0
master_sync_in_progress:0
slave_repl_offset:661996
slave_priority:100
slave_read_only:1
connected_slaves:0
master_replid:e1727a2bef06cd874d6934abad415108801d148d
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:661996
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:4931
repl_backlog_histlen:657066
```
## 此时主节点ip为10.1.3.222 端口为6379

执行
```shell script
kubectl exec -it redis-0 -c redis -- redis-cli -p 26379  info
```
```text
查看sentinel状态信息如下
```
```text
# Server
redis_version:5.0.7
redis_git_sha1:00000000
redis_git_dirty:0
redis_build_id:7359662505fc6f11
redis_mode:sentinel
os:Linux 4.19.76-linuxkit x86_64
arch_bits:64
multiplexing_api:epoll
atomicvar_api:atomic-builtin
gcc_version:8.3.0
process_id:1
run_id:832ddf42a68af2ee3d0f46650c304426ede61d38
tcp_port:26379
uptime_in_seconds:3527
uptime_in_days:0
hz:14
configured_hz:10
lru_clock:16613416
executable:/data/redis-server
config_file:/usr/local/etc/redis/redis.conf

# Clients
connected_clients:3
client_recent_max_input_buffer:2
client_recent_max_output_buffer:0
blocked_clients:0

# CPU
used_cpu_sys:7.134090
used_cpu_user:4.483986
used_cpu_sys_children:0.000697
used_cpu_user_children:0.001395

# Stats
total_connections_received:1059
total_commands_processed:12304
instantaneous_ops_per_sec:2
total_net_input_bytes:620336
total_net_output_bytes:122821
instantaneous_input_kbps:0.13
instantaneous_output_kbps:0.02
rejected_connections:0
sync_full:0
sync_partial_ok:0
sync_partial_err:0
expired_keys:0
expired_stale_perc:0.00
expired_time_cap_reached_count:0
evicted_keys:0
keyspace_hits:0
keyspace_misses:0
pubsub_channels:0
pubsub_patterns:0
latest_fork_usec:0
migrate_cached_sockets:0
slave_expires_tracked_keys:0
active_defrag_hits:0
active_defrag_misses:0
active_defrag_key_hits:0
active_defrag_key_misses:0

# Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
sentinel_simulate_failure_flags:0
master0:name=redis_master,status=ok,address=10.1.3.222:6379,slaves=2,sentinels=3
```
```text
通过执行以上命令返回的消息显示此时redis主节点地址为10.1.3.222:6379
```
## SpringBoot 连接
```yaml
server:
  port: ${SERVER_PORT:19012}
  max-http-header-size: 30000

spring:
  main:
    allow-bean-definition-overriding: true
  application:
    name: @project.artifactId@
  profiles:
    active: dev
  # Redis Sentinel模式连接
  redis:
    sentinel:
      master: redis_master # 必须和Sentinel 配置文件的master name 一致
      #  192.168.31.90为k8s主节点ip,30637为redis服务类型为NodePort的nodePort    
      nodes: 192.168.31.90:30637
    password: Data*2019*
```
### pom.xml中添加如下配置
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```
### 代码配置
```java
/**
 * 扩展redis-cache支持注解cacheName添加超时时间，如果配置了Sentinel则使用Sentinel模式连接
 *
 * @author Luchaoxin
 * @date 2018/10/4
 */
@Slf4j
@Configuration
@ConditionalOnClass({GenericObjectPool.class, JedisConnection.class, Jedis.class})
@AutoConfigureBefore({RedisAutoConfiguration.class})
public class RedisConfiguration {

	@Bean
	@ConfigurationProperties(prefix = "spring.redis")
	public JedisPoolConfig getRedisConfig() {
		JedisPoolConfig config = new JedisPoolConfig();
		return config;
	}

	@Bean
	@ConditionalOnProperty(name = "spring.redis.sentinel.nodes")
	public RedisSentinelConfiguration sentinelConfiguration(RedisProperties redisProperties) {
		RedisSentinelConfiguration redisSentinelConfiguration = new RedisSentinelConfiguration();
		//配置matser的名称
		redisSentinelConfiguration.master(redisProperties.getSentinel().getMaster());
		//配置redis的哨兵sentinel
		Set<RedisNode> redisNodeSet = new HashSet<>();
		redisProperties.getSentinel().getNodes().forEach(x -> {
			String[] array = x.split(":");
			redisNodeSet.add(new RedisNode(array[0], Integer.parseInt(array[1])));
		});
		log.info("redisNodeSet -->" + redisNodeSet);
		redisSentinelConfiguration.setSentinels(redisNodeSet);
		redisSentinelConfiguration.setPassword(redisProperties.getPassword());
		return redisSentinelConfiguration;
	}

	@Bean
	@ConditionalOnMissingBean(RedisConnectionFactory.class)
	@ConditionalOnProperty(name = {"spring.redis.sentinel.nodes"})
	public RedisConnectionFactory redisConnectionFactory(JedisPoolConfig jedisPoolConfig,
														 RedisSentinelConfiguration sentinelConfig) {
		JedisConnectionFactory jedisConnectionFactory = new JedisConnectionFactory(sentinelConfig, jedisPoolConfig);
		return jedisConnectionFactory;
	}
}
```
### SpringBoot启动日志如下：
![](https://github.com/lucky-xin/k8s-redis-sentinel/blob/master/.img/SpringBoot%20Sentinel.png)
