# lua脚本

> 简单了解下redis嵌入lua脚本（随便百度扒的）：

[Redis支持的LUA脚本与其优势](https://www.cnblogs.com/Don/articles/5731856.html)

[redis嵌入lua官方文档](https://redis.io/commands/eval)

[Redis悲观锁、乐观锁和调用Lua脚本的优缺点](https://blog.csdn.net/weixin_45743799/article/details/105023262)

# 序言
> 本教程本着人和代码其中一个能跑就行的原则。
>

![+1+1+1+1.gif](../../static/gif/%2B1%2B1%2B1%2B1.gif)

# 正文
> 通过redis嵌入lua脚本，实现简单的限流、黑名单功能。
> 直接丢代码。Talk is cheap. Show me the code.

 ![代码借我抄.jpg](../../static/img/%E4%BB%A3%E7%A0%81%E5%80%9F%E6%88%91%E6%8A%84.jpg)


## 测试环境
> win11 
>
> jdk8
>
> Redis server v=5.0.9
>
> springboot 2.4.7

## Show me the code.
> 先来看看lua脚本
>
> lua脚本存放在项目的`resource`目录下的`lua文件夹`下面（路径可以自己改，下面SelfRedisScript.java里面改成对应的就行）

![上号.jpg](../../static/img/%E4%B8%8A%E5%8F%B7.jpg)


```lua
--- lua脚本：限流、黑名单专用，慎改
--- 用于高并发情况下保证redis线程安全
--- 注意：
--- 1、redis反序列化问题
--- 2、完成lua脚本后，请在本地测试无误后再提交代码
--- 3、若lua脚本执行报错，redis不会回滚已经执行的命令

-- 获取传递进来的参数
local countKey = KEYS[1]
if countKey == nil then
    return true
end
-- 获取传递进来的阈值
local requestCount = KEYS[2]
-- 获取传递进来的过期时间ttl
local requestTtl = KEYS[3]
-- 获取redis参数
local countVal = redis.call('GET', countKey)
-- 如果不是第一次请求
if countVal then
    -- 由于lua脚本接收到参数都会转为String，所以要转成数字类型才能比较
    local numCountVal = tonumber(countVal)
    -- 如果超过指定阈值，则返回true
    if numCountVal >= tonumber(requestCount) then
        return true
    else
        numCountVal = numCountVal + 1
        redis.call('SETEX', countKey, requestTtl, numCountVal)
    end
else
    redis.call('SETEX', countKey, requestTtl, 1)
end
return false
```

> Java代码（SelfRedisScript .java）注入RedisScript
```java
@Component
public class SelfRedisScript {

    @Bean("redisScriptBoolean")
    public DefaultRedisScript<Boolean> redisScriptBoolean() {
        DefaultRedisScript<Boolean> redisScript = new DefaultRedisScript<>();
        redisScript.setScriptSource(new ResourceScriptSource(new ClassPathResource("lua/limit_blacklisted.lua")));
        redisScript.setResultType(Boolean.class);
        return redisScript;
    }
}
```
> Java代码（RedisTemplateConfig.java）简单配置RedisTemplate
```java
@EnableCaching
@Configuration
@AutoConfigureBefore(RedisAutoConfiguration.class)
public class RedisTemplateConfig {

	@Bean
	@Primary
	public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
		RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
		redisTemplate.setKeySerializer(new StringRedisSerializer());
		redisTemplate.setHashKeySerializer(new StringRedisSerializer());
		redisTemplate.setValueSerializer(new StringRedisSerializer());
		redisTemplate.setHashValueSerializer(new StringRedisSerializer());
		redisTemplate.setConnectionFactory(redisConnectionFactory);
		return redisTemplate;
	}
}
```

## 准备工作做完后，开始实现简单的限流、黑名单
> 在过滤器里面实现功能
>
> `CommonConstants`类中的常量、`ApplicationConfig`从`application.yml`从获取值的代码就不贴出来了
>
> `WebUtils.returnResponse`单独列出来了，自行修改补充。
>
>  `RedisConstants`常量：
> ```java
>    /**
>     * 限流机制key前缀 REQUEST_LIMIT:127.0.0.1:/api/test
>     */
>    String REQUEST_LIMIT = "REQUEST_LIMIT:%s:%s";
>
>    /**
>     * 黑名单机制key前缀 REQUEST_LIMIT:127.0.0.1:
>     */
>    String REQUEST_BLACKLISTED = "REQUEST_BLACKLISTED:%s";
> ```


```java
@Slf4j
public class RequestLimitFilter implements Filter {

    private final ApplicationConfig applicationConfig;
    private Long limitTimeSeconds;
    private Integer limitCount;
    private Long blacklistedTimeSeconds;
    private Integer blacklistedCount;
    private List<String> limitIgnores;
    private RedisTemplate<String, Object> redisTemplate;
    private DefaultRedisScript<Boolean> script;

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        HttpServletRequest request = (HttpServletRequest) servletRequest;
        HttpServletResponse response = (HttpServletResponse) servletResponse;
        String requestURI = request.getRequestURI();
        if (!WebUtils.uriMatch(this.limitIgnores, requestURI)) {
            // 获取ip
            String realIp = WebUtils.getIP(request);
            // 黑名单限制
            String blacklistedKey = String.format(RedisConstants.REQUEST_BLACKLISTED, realIp);
            // key为空返回true，超过指定阈值返回true，其他返回false
            Boolean blackPass = getPass(blacklistedKey, blacklistedCount, blacklistedTimeSeconds);
            if (blackPass) {
                WebUtils.returnResponse(response, JSONUtil.toJsonStr(R.failed(StatusCode.BLACKLISTED)));
                return;
            }
            // 限流限制
            String limitKey = String.format(RedisConstants.REQUEST_LIMIT, realIp, requestURI);
            Boolean limitPass = getPass(limitKey, limitCount, limitTimeSeconds);
            if (limitPass) {
                WebUtils.returnResponse(response, JSONUtil.toJsonStr(R.failed(StatusCode.LIMITED)));
                return;
            }
        }
        filterChain.doFilter(servletRequest, servletResponse);
    }

    @Override
    public void destroy() {
        Filter.super.destroy();
    }

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        // 限流
        this.limitCount = ObjectUtil.isNull(applicationConfig.getLimitCount())
                ? CommonConstants.REQUEST_LIMIT_COUNT : applicationConfig.getLimitCount();
        this.limitTimeSeconds = ObjectUtil.isNull(applicationConfig.getLimitTimeSeconds())
                ? CommonConstants.REQUEST_LIMIT_TIME_SECONDS : applicationConfig.getLimitTimeSeconds();
        // 黑名单
        this.blacklistedCount = ObjectUtil.isNull(applicationConfig.getBlacklistedCount())
                ? CommonConstants.REQUEST_BLACKLISTED_COUNT : applicationConfig.getBlacklistedCount();
        this.blacklistedTimeSeconds = ObjectUtil.isNull(applicationConfig.getBlacklistedTimeSeconds())
                ? CommonConstants.REQUEST_BLACKLISTED_TIME_SECONDS : applicationConfig.getBlacklistedTimeSeconds();
        // 过滤请求，从application.yml从获取值
        this.limitIgnores = IterUtil.isEmpty(applicationConfig.getLimitIgnores())
                ? Collections.emptyList() : applicationConfig.getLimitIgnores();
        // lua
        this.redisTemplate = SpringContextHolder.getBean(RedisTemplate.class);
        this.script = SpringContextHolder.getBean("redisScriptBoolean");
        Filter.super.init(filterConfig);
    }

    public RequestLimitFilter(ApplicationConfig applicationConfig) {
        this.applicationConfig = applicationConfig;
    }

    /**
     *  调用lua脚本，获取执行结果
     * @param key 缓存key
     * @param count 请求阈值
     * @param timeSeconds  拦截时间
     * @return 执行结果
     */
    private Boolean getPass(String key, Integer count, Long timeSeconds) {
        Boolean execute = redisTemplate.execute(script, Arrays.asList(key, String.valueOf(count), String.valueOf(timeSeconds)));
        return execute == null ? true : execute;
    }
}
```

> WebUtils工具类
```java
	public void returnResponse(HttpServletResponse response, String data) {
		response.setCharacterEncoding("UTF-8");
		response.setContentType("text/html; charset=utf-8");
		try (PrintWriter writer = response.getWriter()) {
			// 通过 PrintWriter 将 data 数据直接 print 回去      
			writer.print(data);
		} catch (IOException ignored) {
		}
	}

	public String getIP(HttpServletRequest request) {
		Assert.notNull(request, "HttpServletRequest is null");
		String ip = request.getHeader(HEADER_X_REQUESTED_FOR);
		if (StrUtil.isBlank(ip) || UNKNOWN.equalsIgnoreCase(ip)) {
			ip = request.getHeader(HEADER_X_FORWARDED_FOR);
		}
		if (StrUtil.isBlank(ip) || UNKNOWN.equalsIgnoreCase(ip)) {
			ip = request.getHeader(HEADER_PROXY_CLIENT_IP);
		}
		if (StrUtil.isBlank(ip) || UNKNOWN.equalsIgnoreCase(ip)) {
			ip = request.getHeader(HEADER_WL_PROXY_CLIENT_IP);
		}
		if (StrUtil.isBlank(ip) || UNKNOWN.equalsIgnoreCase(ip)) {
			ip = request.getHeader(HEADER_HTTP_CLIENT_IP);
		}
		if (StrUtil.isBlank(ip) || UNKNOWN.equalsIgnoreCase(ip)) {
			ip = request.getHeader(HEADER_HTTP_X_FORWARDED_FOR);
		}
		if (StrUtil.isBlank(ip) || UNKNOWN.equalsIgnoreCase(ip)) {
			ip = request.getRemoteAddr();
		}
		return StrUtil.isBlank(ip) ? null : ip.split(",")[0];
	}
```

> 最后注册下RequestLimitFilter.java这个过滤器
```java
@Component
@AllArgsConstructor
public class FilterRegistration {

    private final ApplicationConfig applicationConfig;

    @Bean
    public FilterRegistrationBean<RequestLimitFilter> requestLimitFilter() {
        FilterRegistrationBean<RequestLimitFilter> registration = new FilterRegistrationBean<>();
        registration.setFilter(new RequestLimitFilter(applicationConfig));
        registration.addUrlPatterns("/*");
        registration.setName("RequestLimitFilter");
        registration.setOrder(1);
        return registration;
    }
}
```

## 展示下成果（**计算规则自行调整**）
> 请求即记录

![请求即记录](../../static/project/Redis%26Lua/lua%E8%84%9A%E6%9C%AC/%E8%AF%B7%E6%B1%82%E5%8D%B3%E8%AE%B0%E5%BD%95.png)

> 时间段内多次请求达到限流指定的请求阈值

![达到限流指定的请求阈值](../../static/project/Redis%26Lua/lua%E8%84%9A%E6%9C%AC/%E8%BE%BE%E5%88%B0%E9%99%90%E6%B5%81%E6%8C%87%E5%AE%9A%E7%9A%84%E8%AF%B7%E6%B1%82%E9%98%88%E5%80%BC.png)

> 时间段内多次请求已被限流后，继续请求达到黑名单指定的请求阈值

![达到黑名单指定的请求阈值](../../static/project/Redis%26Lua/lua%E8%84%9A%E6%9C%AC/%E8%BE%BE%E5%88%B0%E9%BB%91%E5%90%8D%E5%8D%95%E6%8C%87%E5%AE%9A%E7%9A%84%E8%AF%B7%E6%B1%82%E9%98%88%E5%80%BC.png)

## 注意事项

![干饭了香喷喷.jpg](../../static/img/%E5%B9%B2%E9%A5%AD%E4%BA%86%E9%A6%99%E5%96%B7%E5%96%B7.jpg)

- RedisTemplate配置的序列化问题
> 如果配置的是JdkSerializationRedisSerializer，就需要改成StringRedisSerializer，如果需要两者兼容，那
>
> 就再给spring丢一个名为jdkRedisSerializer的Bean，然后在 @Autowired时，添加@Qualifier("jdkRedisSerializer")指定注入Bean
> 
> ```java
>   @Bean("jdkRedisSerializer")
>	public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
>		RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
>		redisTemplate.setKeySerializer(new StringRedisSerializer());
>		redisTemplate.setHashKeySerializer(new StringRedisSerializer());
>		redisTemplate.setValueSerializer(new JdkSerializationRedisSerializer());
>		redisTemplate.setHashValueSerializer(new JdkSerializationRedisSerializer());
>		redisTemplate.setConnectionFactory(redisConnectionFactory);
>		return redisTemplate;
>	}
> ```

- lua脚本执行报错问题
> 若lua脚本执行报错，redis不会回滚已经执行的命令，所以在完成lua脚本后，请在本地测试无误后再提交代码