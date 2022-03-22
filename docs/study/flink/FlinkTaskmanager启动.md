## YarnContainerEventHandler
实现了AMRMClientAsync.CallbackHandler, NMClientAsync.CallbackHandler
看onContainersAllocated
### onContainersAllocated
```
public void onContainersAllocated(List<Container> containers) {  
    runAsyncWithFatalHandler(  
            () -> {  
                checkInitialized();  
 log.info("Received {} containers.", containers.size());  
  
 for (Map.Entry<Priority, List<Container>> entry :  
                        groupContainerByPriority(containers).entrySet()) {  
                    onContainersOfPriorityAllocated(entry.getKey(), entry.getValue());  
 }  
  
                // if we are waiting for no further containers, we can go to the  
 // regular heartbeat interval if (getNumRequestedNotAllocatedWorkers() <= 0) {  
                    resourceManagerClient.setHeartbeatInterval(yarnHeartbeatIntervalMillis);  
 }  
            });  
}
```

### onContainersOfPriorityAllocated
```
while (containerIterator.hasNext() && pendingContainerRequestIterator.hasNext()) {  
    final Container container = containerIterator.next();  
 final AMRMClient.ContainerRequest pendingRequest =  
            pendingContainerRequestIterator.next();  
 final ResourceID resourceId = getContainerResourceId(container);  
  
 final CompletableFuture<YarnWorkerNode> requestResourceFuture =  
            pendingRequestResourceFutures.poll();  
 Preconditions.checkState(requestResourceFuture != null);  
  
 if (pendingRequestResourceFutures.isEmpty()) {  
        requestResourceFutures.remove(taskExecutorProcessSpec);  
 }  
  
    startTaskExecutorInContainerAsync(  
            container, taskExecutorProcessSpec, resourceId, requestResourceFuture);  
 removeContainerRequest(pendingRequest);  
  
 numAccepted++;  
}
```
### startTaskExecutorInContainerAsync
```
private void startTaskExecutorInContainerAsync(  
        Container container,  
 TaskExecutorProcessSpec taskExecutorProcessSpec,  
 ResourceID resourceId,  
 CompletableFuture<YarnWorkerNode> requestResourceFuture) {  
    final CompletableFuture<ContainerLaunchContext> containerLaunchContextFuture =  
            FutureUtils.supplyAsync(  
                    () ->  
                            createTaskExecutorLaunchContext(  
                                    resourceId,  
 container.getNodeId().getHost(),  
 taskExecutorProcessSpec),  
 getIoExecutor());  
  
 FutureUtils.assertNoException(  
            containerLaunchContextFuture.handleAsync(  
                    (context, exception) -> {  
                        if (exception == null) {  
                            nodeManagerClient.startContainerAsync(container, context);  
 requestResourceFuture.complete(  
                                    new YarnWorkerNode(container, resourceId));  
 } else {  
                            requestResourceFuture.completeExceptionally(exception);  
 }  
                        return null;  
 },  
 getMainThreadExecutor()));  
}
```
### createTaskExecutorLaunchContext
```
private ContainerLaunchContext createTaskExecutorLaunchContext(  
        ResourceID containerId, String host, TaskExecutorProcessSpec taskExecutorProcessSpec)  
        throws Exception {  
  
    // init the ContainerLaunchContext  
 final String currDir = configuration.getCurrentDir();  
  
 final ContaineredTaskManagerParameters taskManagerParameters =  
            ContaineredTaskManagerParameters.create(flinkConfig, taskExecutorProcessSpec);  
  
 log.info(  
            "TaskExecutor {} will be started on {} with {}.",  
 containerId.getStringWithMetadata(),  
 host,  
 taskExecutorProcessSpec);  
  
 final Configuration taskManagerConfig = BootstrapTools.cloneConfiguration(flinkConfig);  
 taskManagerConfig.set(  
            TaskManagerOptions.TASK_MANAGER_RESOURCE_ID, containerId.getResourceIdString());  
 taskManagerConfig.set(  
            TaskManagerOptionsInternal.TASK_MANAGER_RESOURCE_ID_METADATA,  
 containerId.getMetadata());  
  
 final String taskManagerDynamicProperties =  
            BootstrapTools.getDynamicPropertiesAsString(flinkClientConfig, taskManagerConfig);  
  
 log.debug("TaskManager configuration: {}", taskManagerConfig);  
  
 final ContainerLaunchContext taskExecutorLaunchContext =  
            Utils.createTaskExecutorContext(  
                    flinkConfig,  
 yarnConfig,  
 configuration,  
 taskManagerParameters,  
 taskManagerDynamicProperties,  
 currDir,  
 YarnTaskExecutorRunner.class,  
 log);  
  
 taskExecutorLaunchContext.getEnvironment().put(ENV_FLINK_NODE_ID, host);  
 return taskExecutorLaunchContext;  
}
```

## YarnTaskExecutorRunner 
Utils.createTaskExecutorContext和BootstrapTools.getTaskManagerShellCommand构建启动命令，MainClass是YarnTaskExecutorRunner
### main()
```
public static void main(String[] args) {  
    EnvironmentInformation.logEnvironmentInfo(LOG, "YARN TaskExecutor runner", args);  
 SignalHandler.register(LOG);  
 JvmShutdownSafeguard.installAsShutdownHook(LOG);  
  
 runTaskManagerSecurely(args);  
}

```
### runTaskManagerSecurely
```
private static void runTaskManagerSecurely(String[] args) {  
    Configuration configuration = null;  
  
 try {  
        LOG.debug("All environment variables: {}", ENV);  
  
 final String currDir = ENV.get(Environment.PWD.key());  
 LOG.info("Current working Directory: {}", currDir);  
  
 configuration = TaskManagerRunner.loadConfiguration(args);  
 setupAndModifyConfiguration(configuration, currDir, ENV);  
 } catch (Throwable t) {  
        LOG.error("YARN TaskManager initialization failed.", t);  
 System.exit(INIT_ERROR_EXIT_CODE);  
 }  
  
    TaskManagerRunner.runTaskManagerProcessSecurely(Preconditions.checkNotNull(configuration));  
}
```
## TaskManagerRunner
### runTaskManagerProcessSecurely
```
try {  
    SecurityUtils.install(new SecurityConfiguration(configuration));  
  
 exitCode =  
            SecurityUtils.getInstalledContext()  
                    .runSecured(() -> runTaskManager(configuration, pluginManager));  
} catch (Throwable t) {}

```
### runTaskManager
```
try {  
    taskManagerRunner =  
            new TaskManagerRunner(  
                    configuration,  
 pluginManager,  
 TaskManagerRunner::createTaskExecutorService);  
 taskManagerRunner.start();  
} catch (Exception exception) {}
```
