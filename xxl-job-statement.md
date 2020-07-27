## 2.1.2版本使用说明

##### 执行器端的任务创建

 2.1.2版本之前如下方式配置(从2.1.2版本开始 @JobHandler 注解已过期不建议使用该方式配置任务)
 ```java
@Component
@JobHandler("RecoverPunishJob")
@Slf4j
public class RecoverPunishJob extends IJobHandler {

    @Autowired
    ITbPunishRecordService punishRecordService;

    @Override
    public ReturnT<String> execute(String param) {
        log.info("========解封用户任务开始========");
        punishRecordService.relievePunish();
        log.info("========解封用户任务结束========");
        return SUCCESS;
    }

}

 ```
建议使用 @XxlJob 注解方式(支持基于方法的任务开发方式,支持单个类中开发多个任务方法，进行类复用)
1. 在Spring Bean实例中，开发Job方法，方式格式要求为 "public ReturnT<String> execute(String param)"
2. 为Job方法添加注解 "@XxlJob(value="自定义jobhandler名称", init = "JobHandler初始化方法", destroy = "JobHandler销毁方法")"，注解value值对应的是调度中心新建任务的JobHandler属性的值。
3. 执行日志：需要通过 "XxlJobLogger.log" 打印执行日志；
4. 注意 捕获异常时对 InterruptedException 异常单独抛出
5. 使用XxlJobLogger打印日志,可以在控制台的 执行日志中查看
```java
@Component
public class RecoverPunishJob {

    // 如果出现异常不处理,默认抛给调用者xxl-job去处理
    @XxlJob("OneJob")
    public ReturnT<String> executeJobOne(String param) {
        XxlJobLogger.log("==========xxx任务开始==========");
        // do something
        XxlJobLogger.log("==========xxx任务开始==========");
        return ReturnT.SUCCESS;
    }

    // 如果要自己处理异常,则建议对 InterruptedException 异常单独抛出
    @XxlJob("TwoJob")
    public ReturnT<String> executeJobTwo(String param) {
        XxlJobLogger.log("==========xxx任务开始==========");
        try{
            // do something
        } catch (Exception e) {
            //对 InterruptedException 异常单独抛给调用端处理,否则 [终止任务] 功能无法正常使用
            if (e instanceof InterruptedException) {
                throw e;
            }

            logger.warn("{}", e);
            return ReturnT.FAIL;
        }
        XxlJobLogger.log("==========xxx任务开始==========");
        return ReturnT.SUCCESS;
    }
}

```


#### 调度中心新建调度任务

![1591434707375](doc/images/1591434707375.png)

* JobHandler属性填写任务注解@XxlJob 或者 @JobHandler 中定义的值；

* 路由策略: 在微服务集群环境下,调度器选择调度哪台机器的策略,一般默认第一个

* 运行模式: 一般默认选BEAN就好

* 阻塞处理策略: 因为任务到执行器端是放在队列中的, 当任务较多处理不及时时的处理策略,一般情况下选单机串行

* 任务超时时间: 允许任务延迟执行的最大时间,单位为 秒

* 子任务ID: 可以配置在调度时触发指定的任务(指定JobId,对个用逗号分隔)

* 失败重试次数: 在调度返回失败时,会进行重新触发的次数

* 报警邮件: 任务执行异常时发送提示邮件到填写的邮箱

  

  #### GLUE(Java) 运行模式任务配置:

  任务以源码方式维护在调度中心；该模式的任务实际上是一段继承自IJobHandler的Java类代码并 "groovy" 源码方式维护，它在执行器项目中运行，可使用@Resource/@Autowire注入执行器里中的其他服务；

  

  新增GLUE(Java) 类型任务后做如下图操作:

  ![1591438195500](doc/images/1591438195500.png)

  ![1591439080993](doc/images/1591439080993.png)



#### 调度中心

1. 启动注册监控助手(每30秒执行一次)
   - 移除死掉的执行器(30*3秒内未注册的服务)
   - 刷新自动注册的执行器地址列表-并跟新到xxl_job_group表的address_list字段中

2. 启动调度失败监控助手(每10秒执行一次)
   - 扫描调度日志,找到执行失败的任务日志(按id排序,最多1000条) 进行重新触发
   - 如果当前任务设置了有效的报警邮箱则会发送报警邮件

3. 启动触发线程池(慢触发池最小线程数:100,快触发池最小线程数:200 )
4. 启动日志统计线程(每1分钟执行一次)
   - 统计最近三天的 当天总执行数,正在运行任务数,调度成功任务数,调度失败任务数

5. 启动调度助手()

   1. 启动scheduleThread线程(一个粗略的任务预取线程,每5秒的整数倍时间触发)

   - 根据xxl_job_info表中的trigger_next_time字段查询未来5秒内需要执行的任务,并且一次只取 快触发线程池最大线程数和慢触发线程池最大线程数之和的20倍大小的数量
   - 如果现在时间错过5秒以上,会打印警告日志,并更新下次触发时间
   - 如果现在时间错过5秒以内,则会直接触发调度,并且更新下次触发时间,同时检测下次触发时间是否在未来5秒内,若是则添加到ringData集合,等待ringThread线程调度
   - 如果下次触发时间在未来5秒内,则添加到ringData集合,等待ringThread线程调度

   2.启动ringThread线程(一个更为精细的任务响应线程,每个整秒触发一次)



#### 执行器

1. 初始化JobHandler仓库
   - 扫描带有@JobHandler 注解的类,并添加到jobHandlerRepository中(ConcurrentMap<String, IJobHandler>)
2. 扫描带 @XxlJob 注解的方法, 使用反射的方式转换成 MethodJobHandler(继承IJobHandler) ,然后添加到jobHandlerRepository中
3. 刷新GlueFactory 实例, 用来处理GLUE模式的任务
4. 调用XxlJobExecutor中的start() 方法,初始化如下信息
   * 初始化日志路径
   * 初始化AdminBizClient(连接调度中心的对象),使用配置中的xxl.job.admin.addresses和xxl.job.accessToken来创建,并添加到adminBizList集合中
   * 启动日志文件清理线程, 按xxl.job.executor.logretentiondays配置清理过期的日志,若配置的值小于3 则不会启动清理线程
   * ![image](doc/images/20200609092413449.png)
   * 初始化触发回调线程,会监控回调阻塞队列callBackQueue,实时执行回调任务
   * (后面添加)初始化最大失败回调次数,取xxl.job.retry.callback.max-times配置,取值[0,1000)之间,超出范围默认15
   * 初始化Rpc提供程序,取xxl.job.executor.ip, xxl.job.executor.port, xxl.job.executor.appname 和 xxl.job.accessToken

