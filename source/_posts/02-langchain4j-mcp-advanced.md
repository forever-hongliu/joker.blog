---
title: 02-LangChain4j MCP 高级特性与工具开发
date: 2025-07-25 12:10:51
tags:
  - MCP
  - Langchain4j
  - Java
  - Advanced
categories: MCP
cover: https://cdn.pixabay.com/photo/2019/10/19/14/46/tech-4561577_640.jpg
---


# LangChain4j MCP 高级特性与工具开发

> **LangChain4j MCP 系列第二篇** - 深入探索 LangChain4j MCP 的高级特性、自定义工具开发和企业级应用实践

## 📋 目录

- [高级 MCP 客户端配置](#高级-mcp-客户端配置)
- [自定义工具开发](#自定义工具开发)
- [多客户端管理策略](#多客户端管理策略)
- [异步与并发处理](#异步与并发处理)
- [工具链编排](#工具链编排)
- [企业级集成模式](#企业级集成模式)

## ⚙️ 高级 MCP 客户端配置

### 动态客户端工厂

```java
@Component
public class DynamicMcpClientFactory {
    private final Map<String, McpClientTemplate> clientTemplates = new ConcurrentHashMap<>();
    private final Map<String, McpClient> activeClients = new ConcurrentHashMap<>();
    
    @Data
    public static class McpClientTemplate {
        private String name;
        private TransportType transportType;
        private Map<String, Object> transportConfig;
        private Duration toolExecutionTimeout;
        private String protocolVersion;
        private boolean autoReconnect;
        private int maxRetries;
    }
    
    public McpClient createClient(String templateName, Map<String, Object> runtimeConfig) {
        McpClientTemplate template = clientTemplates.get(templateName);
        if (template == null) {
            throw new IllegalArgumentException("Unknown client template: " + templateName);
        }
        
        DefaultMcpClient.Builder builder = new DefaultMcpClient.Builder()
            .clientName(template.getName())
            .protocolVersion(template.getProtocolVersion())
            .toolExecutionTimeout(template.getToolExecutionTimeout());
            
        // 合并模板配置和运行时配置
        Map<String, Object> finalConfig = new HashMap<>(template.getTransportConfig());
        finalConfig.putAll(runtimeConfig);
        
        // 创建传输层
        McpTransport transport = createTransport(template.getTransportType(), finalConfig);
        
        // 添加高级特性
        if (template.isAutoReconnect()) {
            transport = new ReconnectingMcpTransport(transport, template.getMaxRetries());
        }
        
        builder.transport(transport);
        
        McpClient client = builder.build();
        activeClients.put(templateName + "_" + System.currentTimeMillis(), client);
        
        return client;
    }
    
    private McpTransport createTransport(TransportType type, Map<String, Object> config) {
        return switch (type) {
            case STDIO -> createStdioTransport(config);
            case HTTP -> createHttpTransport(config);
            case WEBSOCKET -> createWebSocketTransport(config);
        };
    }
}
```

### 连接池管理

```java
@Component
public class McpClientPool {
    private final BlockingQueue<McpClient> availableClients;
    private final Set<McpClient> allClients;
    private final McpClientFactory clientFactory;
    private final int maxPoolSize;
    private final int minPoolSize;
    
    public McpClientPool(McpClientFactory clientFactory, 
                        @Value("${mcp.pool.max-size:10}") int maxPoolSize,
                        @Value("${mcp.pool.min-size:2}") int minPoolSize) {
        this.clientFactory = clientFactory;
        this.maxPoolSize = maxPoolSize;
        this.minPoolSize = minPoolSize;
        this.availableClients = new LinkedBlockingQueue<>(maxPoolSize);
        this.allClients = ConcurrentHashMap.newKeySet();
        
        initializePool();
    }
    
    public McpClient borrowClient() throws InterruptedException {
        McpClient client = availableClients.poll(5, TimeUnit.SECONDS);
        if (client == null) {
            throw new RuntimeException("No available MCP client in pool");
        }
        
        // 健康检查
        if (!isClientHealthy(client)) {
            // 替换不健康的客户端
            replaceClient(client);
            return borrowClient();
        }
        
        return client;
    }
    
    public void returnClient(McpClient client) {
        if (allClients.contains(client) && isClientHealthy(client)) {
            availableClients.offer(client);
        } else {
            // 客户端已损坏，创建新的替换
            replaceClient(client);
        }
    }
    
    private boolean isClientHealthy(McpClient client) {
        try {
            // 执行简单的健康检查
            client.listTools().get(1, TimeUnit.SECONDS);
            return true;
        } catch (Exception e) {
            logger.warn("Client health check failed", e);
            return false;
        }
    }
    
    @Scheduled(fixedRate = 30000) // 每30秒检查一次
    public void maintainPool() {
        // 确保最小连接数
        while (availableClients.size() < minPoolSize) {
            try {
                McpClient newClient = clientFactory.createClient();
                allClients.add(newClient);
                availableClients.offer(newClient);
            } catch (Exception e) {
                logger.error("Failed to create new client for pool maintenance", e);
                break;
            }
        }
        
        // 清理过期或不健康的连接
        cleanupUnhealthyClients();
    }
}
```

### 智能重连机制

```java
public class ReconnectingMcpTransport implements McpTransport {
    private final McpTransport delegate;
    private final int maxRetries;
    private final Duration initialDelay;
    private final Duration maxDelay;
    private final double backoffMultiplier;
    
    private volatile boolean connected = false;
    private final AtomicInteger retryCount = new AtomicInteger(0);
    
    public ReconnectingMcpTransport(McpTransport delegate, int maxRetries) {
        this.delegate = delegate;
        this.maxRetries = maxRetries;
        this.initialDelay = Duration.ofSeconds(1);
        this.maxDelay = Duration.ofMinutes(5);
        this.backoffMultiplier = 2.0;
    }
    
    @Override
    public CompletableFuture<String> send(String message) {
        if (!connected) {
            return reconnectAndSend(message);
        }
        
        return delegate.send(message)
            .exceptionallyCompose(throwable -> {
                logger.warn("Send failed, attempting reconnection", throwable);
                connected = false;
                return reconnectAndSend(message);
            });
    }
    
    private CompletableFuture<String> reconnectAndSend(String message) {
        return reconnect()
            .thenCompose(v -> delegate.send(message));
    }
    
    private CompletableFuture<Void> reconnect() {
        return CompletableFuture.runAsync(() -> {
            int attempts = retryCount.get();
            
            while (attempts < maxRetries && !connected) {
                try {
                    // 计算退避延迟
                    Duration delay = calculateBackoffDelay(attempts);
                    Thread.sleep(delay.toMillis());
                    
                    // 尝试重连
                    delegate.close().get(5, TimeUnit.SECONDS);
                    // 重新初始化连接
                    initializeConnection();
                    
                    connected = true;
                    retryCount.set(0);
                    logger.info("Successfully reconnected after {} attempts", attempts + 1);
                    return;
                    
                } catch (Exception e) {
                    attempts++;
                    retryCount.set(attempts);
                    logger.warn("Reconnection attempt {} failed", attempts, e);
                }
            }
            
            if (!connected) {
                throw new RuntimeException("Failed to reconnect after " + maxRetries + " attempts");
            }
        });
    }
    
    private Duration calculateBackoffDelay(int attempt) {
        long delayMs = (long) (initialDelay.toMillis() * Math.pow(backoffMultiplier, attempt));
        return Duration.ofMillis(Math.min(delayMs, maxDelay.toMillis()));
    }
}
```

## 🛠️ 自定义工具开发

### 工具接口设计

```java
public interface CustomMcpTool {
    String getName();
    String getDescription();
    JsonSchema getInputSchema();
    CompletableFuture<ToolExecutionResult> execute(Map<String, Object> arguments);
    
    // 可选的元数据
    default Map<String, Object> getMetadata() {
        return Collections.emptyMap();
    }
    
    // 工具分类
    default String getCategory() {
        return "general";
    }
    
    // 权限要求
    default Set<String> getRequiredPermissions() {
        return Collections.emptySet();
    }
}
```

### 数据库操作工具示例

```java
@Component
public class DatabaseQueryTool implements CustomMcpTool {
    private final JdbcTemplate jdbcTemplate;
    private final SqlValidator sqlValidator;
    
    @Override
    public String getName() {
        return "database_query";
    }
    
    @Override
    public String getDescription() {
        return "Execute SQL queries against the database with safety checks";
    }
    
    @Override
    public JsonSchema getInputSchema() {
        return JsonSchema.builder()
            .type("object")
            .properties(Map.of(
                "sql", JsonSchema.builder()
                    .type("string")
                    .description("SQL query to execute")
                    .build(),
                "limit", JsonSchema.builder()
                    .type("integer")
                    .description("Maximum number of rows to return")
                    .minimum(1)
                    .maximum(1000)
                    .defaultValue(100)
                    .build(),
                "timeout", JsonSchema.builder()
                    .type("integer")
                    .description("Query timeout in seconds")
                    .minimum(1)
                    .maximum(30)
                    .defaultValue(10)
                    .build()
            ))
            .required(List.of("sql"))
            .build();
    }
    
    @Override
    public CompletableFuture<ToolExecutionResult> execute(Map<String, Object> arguments) {
        return CompletableFuture.supplyAsync(() -> {
            try {
                String sql = (String) arguments.get("sql");
                Integer limit = (Integer) arguments.getOrDefault("limit", 100);
                Integer timeout = (Integer) arguments.getOrDefault("timeout", 10);
                
                // SQL 安全验证
                ValidationResult validation = sqlValidator.validate(sql);
                if (!validation.isValid()) {
                    return ToolExecutionResult.builder()
                        .isError(true)
                        .content(List.of(TextContent.from("SQL validation failed: " + validation.getError())))
                        .build();
                }
                
                // 添加 LIMIT 子句（如果没有）
                String safeSql = addLimitClause(sql, limit);
                
                // 执行查询
                List<Map<String, Object>> results = executeWithTimeout(safeSql, timeout);
                
                // 格式化结果
                String formattedResults = formatResults(results);
                
                return ToolExecutionResult.builder()
                    .content(List.of(TextContent.from(formattedResults)))
                    .build();
                    
            } catch (Exception e) {
                logger.error("Database query execution failed", e);
                return ToolExecutionResult.builder()
                    .isError(true)
                    .content(List.of(TextContent.from("Query execution failed: " + e.getMessage())))
                    .build();
            }
        });
    }
    
    private List<Map<String, Object>> executeWithTimeout(String sql, int timeoutSeconds) {
        return jdbcTemplate.execute((Connection conn) -> {
            try (Statement stmt = conn.createStatement()) {
                stmt.setQueryTimeout(timeoutSeconds);
                try (ResultSet rs = stmt.executeQuery(sql)) {
                    return extractResults(rs);
                }
            }
        });
    }
    
    @Override
    public Set<String> getRequiredPermissions() {
        return Set.of("database:read");
    }
}
```

### 文件操作工具

```java
@Component
public class FileOperationTool implements CustomMcpTool {
    private final FileSystemService fileSystemService;
    private final SecurityManager securityManager;
    
    @Override
    public String getName() {
        return "file_operations";
    }
    
    @Override
    public String getDescription() {
        return "Perform safe file operations like read, write, list, and search";
    }
    
    @Override
    public JsonSchema getInputSchema() {
        return JsonSchema.builder()
            .type("object")
            .properties(Map.of(
                "operation", JsonSchema.builder()
                    .type("string")
                    .enumValues(List.of("read", "write", "list", "search", "delete"))
                    .description("File operation to perform")
                    .build(),
                "path", JsonSchema.builder()
                    .type("string")
                    .description("File or directory path")
                    .build(),
                "content", JsonSchema.builder()
                    .type("string")
                    .description("Content to write (for write operation)")
                    .build(),
                "pattern", JsonSchema.builder()
                    .type("string")
                    .description("Search pattern (for search operation)")
                    .build(),
                "recursive", JsonSchema.builder()
                    .type("boolean")
                    .description("Whether to search recursively")
                    .defaultValue(false)
                    .build()
            ))
            .required(List.of("operation", "path"))
            .build();
    }
    
    @Override
    public CompletableFuture<ToolExecutionResult> execute(Map<String, Object> arguments) {
        return CompletableFuture.supplyAsync(() -> {
            try {
                String operation = (String) arguments.get("operation");
                String path = (String) arguments.get("path");
                
                // 安全检查
                if (!securityManager.isPathAllowed(path)) {
                    return createErrorResult("Access denied: Path not allowed");
                }
                
                return switch (operation) {
                    case "read" -> handleRead(path);
                    case "write" -> handleWrite(path, (String) arguments.get("content"));
                    case "list" -> handleList(path);
                    case "search" -> handleSearch(path, (String) arguments.get("pattern"), 
                                                 (Boolean) arguments.getOrDefault("recursive", false));
                    case "delete" -> handleDelete(path);
                    default -> createErrorResult("Unknown operation: " + operation);
                };
                
            } catch (Exception e) {
                logger.error("File operation failed", e);
                return createErrorResult("Operation failed: " + e.getMessage());
            }
        });
    }
    
    private ToolExecutionResult handleRead(String path) throws IOException {
        String content = fileSystemService.readFile(path);
        return ToolExecutionResult.builder()
            .content(List.of(TextContent.from(content)))
            .build();
    }
    
    private ToolExecutionResult handleWrite(String path, String content) throws IOException {
        fileSystemService.writeFile(path, content);
        return ToolExecutionResult.builder()
            .content(List.of(TextContent.from("File written successfully: " + path)))
            .build();
    }
    
    private ToolExecutionResult handleSearch(String path, String pattern, boolean recursive) throws IOException {
        List<String> matches = fileSystemService.searchFiles(path, pattern, recursive);
        String result = "Found " + matches.size() + " matches:\n" + String.join("\n", matches);
        return ToolExecutionResult.builder()
            .content(List.of(TextContent.from(result)))
            .build();
    }
}
```

### 工具注册与管理

```java
@Component
public class CustomToolRegistry {
    private final Map<String, CustomMcpTool> tools = new ConcurrentHashMap<>();
    private final PermissionManager permissionManager;
    
    @Autowired
    public CustomToolRegistry(List<CustomMcpTool> customTools, PermissionManager permissionManager) {
        this.permissionManager = permissionManager;
        customTools.forEach(this::registerTool);
    }
    
    public void registerTool(CustomMcpTool tool) {
        tools.put(tool.getName(), tool);
        logger.info("Registered custom tool: {}", tool.getName());
    }
    
    public List<Tool> getAvailableTools(String userId) {
        return tools.values().stream()
            .filter(tool -> hasPermission(userId, tool))
            .map(this::convertToMcpTool)
            .collect(Collectors.toList());
    }
    
    public CompletableFuture<ToolExecutionResult> executeTool(String toolName, 
                                                             Map<String, Object> arguments, 
                                                             String userId) {
        CustomMcpTool tool = tools.get(toolName);
        if (tool == null) {
            return CompletableFuture.completedFuture(
                createErrorResult("Tool not found: " + toolName));
        }
        
        if (!hasPermission(userId, tool)) {
            return CompletableFuture.completedFuture(
                createErrorResult("Permission denied for tool: " + toolName));
        }
        
        // 参数验证
        ValidationResult validation = validateArguments(tool, arguments);
        if (!validation.isValid()) {
            return CompletableFuture.completedFuture(
                createErrorResult("Invalid arguments: " + validation.getError()));
        }
        
        // 执行工具
        return tool.execute(arguments)
            .exceptionally(throwable -> {
                logger.error("Tool execution failed: " + toolName, throwable);
                return createErrorResult("Tool execution failed: " + throwable.getMessage());
            });
    }
    
    private boolean hasPermission(String userId, CustomMcpTool tool) {
        Set<String> requiredPermissions = tool.getRequiredPermissions();
        return requiredPermissions.isEmpty() || 
               permissionManager.hasAllPermissions(userId, requiredPermissions);
    }
}
```

## 🔄 多客户端管理策略

### 客户端路由器

```java
@Component
public class McpClientRouter {
    private final Map<String, McpClient> clients = new ConcurrentHashMap<>();
    private final LoadBalancer loadBalancer;
    private final HealthChecker healthChecker;
    
    public enum RoutingStrategy {
        ROUND_ROBIN,
        LEAST_CONNECTIONS,
        RESPONSE_TIME,
        TOOL_SPECIFIC,
        GEOGRAPHIC
    }
    
    public McpClient selectClient(ToolExecutionRequest request, RoutingStrategy strategy) {
        return switch (strategy) {
            case ROUND_ROBIN -> loadBalancer.selectRoundRobin(getHealthyClients());
            case LEAST_CONNECTIONS -> loadBalancer.selectLeastConnections(getHealthyClients());
            case RESPONSE_TIME -> loadBalancer.selectFastestResponse(getHealthyClients());
            case TOOL_SPECIFIC -> selectByToolCapability(request.getName());
            case GEOGRAPHIC -> selectByGeography(request);
        };
    }
    
    private McpClient selectByToolCapability(String toolName) {
        return clients.values().stream()
            .filter(client -> supportsToolOptimally(client, toolName))
            .min(Comparator.comparing(this::getCurrentLoad))
            .orElse(loadBalancer.selectRoundRobin(getHealthyClients()));
    }
    
    private List<McpClient> getHealthyClients() {
        return clients.values().stream()
            .filter(healthChecker::isHealthy)
            .collect(Collectors.toList());
    }
    
    // 智能故障转移
    public CompletableFuture<ToolExecutionResult> executeWithFailover(
            ToolExecutionRequest request, RoutingStrategy strategy) {
        
        List<McpClient> candidates = getHealthyClients();
        if (candidates.isEmpty()) {
            return CompletableFuture.completedFuture(
                createErrorResult("No healthy clients available"));
        }
        
        return executeWithFailover(request, candidates, 0);
    }
    
    private CompletableFuture<ToolExecutionResult> executeWithFailover(
            ToolExecutionRequest request, List<McpClient> candidates, int attemptIndex) {
        
        if (attemptIndex >= candidates.size()) {
            return CompletableFuture.completedFuture(
                createErrorResult("All clients failed"));
        }
        
        McpClient client = candidates.get(attemptIndex);
        
        return client.executeTool(request)
            .exceptionallyCompose(throwable -> {
                logger.warn("Client {} failed, trying next", client, throwable);
                healthChecker.markUnhealthy(client);
                return executeWithFailover(request, candidates, attemptIndex + 1);
            });
    }
}
```

### 分布式客户端协调

```java
@Component
public class DistributedMcpCoordinator {
    private final RedisTemplate<String, Object> redisTemplate;
    private final MessageChannel coordinationChannel;
    
    // 分布式锁管理
    public <T> CompletableFuture<T> executeWithDistributedLock(
            String lockKey, Duration lockTimeout, Supplier<CompletableFuture<T>> operation) {
        
        return CompletableFuture.supplyAsync(() -> {
            String lockValue = UUID.randomUUID().toString();
            
            try {
                // 获取分布式锁
                Boolean acquired = redisTemplate.opsForValue()
                    .setIfAbsent(lockKey, lockValue, lockTimeout);
                
                if (!acquired) {
                    throw new RuntimeException("Failed to acquire lock: " + lockKey);
                }
                
                // 执行操作
                return operation.get().get();
                
            } catch (Exception e) {
                throw new RuntimeException("Distributed operation failed", e);
            } finally {
                // 释放锁
                releaseLock(lockKey, lockValue);
            }
        });
    }
    
    // 客户端状态同步
    public void broadcastClientStatus(String clientId, ClientStatus status) {
        ClientStatusMessage message = ClientStatusMessage.builder()
            .clientId(clientId)
            .status(status)
            .timestamp(Instant.now())
            .nodeId(getNodeId())
            .build();
            
        coordinationChannel.send(MessageBuilder.withPayload(message).build());
    }
    
    @EventListener
    public void handleClientStatusUpdate(ClientStatusMessage message) {
        if (!message.getNodeId().equals(getNodeId())) {
            // 更新远程客户端状态
            updateRemoteClientStatus(message.getClientId(), message.getStatus());
        }
    }
    
    // 负载信息共享
    public void shareLoadMetrics() {
        LoadMetrics metrics = collectLocalLoadMetrics();
        
        redisTemplate.opsForHash().put(
            "mcp:load:metrics", 
            getNodeId(), 
            metrics
        );
        
        // 设置过期时间
        redisTemplate.expire("mcp:load:metrics", Duration.ofMinutes(5));
    }
    
    public Map<String, LoadMetrics> getClusterLoadMetrics() {
        return redisTemplate.opsForHash()
            .entries("mcp:load:metrics")
            .entrySet()
            .stream()
            .collect(Collectors.toMap(
                entry -> (String) entry.getKey(),
                entry -> (LoadMetrics) entry.getValue()
            ));
    }
}
```

## ⚡ 异步与并发处理

### 并行工具执行

```java
@Service
public class ParallelToolExecutor {
    private final ExecutorService executorService;
    private final Semaphore concurrencyLimiter;
    
    public ParallelToolExecutor(@Value("${mcp.parallel.max-concurrency:10}") int maxConcurrency) {
        this.executorService = Executors.newFixedThreadPool(maxConcurrency);
        this.concurrencyLimiter = new Semaphore(maxConcurrency);
    }
    
    public CompletableFuture<List<ToolExecutionResult>> executeParallel(
            List<ToolExecutionRequest> requests, McpClient client) {
        
        List<CompletableFuture<ToolExecutionResult>> futures = requests.stream()
            .map(request -> executeWithConcurrencyControl(request, client))
            .collect(Collectors.toList());
            
        return CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
            .thenApply(v -> futures.stream()
                .map(CompletableFuture::join)
                .collect(Collectors.toList()));
    }
    
    private CompletableFuture<ToolExecutionResult> executeWithConcurrencyControl(
            ToolExecutionRequest request, McpClient client) {
        
        return CompletableFuture.supplyAsync(() -> {
            try {
                concurrencyLimiter.acquire();
                return client.executeTool(request).get();
            } catch (Exception e) {
                throw new RuntimeException("Tool execution failed", e);
            } finally {
                concurrencyLimiter.release();
            }
        }, executorService);
    }
    
    // 批量执行优化
    public CompletableFuture<List<ToolExecutionResult>> executeBatch(
            List<ToolExecutionRequest> requests, McpClient client, int batchSize) {
        
        List<List<ToolExecutionRequest>> batches = partitionList(requests, batchSize);
        
        return batches.stream()
            .map(batch -> executeParallel(batch, client))
            .reduce(CompletableFuture.completedFuture(new ArrayList<>()),
                (acc, batchResult) -> acc.thenCombine(batchResult, (list1, list2) -> {
                    List<ToolExecutionResult> combined = new ArrayList<>(list1);
                    combined.addAll(list2);
                    return combined;
                }));
    }
}
```

### 响应式流处理

```java
@Service
public class ReactiveToolProcessor {
    
    public Flux<ToolExecutionResult> processToolStream(
            Flux<ToolExecutionRequest> requestStream, McpClient client) {
        
        return requestStream
            .groupBy(ToolExecutionRequest::getName) // 按工具名分组
            .flatMap(groupedFlux -> 
                groupedFlux
                    .buffer(Duration.ofMillis(100), 10) // 批处理
                    .flatMap(batch -> processBatch(batch, client))
            )
            .onErrorResume(this::handleError)
            .doOnNext(this::logResult);
    }
    
    private Flux<ToolExecutionResult> processBatch(
            List<ToolExecutionRequest> batch, McpClient client) {
        
        return Flux.fromIterable(batch)
            .flatMap(request -> 
                Mono.fromFuture(client.executeTool(request))
                    .timeout(Duration.ofSeconds(30))
                    .retry(3)
                    .onErrorReturn(createErrorResult("Execution failed"))
            );
    }
    
    private Flux<ToolExecutionResult> handleError(Throwable error) {
        logger.error("Stream processing error", error);
        return Flux.just(createErrorResult("Stream processing failed: " + error.getMessage()));
    }
    
    // 背压处理
    public Flux<ToolExecutionResult> processWithBackpressure(
            Flux<ToolExecutionRequest> requestStream, McpClient client) {
        
        return requestStream
            .onBackpressureBuffer(1000) // 缓冲区大小
            .limitRate(10) // 限制处理速率
            .flatMap(request -> 
                Mono.fromFuture(client.executeTool(request))
                    .subscribeOn(Schedulers.boundedElastic())
            );
    }
}
```

## 🔗 工具链编排

### 工作流引擎

```java
@Component
public class ToolWorkflowEngine {
    
    @Data
    public static class WorkflowDefinition {
        private String name;
        private List<WorkflowStep> steps;
        private Map<String, Object> globalVariables;
        private Duration timeout;
    }
    
    @Data
    public static class WorkflowStep {
        private String name;
        private String toolName;
        private Map<String, Object> arguments;
        private List<String> dependsOn;
        private String condition; // SpEL expression
        private boolean parallel;
        private int retryCount;
    }
    
    public CompletableFuture<WorkflowResult> executeWorkflow(
            WorkflowDefinition workflow, McpClient client) {
        
        WorkflowContext context = new WorkflowContext(workflow.getGlobalVariables());
        
        return executeSteps(workflow.getSteps(), client, context)
            .orTimeout(workflow.getTimeout().toMillis(), TimeUnit.MILLISECONDS)
            .thenApply(results -> WorkflowResult.builder()
                .workflowName(workflow.getName())
                .stepResults(results)
                .context(context)
                .success(true)
                .build())
            .exceptionally(throwable -> WorkflowResult.builder()
                .workflowName(workflow.getName())
                .success(false)
                .error(throwable.getMessage())
                .build());
    }
    
    private CompletableFuture<Map<String, ToolExecutionResult>> executeSteps(
            List<WorkflowStep> steps, McpClient client, WorkflowContext context) {
        
        // 构建依赖图
        Map<String, Set<String>> dependencies = buildDependencyGraph(steps);
        
        // 拓扑排序
        List<List<WorkflowStep>> executionLevels = topologicalSort(steps, dependencies);
        
        Map<String, ToolExecutionResult> results = new ConcurrentHashMap<>();
        
        return executionLevels.stream()
            .reduce(CompletableFuture.completedFuture(results),
                (acc, level) -> acc.thenCompose(currentResults -> 
                    executeLevel(level, client, context, currentResults)),
                (f1, f2) -> f1.thenCombine(f2, (r1, r2) -> {
                    r1.putAll(r2);
                    return r1;
                }));
    }
    
    private CompletableFuture<Map<String, ToolExecutionResult>> executeLevel(
            List<WorkflowStep> steps, McpClient client, 
            WorkflowContext context, Map<String, ToolExecutionResult> previousResults) {
        
        List<CompletableFuture<Pair<String, ToolExecutionResult>>> futures = steps.stream()
            .filter(step -> evaluateCondition(step.getCondition(), context))
            .map(step -> executeStepWithRetry(step, client, context)
                .thenApply(result -> Pair.of(step.getName(), result)))
            .collect(Collectors.toList());
            
        return CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
            .thenApply(v -> {
                Map<String, ToolExecutionResult> levelResults = new HashMap<>(previousResults);
                futures.forEach(future -> {
                    Pair<String, ToolExecutionResult> pair = future.join();
                    levelResults.put(pair.getFirst(), pair.getSecond());
                    context.setStepResult(pair.getFirst(), pair.getSecond());
                });
                return levelResults;
            });
    }
    
    private CompletableFuture<ToolExecutionResult> executeStepWithRetry(
            WorkflowStep step, McpClient client, WorkflowContext context) {
        
        return executeStepOnce(step, client, context)
            .exceptionallyCompose(throwable -> {
                if (step.getRetryCount() > 0) {
                    logger.warn("Step {} failed, retrying", step.getName(), throwable);
                    WorkflowStep retryStep = step.toBuilder()
                        .retryCount(step.getRetryCount() - 1)
                        .build();
                    return executeStepWithRetry(retryStep, client, context);
                } else {
                    return CompletableFuture.completedFuture(
                        createErrorResult("Step failed after retries: " + throwable.getMessage()));
                }
            });
    }
}
```

## 🏢 企业级集成模式

### Spring Boot 自动配置

```java
@Configuration
@ConditionalOnClass(McpClient.class)
@EnableConfigurationProperties(McpProperties.class)
public class McpAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public McpClientFactory mcpClientFactory(McpProperties properties) {
        return new DefaultMcpClientFactory(properties);
    }
    
    @Bean
    @ConditionalOnMissingBean
    public McpClientPool mcpClientPool(McpClientFactory factory, McpProperties properties) {
        return new McpClientPool(factory, properties.getPool());
    }
    
    @Bean
    @ConditionalOnProperty(name = "mcp.workflow.enabled", havingValue = "true")
    public ToolWorkflowEngine workflowEngine() {
        return new ToolWorkflowEngine();
    }
    
    @Bean
    @ConditionalOnProperty(name = "mcp.monitoring.enabled", havingValue = "true")
    public McpMetricsCollector metricsCollector(MeterRegistry meterRegistry) {
        return new McpMetricsCollector(meterRegistry);
    }
}
```

### 监控与可观测性

```java
@Component
public class McpObservabilityManager {
    private final MeterRegistry meterRegistry;
    private final TracingContext tracingContext;
    
    public <T> CompletableFuture<T> withObservability(
            String operationName, Supplier<CompletableFuture<T>> operation) {
        
        Span span = tracingContext.nextSpan().name(operationName).start();
        Timer.Sample sample = Timer.start(meterRegistry);
        
        return operation.get()
            .whenComplete((result, throwable) -> {
                sample.stop(Timer.builder("mcp.operation.duration")
                    .tag("operation", operationName)
                    .tag("status", throwable == null ? "success" : "error")
                    .register(meterRegistry));
                    
                if (throwable != null) {
                    span.tag("error", throwable.getMessage());
                    meterRegistry.counter("mcp.operation.errors",
                        "operation", operationName).increment();
                }
                
                span.end();
            });
    }
}
```

## 🎯 下一步学习

完成高级特性学习后，建议继续深入：

1. **[第三篇：LangChain4j MCP 生产环境实践](/posts/03-langchain4j-mcp-production/)**
2. **[第四篇：LangChain4j MCP 性能优化与监控](/posts/04-langchain4j-mcp-performance/)**
3. **[第五篇：LangChain4j MCP 测试策略与质量保证](/posts/05-langchain4j-mcp-testing/)**
4. **[终篇: LangChain4j MCP 技术总结与最佳实践](/posts/06-langchain4j-mcp-summary/)**
