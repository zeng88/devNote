quartz-spring_demo
基于 Spring + Quartz 的动态定时任务管理 Demo | 作者: snailxr | Java 7 / WAR 部署

一句话总结：把 Quartz 定时任务的配置（cron 表达式、执行目标）存到 MySQL 数据库里，应用启动时自动加载到 Quartz 调度器，运行时通过 Web 页面动态增删改查、启停、立即执行——无需重启应用。
技术栈
分类	技术	版本
框架	Spring (core / context / webmvc / tx / jdbc)	3.2.2.RELEASE
调度	Quartz + quartz-jobs	2.2.1
ORM	MyBatis + mybatis-spring	3.2.2 / 1.2.0
数据库	MySQL + Druid 连接池	5.1.24 / 0.2.20
前端	JSP + jQuery	1.9.1
构建	Maven (war 包)	JDK 1.7
日志	Log4j + SLF4J	1.2.16 / 1.5.2
核心设计思路
传统 Quartz 用法是在代码或 XML 里硬编码任务配置，修改 cron 表达式需要改代码重新部署。这个 Demo 的核心改进是数据库驱动调度：

任务配置（名称、分组、cron 表达式、目标类/方法、是否并发）存在 MySQL task_schedule_job 表
应用启动时 @PostConstruct init() 从数据库加载所有任务，注册到 Quartz Scheduler
通过反射 TaskUtils.invokMethod() 动态调用目标类的方法——支持 Spring Bean 和普通类两种方式
Web 页面操作（添加/启停/改 cron/立即执行）实时同步到 Quartz 调度器，无需重启
项目结构
src/main/java/com/snailxr/base/ ├── task/ — 定时任务核心 │ ├── domain/ScheduleJob.java — 任务实体 (jobId, cron, beanClass, springId, methodName...) │ ├── service/JobTaskService.java — 核心：Quartz 调度操作 + DB 持久化 │ ├── controller/JobTaskController — Web 接口 (taskList/add/changeStatus/updateCron) │ ├── dao/ScheduleJobMapper.java — MyBatis Mapper 接口 │ ├── dao/mapper/ScheduleJobMapper.xml — SQL 映射 │ ├── QuartzJobFactory.java — 无状态 Job (允许并发) │ ├── QuartzJobFactoryDisallowConcurrent.java — @DisallowConcurrentExecution (禁止重叠) │ ├── TaskUtils.java — 反射调用工具 (核心) │ └── TaskTest.java — 示例任务 (run / run1 方法) ├── support/ │ ├── RetObj.java — 统一返回对象 (flag + msg) │ └── spring/SpringUtils.java — Spring BeanFactory 工具 (通过 bean name 获取实例) src/main/resources/ ├── spring.xml — Spring 主配置 (数据源/MyBatis/事务/SchedulerFactoryBean) ├── spring-mvc.xml — Spring MVC 配置 (扫描 Controller/JSP 视图/文件上传) ├── config.properties — 数据库连接配置 └── MyBatisConfiguration.xml — MyBatis 全局配置 src/main/webapp/ ├── WEB-INF/web.xml — Web 应用部署描述符 ├── WEB-INF/base/task/taskList.jsp — 任务管理页面 ├── sql/quartz-demo.sql — 建表 + 示例数据 └── scripts/jquery-1.9.1.min.js
关键文件解析
ScheduleJob.java — 任务实体
每个定时任务的完整描述。核心字段：

字段	说明
jobName / jobGroup	任务名称和分组（联合唯一键）
cronExpression	Quartz cron 表达式，如 0/5 * * * * ?
beanClass	目标类全限定名（反射 newInstance 调用）
springId	Spring Bean 名称（优先于 beanClass）
methodName	要调用的方法名（无参方法）
isConcurrent	"1"=禁止并发, "0"=允许并发
jobStatus	"1"=运行中, "0"=已停止
TaskUtils.invokMethod() — 反射调用核心
Quartz 触发时，JobFactory 取出 ScheduleJob 对象，通过反射找到目标类和方法并执行：

// 1. 优先用 springId 从 Spring 容器获取 Bean
if (springId 不为空) {
    object = SpringUtils.getBean(springId);
}
// 2. 否则用 beanClass 反射创建实例
else if (beanClass 不为空) {
    clazz = Class.forName(beanClass);
    object = clazz.newInstance();
}
// 3. 反射调用指定方法
method = object.getClass().getDeclaredMethod(methodName);
method.invoke(object);
JobTaskService — 调度管理核心
方法	功能
init()	@PostConstruct — 启动时从 DB 加载所有任务注册到 Quartz
addJob()	创建 JobDetail + CronTrigger，注册到 Scheduler
changeStatus()	启动/停止任务（start→addJob, stop→deleteJob）
updateCron()	更新 cron 表达式，运行中的任务实时 rescheduleJob
pauseJob / resumeJob	暂停/恢复
runAJobNow()	立即触发一次执行
getAllJob / getRunningJob	从 Quartz Scheduler 查询运行时状态
数据库表 task_schedule_job
CREATE TABLE task_schedule_job (
  job_id        BIGINT AUTO_INCREMENT PRIMARY KEY,
  job_name      VARCHAR(255),          -- 任务名称
  job_group     VARCHAR(255),          -- 任务分组
  job_status    VARCHAR(255),          -- 0=停止 1=运行
  cron_expression VARCHAR(255) NOT NULL, -- cron 表达式
  bean_class    VARCHAR(255),          -- 目标类全限定名
  is_concurrent VARCHAR(255),          -- 1=禁止并发 0=允许
  spring_id     VARCHAR(255),          -- Spring Bean 名称
  method_name   VARCHAR(255) NOT NULL, -- 执行方法名
  UNIQUE KEY name_group (job_name, job_group)
);
Web 接口
URL	功能
/task/taskList.htm	任务列表页面（JSP 渲染）
/task/add.htm	添加任务（校验 cron + 目标类/方法 → 存 DB）
/task/changeJobStatus.htm	启停任务（cmd=start/stop）
/task/updateCron.htm	修改 cron 表达式
两种 Job 执行模式
1. QuartzJobFactory（允许并发）

实现 Job 接口，每次 cron 触发都会新开一个 Job 实例执行。如果上一次还没执行完，下一次照常触发——适合轻量快速任务。

2. QuartzJobFactoryDisallowConcurrentExecution（禁止并发）

加了 @DisallowConcurrentExecution 注解，Quartz 保证同一个 Job 上一次执行完之前不会触发下一次——适合耗时较长、不能重叠执行的任务（如数据同步、批处理）。

选择哪种模式由数据库 is_concurrent 字段控制："1" 禁止并发，"0" 允许并发。注册 Job 时根据该字段选择不同的 JobFactory 类。

如何运行
# 1. 创建数据库并导入 SQL
mysql -u root -p -e "CREATE DATABASE \`quartz-demo\`"
mysql -u root -p quartz-demo < src/main/webapp/sql/quartz-demo.sql

# 2. 修改数据库连接 (如需要)
# src/main/resources/config.properties
# jdbc_url=jdbc:mysql://localhost:3306/quartz-demo
# jdbc_username=root
# jdbc_password=root

# 3. Maven 打包
mvn clean package

# 4. 部署到 Tomcat (或将 war 包放入 webapps)
cp target/quartz-spring_demo.war $CATALINA_HOME/webapps/

# 5. 启动 Tomcat，访问
# http://localhost:8080/quartz-spring_demo/task/taskList.htm
注意事项
项目使用 Spring 3.2 + JDK 1.7，属于较老技术栈，pom.xml 中引用了内部 Nexus 仓库（已失效），需要确保依赖能从 Maven Central 获取
Quartz 2.2.1 需要 Spring 3.1+ 才支持，pom.xml 注释中有说明
反射调用只支持无参方法（getDeclaredMethod(methodName)）
任务目标类两种方式二选一：springId（推荐，可注入依赖）或 beanClass（每次 new 新实例，无法注入）
没有做 Quartz 持久化到数据库（JobStore TX），任务状态重启后会从自定义表重新加载
