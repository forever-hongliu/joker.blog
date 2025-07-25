---
title: 04-LangChain4j MCP 性能优化与监控
date: 2025-07-25 18:41:58
tags:
  - MCP
  - Langchain4j
  - Java
  - Performance
categories: MCP
cover: https://cdn.pixabay.com/photo/2018/08/27/04/56/earth-3634034_1280.jpg
---

# LangChain4j MCP 性能优化与监控

> **LangChain4j MCP 系列第四篇** - 深入探讨 LangChain4j MCP 应用的性能优化技术、监控策略和调优实践

## 📋 目录

- [性能基准测试](#性能基准测试)
- [内存优化策略](#内存优化策略)
- [并发性能优化](#并发性能优化)
- [网络与I/O优化](#网络与io优化)
- [缓存策略优化](#缓存策略优化)
- [监控与告警体系](#监控与告警体系)

## 📊 性能基准测试

### 基准测试框架

```java
@Component
public class McpPerformanceBenchmark {
    
    @Autowired
    private McpClient mcpClient;
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    public BenchmarkResult runComprehensiveBenchmark() {
        BenchmarkResult.Builder result = BenchmarkResult.builder();
        
        // 1. 单线程性能测试
        result.singleThreadPerformance(benchmarkSingleThread());
        
        // 2. 多线程并发测试
        result.concurrentPerformance(benchmarkConcurrent());
        
        // 3. 内存使用测试
        result.memoryUsage(benchmarkMemoryUsage());
        
        // 4. 延迟分布测试
        result.latencyDistribution(benchmarkLatencyDistribution());
        
        // 5. 吞吐量测试
        result.throughputAnalysis(benchmarkThroughput());
        
        return result.build();
    }
    
    private SingleThreadBenchmark benchmarkSingleThread() {
        int iterations = 1000;
        List<Duration> latencies = new ArrayList<>();
        
        Instant start = Instant.now();
        
        for (int i = 0; i < iterations; i++) {
            Instant requestStart = Instant.now();
            
            try {
                ToolExecutionRequest request = ToolExecutionRequest.builder()
                    .name("echo")
                    .arguments(Map.of("text", "benchmark-" + i))
                    .build();
                    
                mcpClient.executeTool(request).get(30, TimeUnit.SECONDS);
                
                Duration latency = Duration.between(requestStart, Instant.now());
                latencies.add(latency);
                
            } catch (Exception e) {
                logger.error("Benchmark iteration {} failed", i, e);
            }
        }
        
        Duration totalTime = Duration.between(start, Instant.now());
        
        return SingleThreadBenchmark.builder()
            .iterations(iterations)
            .totalTime(totalTime)
            .averageLatency(calculateAverage(latencies))
            .p50Latency(calculatePercentile(latencies, 0.5))
            .p95Latency(calculatePercentile(latencies, 0.95))
            .p99Latency(calculatePercentile(latencies, 0.99))
            .throughput(iterations / totalTime.toSeconds())
            .build();
    }
    
    private ConcurrentBenchmark benchmarkConcurrent() {
        int threadCount = 20;
        int iterationsPerThread = 100;
        ExecutorService executor = Executors.newFixedThreadPool(threadCount);
        CountDownLatch latch = new CountDownLatch(threadCount);
        
        AtomicLong totalRequests = new AtomicLong(0);
        AtomicLong successfulRequests = new AtomicLong(0);
        AtomicLong failedRequests = new AtomicLong(0);
        List<Duration> allLatencies = Collections.synchronizedList(new ArrayList<>());
        
        Instant start = Instant.now();
        
        for (int t = 0; t < threadCount; t++) {
            final int threadId = t;
            executor.submit(() -> {
                try {
                    for (int i = 0; i < iterationsPerThread; i++) {
                        Instant requestStart = Instant.now();
                        totalRequests.incrementAndGet();
                        
                        try {
                            ToolExecutionRequest request = ToolExecutionRequest.builder()
                                .name("echo")
                                .arguments(Map.of("text", "thread-" + threadId + "-req-" + i))
                                .build();
                                
                            mcpClient.executeTool(request).get(30, TimeUnit.SECONDS);
                            
                            Duration latency = Duration.between(requestStart, Instant.now());
                            allLatencies.add(latency);
                            successfulRequests.incrementAndGet();
                            
                        } catch (Exception e) {
                            failedRequests.incrementAndGet();
                            logger.warn("Concurrent benchmark request failed", e);
                        }
                    }
                } finally {
                    latch.countDown();
                }
            });
        }
        
        try {
            latch.await(5, TimeUnit.MINUTES);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        Duration totalTime = Duration.between(start, Instant.now());
        executor.shutdown();
        
        return ConcurrentBenchmark.builder()
            .threadCount(threadCount)
            .iterationsPerThread(iterationsPerThread)
            .totalRequests(totalRequests.get())
            .successfulRequests(successfulRequests.get())
            .failedRequests(failedRequests.get())
            .totalTime(totalTime)
            .averageLatency(calculateAverage(allLatencies))
            .p95Latency(calculatePercentile(allLatencies, 0.95))
            .throughput(successfulRequests.get() / totalTime.toSeconds())
            .errorRate(failedRequests.get() / (double) totalRequests.get())
            .build();
    }
    
    private MemoryBenchmark benchmarkMemoryUsage() {
        MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
        
        // 执行 GC 获取基线
        System.gc();
        Thread.sleep(1000);
        
        MemoryUsage beforeHeap = memoryBean.getHeapMemoryUsage();
        MemoryUsage beforeNonHeap = memoryBean.getNonHeapMemoryUsage();
        
        // 执行大量操作
        int operations = 10000;
        for (int i = 0; i < operations; i++) {
            try {
                ToolExecutionRequest request = ToolExecutionRequest.builder()
                    .name("echo")
                    .arguments(Map.of("text", "memory-test-" + i))
                    .build();
                    
                mcpClient.executeTool(request).get(10, TimeUnit.SECONDS);
                
                if (i % 1000 == 0) {
                    // 记录中间状态
                    MemoryUsage currentHeap = memoryBean.getHeapMemoryUsage();
                    logger.debug("Memory usage at iteration {}: {} MB", 
                        i, currentHeap.getUsed() / 1024 / 1024);
                }
                
            } catch (Exception e) {
                logger.warn("Memory benchmark operation failed", e);
            }
        }
        
        // 再次执行 GC
        System.gc();
        Thread.sleep(1000);
        
        MemoryUsage afterHeap = memoryBean.getHeapMemoryUsage();
        MemoryUsage afterNonHeap = memoryBean.getNonHeapMemoryUsage();
        
        return MemoryBenchmark.builder()
            .operations(operations)
            .heapUsedBefore(beforeHeap.getUsed())
            .heapUsedAfter(afterHeap.getUsed())
            .heapMemoryIncrease(afterHeap.getUsed() - beforeHeap.getUsed())
            .nonHeapUsedBefore(beforeNonHeap.getUsed())
            .nonHeapUsedAfter(afterNonHeap.getUsed())
            .memoryPerOperation((afterHeap.getUsed() - beforeHeap.getUsed()) / operations)
            .build();
    }
}
```

### 性能回归测试

```java
@Component
public class PerformanceRegressionTester {
    
    private final PerformanceBaselineRepository baselineRepository;
    private final AlertManager alertManager;
    
    @Scheduled(cron = "0 0 2 * * ?") // 每天凌晨2点执行
    public void runRegressionTest() {
        logger.info("Starting performance regression test");
        
        try {
            // 1. 运行当前性能测试
            BenchmarkResult currentResult = mcpPerformanceBenchmark.runComprehensiveBenchmark();
            
            // 2. 获取历史基线
            PerformanceBaseline baseline = baselineRepository.getLatestBaseline();
            
            // 3. 比较性能指标
            RegressionAnalysis analysis = analyzeRegression(currentResult, baseline);
            
            // 4. 保存当前结果
            baselineRepository.saveResult(currentResult);
            
            // 5. 发送报告
            if (analysis.hasRegression()) {
                alertManager.sendPerformanceRegressionAlert(analysis);
            }
            
            generatePerformanceReport(currentResult, analysis);
            
        } catch (Exception e) {
            logger.error("Performance regression test failed", e);
            alertManager.sendTestFailureAlert("Performance regression test failed", e);
        }
    }
    
    private RegressionAnalysis analyzeRegression(BenchmarkResult current, PerformanceBaseline baseline) {
        RegressionAnalysis.Builder analysis = RegressionAnalysis.builder();
        
        // 吞吐量回归检查
        double throughputChange = (current.getThroughput() - baseline.getThroughput()) / baseline.getThroughput();
        if (throughputChange < -0.1) { // 吞吐量下降超过10%
            analysis.addRegression(RegressionType.THROUGHPUT_DEGRADATION, throughputChange);
        }
        
        // 延迟回归检查
        double latencyChange = (current.getP95Latency().toMillis() - baseline.getP95Latency().toMillis()) 
            / (double) baseline.getP95Latency().toMillis();
        if (latencyChange > 0.2) { // P95延迟增加超过20%
            analysis.addRegression(RegressionType.LATENCY_INCREASE, latencyChange);
        }
        
        // 内存使用回归检查
        double memoryChange = (current.getMemoryPerOperation() - baseline.getMemoryPerOperation()) 
            / (double) baseline.getMemoryPerOperation();
        if (memoryChange > 0.15) { // 内存使用增加超过15%
            analysis.addRegression(RegressionType.MEMORY_INCREASE, memoryChange);
        }
        
        // 错误率回归检查
        if (current.getErrorRate() > baseline.getErrorRate() + 0.01) { // 错误率增加超过1%
            analysis.addRegression(RegressionType.ERROR_RATE_INCREASE, 
                current.getErrorRate() - baseline.getErrorRate());
        }
        
        return analysis.build();
    }
}
```

## 🧠 内存优化策略

### 对象池化

```java
@Component
public class McpObjectPoolManager {
    
    // ToolExecutionRequest 对象池
    private final ObjectPool<ToolExecutionRequest.Builder> requestBuilderPool;
    
    // 响应对象池
    private final ObjectPool<ToolExecutionResult.Builder> resultBuilderPool;
    
    // 字符串缓冲区池
    private final ObjectPool<StringBuilder> stringBuilderPool;
    
    public McpObjectPoolManager() {
        // 配置请求构建器池
        this.requestBuilderPool = new GenericObjectPool<>(
            new BasePooledObjectFactory<ToolExecutionRequest.Builder>() {
                @Override
                public ToolExecutionRequest.Builder create() {
                    return ToolExecutionRequest.builder();
                }
                
                @Override
                public PooledObject<ToolExecutionRequest.Builder> wrap(ToolExecutionRequest.Builder obj) {
                    return new DefaultPooledObject<>(obj);
                }
                
                @Override
                public void passivateObject(PooledObject<ToolExecutionRequest.Builder> p) {
                    // 重置构建器状态
                    p.getObject().reset();
                }
            },
            createPoolConfig(50, 10)
        );
        
        // 配置字符串构建器池
        this.stringBuilderPool = new GenericObjectPool<>(
            new BasePooledObjectFactory<StringBuilder>() {
                @Override
                public StringBuilder create() {
                    return new StringBuilder(1024);
                }
                
                @Override
                public PooledObject<StringBuilder> wrap(StringBuilder obj) {
                    return new DefaultPooledObject<>(obj);
                }
                
                @Override
                public void passivateObject(PooledObject<StringBuilder> p) {
                    p.getObject().setLength(0); // 清空内容
                }
            },
            createPoolConfig(100, 20)
        );
    }
    
    public ToolExecutionRequest.Builder borrowRequestBuilder() {
        try {
            return requestBuilderPool.borrowObject();
        } catch (Exception e) {
            logger.warn("Failed to borrow request builder from pool, creating new one", e);
            return ToolExecutionRequest.builder();
        }
    }
    
    public void returnRequestBuilder(ToolExecutionRequest.Builder builder) {
        try {
            requestBuilderPool.returnObject(builder);
        } catch (Exception e) {
            logger.warn("Failed to return request builder to pool", e);
        }
    }
    
    public StringBuilder borrowStringBuilder() {
        try {
            return stringBuilderPool.borrowObject();
        } catch (Exception e) {
            logger.warn("Failed to borrow StringBuilder from pool, creating new one", e);
            return new StringBuilder(1024);
        }
    }
    
    public void returnStringBuilder(StringBuilder sb) {
        try {
            stringBuilderPool.returnObject(sb);
        } catch (Exception e) {
            logger.warn("Failed to return StringBuilder to pool", e);
        }
    }
    
    private GenericObjectPoolConfig<Object> createPoolConfig(int maxTotal, int maxIdle) {
        GenericObjectPoolConfig<Object> config = new GenericObjectPoolConfig<>();
        config.setMaxTotal(maxTotal);
        config.setMaxIdle(maxIdle);
        config.setMinIdle(5);
        config.setTestOnBorrow(false);
        config.setTestOnReturn(false);
        config.setTestWhileIdle(true);
        config.setTimeBetweenEvictionRunsMillis(30000);
        config.setBlockWhenExhausted(false);
        return config;
    }
}
```

### 内存泄漏检测

```java
@Component
public class MemoryLeakDetector {
    
    private final Map<String, WeakReference<Object>> trackedObjects = new ConcurrentHashMap<>();
    private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
    
    @PostConstruct
    public void startMonitoring() {
        // 每5分钟检查一次内存泄漏
        scheduler.scheduleAtFixedRate(this::detectLeaks, 5, 5, TimeUnit.MINUTES);
        
        // 每小时生成内存报告
        scheduler.scheduleAtFixedRate(this::generateMemoryReport, 1, 1, TimeUnit.HOURS);
    }
    
    public void trackObject(String key, Object object) {
        trackedObjects.put(key, new WeakReference<>(object));
    }
    
    private void detectLeaks() {
        try {
            // 强制垃圾回收
            System.gc();
            Thread.sleep(1000);
            System.gc();
            
            // 检查弱引用
            List<String> potentialLeaks = new ArrayList<>();
            
            trackedObjects.entrySet().removeIf(entry -> {
                WeakReference<Object> ref = entry.getValue();
                if (ref.get() == null) {
                    return true; // 对象已被回收，移除引用
                } else {
                    // 对象仍然存在，可能是内存泄漏
                    potentialLeaks.add(entry.getKey());
                    return false;
                }
            });
            
            if (!potentialLeaks.isEmpty()) {
                logger.warn("Potential memory leaks detected: {}", potentialLeaks);
                
                // 生成堆转储进行详细分析
                if (potentialLeaks.size() > 100) {
                    generateHeapDump();
                }
            }
            
        } catch (Exception e) {
            logger.error("Memory leak detection failed", e);
        }
    }
    
    private void generateMemoryReport() {
        MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
        List<GarbageCollectorMXBean> gcBeans = ManagementFactory.getGarbageCollectorMXBeans();
        
        MemoryUsage heapUsage = memoryBean.getHeapMemoryUsage();
        MemoryUsage nonHeapUsage = memoryBean.getNonHeapMemoryUsage();
        
        StringBuilder report = new StringBuilder();
        report.append("=== Memory Report ===\n");
        report.append(String.format("Heap Memory: Used=%d MB, Max=%d MB, Usage=%.2f%%\n",
            heapUsage.getUsed() / 1024 / 1024,
            heapUsage.getMax() / 1024 / 1024,
            (double) heapUsage.getUsed() / heapUsage.getMax() * 100));
            
        report.append(String.format("Non-Heap Memory: Used=%d MB, Max=%d MB\n",
            nonHeapUsage.getUsed() / 1024 / 1024,
            nonHeapUsage.getMax() / 1024 / 1024));
            
        report.append("Garbage Collection:\n");
        for (GarbageCollectorMXBean gcBean : gcBeans) {
            report.append(String.format("  %s: Collections=%d, Time=%d ms\n",
                gcBean.getName(),
                gcBean.getCollectionCount(),
                gcBean.getCollectionTime()));
        }
        
        report.append(String.format("Tracked Objects: %d\n", trackedObjects.size()));
        
        logger.info(report.toString());
        
        // 记录到监控系统
        recordMemoryMetrics(heapUsage, nonHeapUsage, gcBeans);
    }
    
    private void generateHeapDump() {
        try {
            MBeanServer server = ManagementFactory.getPlatformMBeanServer();
            HotSpotDiagnosticMXBean mxBean = ManagementFactory.newPlatformMXBeanProxy(
                server, "com.sun.management:type=HotSpotDiagnostic", HotSpotDiagnosticMXBean.class);
                
            String fileName = String.format("/tmp/heapdump-%d.hprof", System.currentTimeMillis());
            mxBean.dumpHeap(fileName, true);
            
            logger.info("Heap dump generated: {}", fileName);
            
        } catch (Exception e) {
            logger.error("Failed to generate heap dump", e);
        }
    }
}
```

## ⚡ 并发性能优化

### 自适应线程池

```java
@Component
public class AdaptiveThreadPoolManager {
    
    private volatile ThreadPoolTaskExecutor mcpExecutor;
    private final AtomicInteger activeRequests = new AtomicInteger(0);
    private final AtomicLong totalExecutionTime = new AtomicLong(0);
    private final AtomicLong executedTasks = new AtomicLong(0);
    
    @PostConstruct
    public void initialize() {
        mcpExecutor = createInitialThreadPool();
        
        // 启动自适应调整
        ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
        scheduler.scheduleAtFixedRate(this::adjustThreadPool, 1, 1, TimeUnit.MINUTES);
    }
    
    private ThreadPoolTaskExecutor createInitialThreadPool() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(200);
        executor.setThreadNamePrefix("mcp-adaptive-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(30);
        executor.initialize();
        return executor;
    }
    
    public CompletableFuture<ToolExecutionResult> executeAsync(
            Supplier<ToolExecutionResult> task) {
        
        return CompletableFuture.supplyAsync(() -> {
            activeRequests.incrementAndGet();
            long startTime = System.nanoTime();
            
            try {
                return task.get();
            } finally {
                long executionTime = System.nanoTime() - startTime;
                totalExecutionTime.addAndGet(executionTime);
                executedTasks.incrementAndGet();
                activeRequests.decrementAndGet();
            }
        }, mcpExecutor);
    }
    
    private void adjustThreadPool() {
        try {
            ThreadPoolExecutor executor = mcpExecutor.getThreadPoolExecutor();
            
            // 收集性能指标
            int currentCoreSize = executor.getCorePoolSize();
            int currentMaxSize = executor.getMaximumPoolSize();
            int activeCount = executor.getActiveCount();
            int queueSize = executor.getQueue().size();
            long completedTasks = executor.getCompletedTaskCount();
            
            // 计算平均执行时间
            long totalTasks = executedTasks.get();
            double avgExecutionTime = totalTasks > 0 ? 
                (totalExecutionTime.get() / totalTasks) / 1_000_000.0 : 0; // 转换为毫秒
            
            // 计算利用率
            double utilization = (double) activeCount / currentCoreSize;
            
            logger.debug("Thread pool metrics - Core: {}, Max: {}, Active: {}, Queue: {}, " +
                "Utilization: {:.2f}, AvgExecTime: {:.2f}ms",
                currentCoreSize, currentMaxSize, activeCount, queueSize, utilization, avgExecutionTime);
            
            // 自适应调整逻辑
            if (utilization > 0.8 && queueSize > 10) {
                // 高利用率且队列积压，增加核心线程数
                int newCoreSize = Math.min(currentCoreSize + 2, 30);
                executor.setCorePoolSize(newCoreSize);
                logger.info("Increased core pool size to {}", newCoreSize);
                
            } else if (utilization < 0.3 && currentCoreSize > 5) {
                // 低利用率，减少核心线程数
                int newCoreSize = Math.max(currentCoreSize - 1, 5);
                executor.setCorePoolSize(newCoreSize);
                logger.info("Decreased core pool size to {}", newCoreSize);
            }
            
            // 调整最大线程数
            if (queueSize > 50 && currentMaxSize < 100) {
                int newMaxSize = Math.min(currentMaxSize + 5, 100);
                executor.setMaximumPoolSize(newMaxSize);
                logger.info("Increased max pool size to {}", newMaxSize);
                
            } else if (queueSize == 0 && activeCount < currentMaxSize * 0.5 && currentMaxSize > 20) {
                int newMaxSize = Math.max(currentMaxSize - 5, 20);
                executor.setMaximumPoolSize(newMaxSize);
                logger.info("Decreased max pool size to {}", newMaxSize);
            }
            
            // 重置统计数据
            totalExecutionTime.set(0);
            executedTasks.set(0);
            
        } catch (Exception e) {
            logger.error("Failed to adjust thread pool", e);
        }
    }
}
```

### 并发限流器

```java
@Component
public class ConcurrentRateLimiter {
    
    private final Map<String, Semaphore> clientLimiters = new ConcurrentHashMap<>();
    private final Map<String, RateLimiter> rateLimiters = new ConcurrentHashMap<>();
    private final LoadingCache<String, AtomicInteger> requestCounters;
    
    public ConcurrentRateLimiter() {
        this.requestCounters = Caffeine.newBuilder()
            .maximumSize(10000)
            .expireAfterWrite(1, TimeUnit.MINUTES)
            .build(key -> new AtomicInteger(0));
    }
    
    public CompletableFuture<ToolExecutionResult> executeWithRateLimit(
            String clientId, Supplier<CompletableFuture<ToolExecutionResult>> operation) {
        
        // 1. 检查并发限制
        Semaphore concurrencyLimiter = clientLimiters.computeIfAbsent(clientId, 
            k -> new Semaphore(getMaxConcurrency(clientId)));
            
        if (!concurrencyLimiter.tryAcquire()) {
            return CompletableFuture.completedFuture(
                createErrorResult("Concurrency limit exceeded for client: " + clientId));
        }
        
        // 2. 检查速率限制
        RateLimiter rateLimiter = rateLimiters.computeIfAbsent(clientId,
            k -> RateLimiter.create(getMaxRequestsPerSecond(clientId)));
            
        if (!rateLimiter.tryAcquire(Duration.ofMillis(100))) {
            concurrencyLimiter.release();
            return CompletableFuture.completedFuture(
                createErrorResult("Rate limit exceeded for client: " + clientId));
        }
        
        // 3. 记录请求
        requestCounters.get(clientId).incrementAndGet();
        
        // 4. 执行操作
        return operation.get()
            .whenComplete((result, throwable) -> {
                concurrencyLimiter.release();
                
                // 记录指标
                if (throwable != null) {
                    recordFailure(clientId, throwable);
                } else {
                    recordSuccess(clientId);
                }
            });
    }
    
    private int getMaxConcurrency(String clientId) {
        // 根据客户端类型和历史性能确定并发限制
        ClientProfile profile = clientProfileService.getProfile(clientId);
        
        return switch (profile.getTier()) {
            case PREMIUM -> 50;
            case STANDARD -> 20;
            case BASIC -> 5;
        };
    }
    
    private double getMaxRequestsPerSecond(String clientId) {
        ClientProfile profile = clientProfileService.getProfile(clientId);
        
        return switch (profile.getTier()) {
            case PREMIUM -> 100.0;
            case STANDARD -> 50.0;
            case BASIC -> 10.0;
        };
    }
    
    @Scheduled(fixedRate = 60000) // 每分钟调整一次
    public void adjustLimits() {
        clientLimiters.forEach((clientId, semaphore) -> {
            // 根据客户端性能动态调整限制
            ClientMetrics metrics = getClientMetrics(clientId);
            
            if (metrics.getErrorRate() > 0.1) {
                // 错误率高，降低限制
                int currentLimit = semaphore.availablePermits() + semaphore.getQueueLength();
                int newLimit = Math.max(currentLimit / 2, 1);
                
                // 重新创建信号量
                clientLimiters.put(clientId, new Semaphore(newLimit));
                logger.info("Reduced concurrency limit for client {} to {}", clientId, newLimit);
                
            } else if (metrics.getAverageLatency().toMillis() < 100 && metrics.getErrorRate() < 0.01) {
                // 性能良好，可以适当提高限制
                int currentLimit = semaphore.availablePermits() + semaphore.getQueueLength();
                int maxLimit = getMaxConcurrency(clientId);
                int newLimit = Math.min(currentLimit + 2, maxLimit);
                
                if (newLimit > currentLimit) {
                    clientLimiters.put(clientId, new Semaphore(newLimit));
                    logger.info("Increased concurrency limit for client {} to {}", clientId, newLimit);
                }
            }
        });
    }
}
```

## 🌐 网络与I/O优化

### 连接池优化

```java
@Configuration
public class NetworkOptimizationConfiguration {
    
    @Bean
    public OkHttpClient optimizedHttpClient() {
        return new OkHttpClient.Builder()
            // 连接池配置
            .connectionPool(new ConnectionPool(50, 5, TimeUnit.MINUTES))
            
            // 超时配置
            .connectTimeout(10, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .writeTimeout(30, TimeUnit.SECONDS)
            .callTimeout(60, TimeUnit.SECONDS)
            
            // 重试配置
            .retryOnConnectionFailure(true)
            
            // 协议配置
            .protocols(Arrays.asList(Protocol.HTTP_2, Protocol.HTTP_1_1))
            
            // 压缩配置
            .addInterceptor(new GzipRequestInterceptor())
            
            // 连接保活
            .addNetworkInterceptor(new ConnectionKeepAliveInterceptor())
            
            // 监控拦截器
            .addInterceptor(new NetworkMetricsInterceptor())
            
            build();
    }
    
    // GZIP 压缩拦截器
    private static class GzipRequestInterceptor implements Interceptor {
        @Override
        public Response intercept(Chain chain) throws IOException {
            Request originalRequest = chain.request();
            
            if (originalRequest.body() == null || 
                originalRequest.header("Content-Encoding") != null) {
                return chain.proceed(originalRequest);
            }
            
            Request compressedRequest = originalRequest.newBuilder()
                .header("Content-Encoding", "gzip")
                .method(originalRequest.method(), gzip(originalRequest.body()))
                .build();
                
            return chain.proceed(compressedRequest);
        }
        
        private RequestBody gzip(final RequestBody body) {
            return new RequestBody() {
                @Override
                public MediaType contentType() {
                    return body.contentType();
                }
                
                @Override
                public long contentLength() {
                    return -1; // 无法预先知道压缩后的长度
                }
                
                @Override
                public void writeTo(BufferedSink sink) throws IOException {
                    BufferedSink gzipSink = Okio.buffer(new GzipSink(sink));
                    body.writeTo(gzipSink);
                    gzipSink.close();
                }
            };
        }
    }
    
    // 连接保活拦截器
    private static class ConnectionKeepAliveInterceptor implements Interceptor {
        @Override
        public Response intercept(Chain chain) throws IOException {
            Response response = chain.proceed(chain.request());
            
            // 添加连接保活头
            return response.newBuilder()
                .header("Connection", "keep-alive")
                .header("Keep-Alive", "timeout=60, max=100")
                .build();
        }
    }
}
```

### 异步I/O优化

```java
@Component
public class AsyncIOOptimizer {
    
    private final CompletableFuture<Void> ioExecutor;
    private final Channel<IOTask> ioTaskChannel;
    
    @PostConstruct
    public void initialize() {
        // 创建专用的I/O线程池
        ForkJoinPool ioPool = new ForkJoinPool(
            Runtime.getRuntime().availableProcessors() * 2,
            ForkJoinPool.defaultForkJoinWorkerThreadFactory,
            null,
            true // 异步模式
        );
        
        // 创建I/O任务通道
        this.ioTaskChannel = Channel.create(1000);
        
        // 启动I/O处理器
        startIOProcessor(ioPool);
    }
    
    public CompletableFuture<String> readAsync(String path) {
        IOTask task = IOTask.builder()
            .type(IOTaskType.READ)
            .path(path)
            .future(new CompletableFuture<>())
            .build();
            
        try {
            ioTaskChannel.send(task);
            return task.getFuture();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return CompletableFuture.failedFuture(e);
        }
    }
    
    public CompletableFuture<Void> writeAsync(String path, String content) {
        IOTask task = IOTask.builder()
            .type(IOTaskType.WRITE)
            .path(path)
            .content(content)
            .future(new CompletableFuture<>())
            .build();
            
        try {
            ioTaskChannel.send(task);
            return task.getFuture();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return CompletableFuture.failedFuture(e);
        }
    }
    
    private void startIOProcessor(ForkJoinPool ioPool) {
        CompletableFuture.runAsync(() -> {
            while (!Thread.currentThread().isInterrupted()) {
                try {
                    IOTask task = ioTaskChannel.receive();
                    
                    // 异步处理I/O任务
                    CompletableFuture.runAsync(() -> processIOTask(task), ioPool);
                    
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                } catch (Exception e) {
                    logger.error("IO processor error", e);
                }
            }
        });
    }
    
    private void processIOTask(IOTask task) {
        try {
            switch (task.getType()) {
                case READ:
                    String content = Files.readString(Paths.get(task.getPath()));
                    task.getFuture().complete(content);
                    break;
                    
                case WRITE:
                    Files.writeString(Paths.get(task.getPath()), task.getContent());
                    task.getFuture().complete(null);
                    break;
                    
                default:
                    task.getFuture().completeExceptionally(
                        new UnsupportedOperationException("Unknown IO task type: " + task.getType()));
            }
        } catch (Exception e) {
            task.getFuture().completeExceptionally(e);
        }
    }
}
```

## 💾 缓存策略优化

### 多级缓存实现

```java
@Component
public class MultiLevelCacheManager {
    
    // L1: 本地内存缓存 (Caffeine)
    private final Cache<String, Object> l1Cache;
    
    // L2: 分布式缓存 (Redis)
    private final RedisTemplate<String, Object> redisTemplate;
    
    // L3: 数据库缓存
    private final DatabaseCacheService databaseCache;
    
    public MultiLevelCacheManager(RedisTemplate<String, Object> redisTemplate,
                                 DatabaseCacheService databaseCache) {
        this.redisTemplate = redisTemplate;
        this.databaseCache = databaseCache;
        
        // 配置L1缓存
        this.l1Cache = Caffeine.newBuilder()
            .maximumSize(10000)
            .expireAfterWrite(5, TimeUnit.MINUTES)
            .expireAfterAccess(2, TimeUnit.MINUTES)
            .recordStats()
            .removalListener((key, value, cause) -> {
                logger.debug("L1 cache eviction: key={}, cause={}", key, cause);
            })
            .build();
    }
    
    public CompletableFuture<Object> get(String key) {
        // L1 缓存查找
        Object value = l1Cache.getIfPresent(key);
        if (value != null) {
            recordCacheHit("L1", key);
            return CompletableFuture.completedFuture(value);
        }
        
        // L2 缓存查找
        return CompletableFuture.supplyAsync(() -> {
            Object l2Value = redisTemplate.opsForValue().get(key);
            if (l2Value != null) {
                recordCacheHit("L2", key);
                // 回填到L1
                l1Cache.put(key, l2Value);
                return l2Value;
            }
            
            // L3 缓存查找
            Object l3Value = databaseCache.get(key);
            if (l3Value != null) {
                recordCacheHit("L3", key);
                // 回填到L2和L1
                redisTemplate.opsForValue().set(key, l3Value, Duration.ofMinutes(30));
                l1Cache.put(key, l3Value);
                return l3Value;
            }
            
            recordCacheMiss(key);
            return null;
        });
    }
    
    public CompletableFuture<Void> put(String key, Object value, Duration ttl) {
        return CompletableFuture.runAsync(() -> {
            // 写入所有缓存层级
            l1Cache.put(key, value);
            redisTemplate.opsForValue().set(key, value, ttl);
            databaseCache.put(key, value, ttl);
        });
    }
    
    public CompletableFuture<Void> evict(String key) {
        return CompletableFuture.runAsync(() -> {
            l1Cache.invalidate(key);
            redisTemplate.delete(key);
            databaseCache.evict(key);
        });
    }
    
    // 智能预热
    @Scheduled(fixedRate = 300000) // 每5分钟
    public void intelligentWarmup() {
        // 分析访问模式
        List<String> hotKeys = analyzeHotKeys();
        
        // 预热热点数据
        hotKeys.forEach(key -> {
            if (l1Cache.getIfPresent(key) == null) {
                get(key).thenAccept(value -> {
                    if (value != null) {
                        logger.debug("Prewarmed cache key: {}", key);
                    }
                });
            }
        });
    }
    
    private List<String> analyzeHotKeys() {
        // 基于访问频率和最近访问时间分析热点键
        return cacheAccessAnalyzer.getHotKeys(Duration.ofHours(1));
    }
    
    @Scheduled(fixedRate = 60000) // 每分钟
    public void reportCacheMetrics() {
        CacheStats l1Stats = l1Cache.stats();
        
        Metrics.gauge("cache.l1.size", l1Cache.estimatedSize());
        Metrics.gauge("cache.l1.hit.rate", l1Stats.hitRate());
        Metrics.gauge("cache.l1.miss.rate", l1Stats.missRate());
        Metrics.gauge("cache.l1.eviction.count", l1Stats.evictionCount());
        
        // Redis 缓存指标
        RedisConnectionFactory connectionFactory = redisTemplate.getConnectionFactory();
        if (connectionFactory instanceof LettuceConnectionFactory) {
            // 获取Redis连接信息
            recordRedisMetrics();
        }
    }
}
```

## 📈 监控与告警体系

### 实时性能监控

```java
@Component
public class RealTimePerformanceMonitor {
    
    private final MeterRegistry meterRegistry;
    private final SlidingWindowReservoir latencyReservoir;
    private final SlidingWindowReservoir throughputReservoir;
    
    public RealTimePerformanceMonitor(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.latencyReservoir = new SlidingWindowReservoir(1000);
        this.throughputReservoir = new SlidingWindowReservoir(60);
        
        // 注册自定义指标
        Gauge.builder("mcp.performance.latency.p95")
            .register(meterRegistry, this, monitor -> monitor.getP95Latency());
            
        Gauge.builder("mcp.performance.throughput.current")
            .register(meterRegistry, this, monitor -> monitor.getCurrentThroughput());
    }
    
    @EventListener
    public void handleToolExecution(ToolExecutionCompletedEvent event) {
        // 记录延迟
        latencyReservoir.update(event.getDuration().toMillis());
        
        // 记录吞吐量
        throughputReservoir.update(1);
        
        // 检查性能阈值
        checkPerformanceThresholds(event);
    }
    
    private void checkPerformanceThresholds(ToolExecutionCompletedEvent event) {
        Duration latency = event.getDuration();
        
        // 延迟告警
        if (latency.toMillis() > 5000) { // 5秒
            alertManager.sendLatencyAlert(event.getToolName(), latency);
        }
        
        // 错误率告警
        double currentErrorRate = calculateCurrentErrorRate();
        if (currentErrorRate > 0.05) { // 5%
            alertManager.sendErrorRateAlert(currentErrorRate);
        }
        
        // 吞吐量告警
        double currentThroughput = getCurrentThroughput();
        double expectedThroughput = getExpectedThroughput();
        if (currentThroughput < expectedThroughput * 0.7) { // 低于期望的70%
            alertManager.sendThroughputAlert(currentThroughput, expectedThroughput);
        }
    }
    
    public double getP95Latency() {
        Snapshot snapshot = latencyReservoir.getSnapshot();
        return snapshot.get95thPercentile();
    }
    
    public double getCurrentThroughput() {
        Snapshot snapshot = throughputReservoir.getSnapshot();
        return snapshot.getMean(); // 每秒请求数
    }
    
    @Scheduled(fixedRate = 30000) // 每30秒
    public void generatePerformanceReport() {
        PerformanceSnapshot snapshot = PerformanceSnapshot.builder()
            .timestamp(Instant.now())
            .p50Latency(latencyReservoir.getSnapshot().getMedian())
            .p95Latency(getP95Latency())
            .p99Latency(latencyReservoir.getSnapshot().get99thPercentile())
            .throughput(getCurrentThroughput())
            .errorRate(calculateCurrentErrorRate())
            .activeConnections(getActiveConnectionCount())
            .memoryUsage(getMemoryUsage())
            .cpuUsage(getCpuUsage())
            .build();
            
        // 保存快照
        performanceSnapshotRepository.save(snapshot);
        
        // 发送到监控系统
        monitoringService.sendSnapshot(snapshot);
    }
}
```

### 智能告警系统

```java
@Component
public class IntelligentAlertManager {
    
    private final Map<String, AlertRule> alertRules = new ConcurrentHashMap<>();
    private final Map<String, AlertState> alertStates = new ConcurrentHashMap<>();
    private final NotificationService notificationService;
    
    @PostConstruct
    public void initializeAlertRules() {
        // 延迟告警规则
        alertRules.put("high_latency", AlertRule.builder()
            .name("High Latency")
            .condition("p95_latency > 5000") // 5秒
            .severity(AlertSeverity.WARNING)
            .cooldownPeriod(Duration.ofMinutes(5))
            .escalationRules(Arrays.asList(
                EscalationRule.builder()
                    .condition("p95_latency > 10000") // 10秒
                    .severity(AlertSeverity.CRITICAL)
                    .delay(Duration.ofMinutes(2))
                    .build()
            ))
            .build());
            
        // 错误率告警规则
        alertRules.put("high_error_rate", AlertRule.builder()
            .name("High Error Rate")
            .condition("error_rate > 0.05") // 5%
            .severity(AlertSeverity.WARNING)
            .cooldownPeriod(Duration.ofMinutes(3))
            .build());
            
        // 内存使用告警规则
        alertRules.put("high_memory_usage", AlertRule.builder()
            .name("High Memory Usage")
            .condition("memory_usage > 0.85") // 85%
            .severity(AlertSeverity.WARNING)
            .cooldownPeriod(Duration.ofMinutes(10))
            .build());
    }
    
    @EventListener
    public void evaluateAlerts(PerformanceMetricsEvent event) {
        alertRules.forEach((ruleId, rule) -> {
            try {
                boolean conditionMet = evaluateCondition(rule.getCondition(), event.getMetrics());
                AlertState currentState = alertStates.get(ruleId);
                
                if (conditionMet) {
                    if (currentState == null || currentState.getState() == AlertStateType.RESOLVED) {
                        // 新告警触发
                        triggerAlert(ruleId, rule, event.getMetrics());
                    } else if (currentState.getState() == AlertStateType.FIRING) {
                        // 检查是否需要升级
                        checkEscalation(ruleId, rule, currentState, event.getMetrics());
                    }
                } else {
                    if (currentState != null && currentState.getState() == AlertStateType.FIRING) {
                        // 告警恢复
                        resolveAlert(ruleId, rule);
                    }
                }
                
            } catch (Exception e) {
                logger.error("Failed to evaluate alert rule: {}", ruleId, e);
            }
        });
    }
    
    private void triggerAlert(String ruleId, AlertRule rule, PerformanceMetrics metrics) {
        AlertState state = AlertState.builder()
            .ruleId(ruleId)
            .state(AlertStateType.FIRING)
            .severity(rule.getSeverity())
            .triggeredAt(Instant.now())
            .metrics(metrics)
            .build();
            
        alertStates.put(ruleId, state);
        
        // 发送通知
        Alert alert = Alert.builder()
            .id(UUID.randomUUID().toString())
            .ruleId(ruleId)
            .name(rule.getName())
            .severity(rule.getSeverity())
            .message(generateAlertMessage(rule, metrics))
            .triggeredAt(state.getTriggeredAt())
            .build();
            
        notificationService.sendAlert(alert);
        
        logger.warn("Alert triggered: {} - {}", rule.getName(), alert.getMessage());
    }
    
    private void checkEscalation(String ruleId, AlertRule rule, AlertState currentState, 
                                PerformanceMetrics metrics) {
        
        for (EscalationRule escalationRule : rule.getEscalationRules()) {
            if (evaluateCondition(escalationRule.getCondition(), metrics)) {
                Duration timeSinceTriggered = Duration.between(currentState.getTriggeredAt(), Instant.now());
                
                if (timeSinceTriggered.compareTo(escalationRule.getDelay()) >= 0 &&
                    currentState.getSeverity().ordinal() < escalationRule.getSeverity().ordinal()) {
                    
                    // 升级告警
                    escalateAlert(ruleId, rule, escalationRule, metrics);
                    break;
                }
            }
        }
    }
    
    private void escalateAlert(String ruleId, AlertRule rule, EscalationRule escalationRule, 
                              PerformanceMetrics metrics) {
        
        AlertState currentState = alertStates.get(ruleId);
        AlertState escalatedState = currentState.toBuilder()
            .severity(escalationRule.getSeverity())
            .escalatedAt(Instant.now())
            .build();
            
        alertStates.put(ruleId, escalatedState);
        
        Alert escalatedAlert = Alert.builder()
            .id(UUID.randomUUID().toString())
            .ruleId(ruleId)
            .name(rule.getName() + " (Escalated)")
            .severity(escalationRule.getSeverity())
            .message("ESCALATED: " + generateAlertMessage(rule, metrics))
            .triggeredAt(escalatedState.getEscalatedAt())
            .build();
            
        notificationService.sendAlert(escalatedAlert);
        
        logger.error("Alert escalated: {} - {}", rule.getName(), escalatedAlert.getMessage());
    }
}
```

## 🎯 下一步学习

完成性能优化与监控学习后，建议继续深入：

1. **[第五篇：LangChain4j MCP 测试策略与质量保证](05-langchain4j-mcp-testing.md)**
2. **[LangChain4j MCP 技术总结与最佳实践](langchain4j-mcp-summary.md)**
