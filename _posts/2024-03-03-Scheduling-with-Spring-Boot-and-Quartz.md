---
layout: post
title: "Scheduling with Spring Boot and Quartz"
description: "Scheduling with Spring Boot and Quartz."
date: 2024-03-03
feature_image: images/posts/2024-03-03-Scheduling-with-Spring-Boot-and-Quartz-logo.jpg
tags: [java, spring]
---
Sometimes we need a mechanism that allows tasks to be executed automatically at specific times or intervals, without requiring manual intervention. In this article I'll try to explain how we can do it with **Spring Boot** and **Quartz** scheduler. I should also warn you right away that the purpose of this article is more about the structure of an application using these technologies rather than specific scheduler settings, since everyone will have their own.

<!--more-->

Source code you can find on [github](https://github.com/AndreyStarikov/spring-boot-quartz-example).

Let's start by scheduling tasks to be run at a certain time period or certain interval in a single-node application. For such cases, the native Spring scheduler is more than enough. As we use Spring Boot, the first step is to add this library to the project dependencies.
```kotlin
plugins {
    id("org.springframework.boot") version "3.2.2"
    id("io.spring.dependency-management") version "1.1.4"
    id("java")
} 
  
group = "com.example.spring-boot-scheduler"  
version = "0.0.1-snapshot"  
  
springBoot {  
    mainClass.set("com.example.springbootscheduler.Application")  
}  
  
repositories {  
    mavenCentral()  
}  
  
java {  
    toolchain {  
        languageVersion.set(JavaLanguageVersion.of(17))  
    }  
}  
  
dependencies {  
    implementation("org.springframework.boot:spring-boot-starter")  
}
```
To enable scheduling in Spring application we must put `@EnableScheduling` annotation on Application class.
```java
@EnableScheduling  
@SpringBootApplication  
public class Application {  
  public static void main(String[] args) {  
    SpringApplication.run(Application.class, args);  
  }  
}
```
Further, to execute a task on a schedule, we need the Spring's `@Scheduled` annotation. This annotation marks a method to be scheduled. The annotated method must expect no arguments and typically has a void return type. Although it is possible to return a value, the scheduler will ignore it. When using this annotation, one of the attributes —`cron`, `fixedDelay`, or `fixedRate`—must be specified. Here's a quick overview:
- `fixedRate` specifies the interval between method invocations, measured from the start time of each invocation.
- `fixedDelay` sets the interval between invocations, measured from the completion of the task.
- `cron` is used for more sophisticated task scheduling with cron expression.

As an example of the task, we will simply output a log message with the current time every 5 seconds. Let's look at the code of this task:

```java
@Component  
public class Task {  
  
  private static final Logger log = LoggerFactory.getLogger(Task.class);  
  
  private static final DateTimeFormatter dtf = DateTimeFormatter.ofPattern("HH:mm:ss");  
  
  @Scheduled(fixedRate = 5000)  
  public void task() {  
    log.info("Current time is {}", LocalTime.now().format(dtf));  
  }  
}
```
Now let's run and test our application. As we use gradle, we will use `./gradlew bootRun` command. In the terminal, we can see the output of our app:
```
Current time is 17:28:08
Current time is 17:28:13
Current time is 17:28:18
...
```
It works. Every 5 seconds we have message in logs with current time.

What's next? Now, let's consider a scenario where we need to dynamically schedule tasks with specific arguments while our application is running. In such cases, it is better to use Quartz, a popular open-source job scheduling library. To integrate Quartz into our project, we will add the 'quartz-spring-boot-starter' dependency.
```
implementation("org.springframework.boot:spring-boot-starter-quartz")
```
After that we need to add configuration to `application.yml`. For simplicity of this case we will use in-memory job storage.
```yml
spring:  
  application:  
    name: quartz-single-node-scheduler  
  quartz:  
    job-store-type: memory  
    properties:  
      org:  
        quartz:  
          scheduler:  
            instanceName: ${spring.application.name}-scheduler  
            skipUpdateCheck: true  
          threadPool:  
            class: org.quartz.simpl.SimpleThreadPool  
            threadCount: 1
```
All properties you can find in official documentation: http://www.quartz-scheduler.org/documentation/quartz-2.3.0/configuration/ConfigMain.html

The main components of Quartz are `Job`, `JobDetail`, `Trigger`, and `Scheduler`.
- `Job` defines the action to schedule.
- `JobDetail` describes the parameters of the job.
- `Trigger` specifies when the job should be executed.
- `Scheduler` orchestrates the execution of jobs.
  To schedule a job, we need to describe this job, define the conditions for starting it, and then submit these descriptions to the scheduler. Let's try to do it!

As an example, we will create a job via REST controller that will output a message to logs.
First, since we no longer need the Spring native scheduler, we can remove the `@EnableScheduling` annotation from the `Application` class.
Next, we'll define our first `Job`. A Quartz job must implement the `org.quartz.Job` interface. We'll create a class and implement the `execute()` method of this interface. Within this method, we'll take the `JOB_NAME` parameter of the job and show it in logs. The final structure of our class will look like:
```java
public class JobExample implements Job {  
    
  public static final String JOB_NAME = "JOB_NAME";  
  private static final Logger log = LoggerFactory.getLogger(JobExample.class);  
  
  @Override  
  public void execute(final JobExecutionContext context) {  
    var jobDataMap = context.getJobDetail().getJobDataMap();  
    var jobName = jobDataMap.getString(JOB_NAME);  
    log.info("Job {} is running", jobName);  
  }  
}
```
`JOB_NAME` is a constant string that we will use as a key under which we will store the job name in the `jobDataMap`. The `JobDataMap` is a key-value storage that holds any additional data passed to the job when it was scheduled.

Now let's move on to the rest-controller, where the job will be created on request. The controller code is as follows:
```java
@RestController
public class Controller {

  private static final Logger log = LoggerFactory.getLogger(Controller.class);

  private final SchedulerFactoryBean schedulerFactoryBean;

  public Controller(SchedulerFactoryBean schedulerFactoryBean) {
    this.schedulerFactoryBean = schedulerFactoryBean;
  }

  @GetMapping("/job/{jobName}")
  public void scheduleTask(@PathVariable String jobName) throws SchedulerException {
    log.info("Job {} is scheduling", jobName);
    var jobDetail = JobBuilder.newJob(JobExample.class)
        .usingJobData(JobExample.JOB_NAME, jobName)
        .build();
    var trigger = TriggerBuilder.newTrigger()
        .forJob(jobDetail.getKey().getName())
        .startNow()
        .withSchedule(SimpleScheduleBuilder.simpleSchedule()).build();
    schedulerFactoryBean.getScheduler().scheduleJob(jobDetail, trigger);
  }
}
```
`JobDetail` describes the parameters of the job instance. Here, we specify the job class to use, and also add a job name parameter (`JOB_NAME`) to the `jobDataMap` key-value storage.
As for the `Trigger`, it determines when the job must be executed. While there are multiple options for configuring triggers, for simplicity, let's start the job immediately using the `startNow()` option.

We've now defined the task, specified when and with what parameters it should run. The final step is to submit the job to the scheduler. For our scheduler, we'll use the `SchedulerFactoryBean` bean provided by Spring.
Done. With these configurations in place, after running our application and sending a GET request to `localhost:8080/job/"first job"`, the outcome will be as follows:
```
Job "first job" is scheduling
Job "first job" is running
```

So now we know how to run a statically defined job and how to create and run a job at application runtime. What's next?
In real-world applications, we need job persistence to prevent information loss when the application is restarted. We also need the ability to run jobs on multiple instances of our application without repetition. So next I will give an example of an application that will run both static jobs and jobs created in runtime in a cluster. The application will use **PostgreSQL** to store information about the jobs and also to synchronize between multiple instances of the application. Docker-compose file for Postgres can be found in the [repository](https://github.com/AndreyStarikov/spring-boot-quartz-example/blob/master/quartz-cluster-scheduler/docker-compose.yml). Keep in mind that **Quartz** can also be used with other databases, not only relational ones (**MongoDB** for example). We will use **Liquibase** to initialize tables for Quartz scheduler in the database.

Let's start by adding new dependencies for work with database to our `build.gradle.kts`:
```
implementation("org.springframework.boot:spring-boot-starter-jdbc")  
implementation("org.postgresql:postgresql:42.7.1")  
implementation("org.liquibase:liquibase-core:4.25.1")
```
After that we need to add additional configuration to `application.yml`.
```yaml
spring:
  application:
    name: quartz-cluster-scheduler
  datasource:
    driver-class-name: org.postgresql.Driver
    url: jdbc:postgresql://localhost:5432/quartz_test
#    jdbc-url: jdbc:postgresql://localhost:5432/quartz_test
    username: quartz_user
    password: quartz_password
  liquibase:
    changeLog: "classpath:db/master-changelog.yml"
  quartz:
    job-store-type: jdbc
    jdbc:
      initialize-schema: never
    properties:
      org:
        quartz:
          jobStore:
            class: org.springframework.scheduling.quartz.LocalDataSourceJobStore
            driverDelegateClass: org.quartz.impl.jdbcjobstore.PostgreSQLDelegate
            isClustered: true
            misfireThreshold: 60000
            clusterCheckinInterval: 2000
          scheduler:
            instanceId: AUTO
            instanceName: ${spring.application.name}-scheduler
            skipUpdateCheck: true
          threadPool:
            class: org.quartz.simpl.SimpleThreadPool
            threadCount: 4

scheduler:
  staticJobIntervalInMilliseconds: 5000
```
I'd like to point out some parameters:
- `spring.datasource.*` parameters are used to configure managed by Spring datasource for database connection.
- `spring.liquibase.changeLog` - path for Liquibase change log file.
- `spring.quartz.jdbc.initialize-schema: never`. We use parameter `never` in conjunction with Liquibase to control the creation of tables for Quartz. With other parameter values Spring will recreate service tables for Quartz every time the application starts. We don't need this, because we don't want data to be lost.
- `spring.quartz.properties.org.quartz.jobStore.class` - using a `LocalDataSourceJobStore` class means that Quartz will use managed by Spring datasource instead of its own.
- `spring.quartz.properties.org.quartz.jobStore.isClustered: true` - this parameter lets Quartz know that we are using multiple instances of our application.
- `scheduler.staticJobIntervalInMilliseconds: 5000` - parameter of our application which we will use to configure the trigger of our static job example to fire every 5 seconds.

After completing these steps, let's proceed to describe jobs. Describing a job created at runtime doesn't differ significantly. I just recommend adding an identity key to both a job and a trigger manually. Assigning clear names to keys will allow us to distinguish between jobs and triggers in the database more easily, simplifying the debugging process.

```java
@RestController
public class Controller {

  private static final Logger log = LoggerFactory.getLogger(Controller.class);

  private final SchedulerFactoryBean schedulerFactoryBean;

  public Controller(SchedulerFactoryBean schedulerFactoryBean) {
    this.schedulerFactoryBean = schedulerFactoryBean;
  }

  @GetMapping("/job/{jobName}")
  public void scheduleTask(@PathVariable String jobName) throws SchedulerException {
    log.info("Dynamic job {} is scheduling", jobName);
    var jobDetail = JobBuilder.newJob(DynamicJobExample.class)
        .withIdentity(DynamicJobExample.DYNAMIC_JOB_IDENTITY_KEY + jobName)
        .usingJobData(DynamicJobExample.JOB_NAME, jobName)
        .build();
    var trigger = TriggerBuilder.newTrigger()
        .forJob(jobDetail.getKey().getName())
        .withIdentity(DynamicJobExample.DYNAMIC_JOB_IDENTITY_KEY + jobName)
        .startNow()
        .withSchedule(SimpleScheduleBuilder.simpleSchedule()).build();
    schedulerFactoryBean.getScheduler().scheduleJob(jobDetail, trigger);
  }
}
```
And the dynamic job now looks like this:
```java
public class DynamicJobExample implements Job {

  public static final String JOB_NAME = "JOB_NAME";
  public static final String DYNAMIC_JOB_IDENTITY_KEY = "DYNAMIC_JOB_";
  private static final Logger log = LoggerFactory.getLogger(DynamicJobExample.class);

  @Override
  public void execute(final JobExecutionContext context) {
    var jobDataMap = context.getJobDetail().getJobDataMap();
    var jobName = jobDataMap.getString(JOB_NAME);
    log.info("Dynamic job {} is running", jobName);
  }
}
```
Now when creating a task it will be saved to the database instead of living in the application memory, which will prevent its loss when restarting the application.

The last thing we have left to figure out is how to create static jobs. Of course, we can use the Spring scheduling mechanism, but then the task will be launched on each application node. How to make a static job run but only on one application instance in the cluster? This is where Quartz will help us again.

I will describe the job and it's configuration in one class `StaticJobExampleConfig`for simplicity. Inside this class let's put the definition of the job:
```java
@DisallowConcurrentExecution
public static class StaticJobExampleJob implements Job {

	private static final Logger log = LoggerFactory.getLogger(StaticJobExampleJob.class);
	private static final DateTimeFormatter dtf = DateTimeFormatter.ofPattern("HH:mm:ss");
	static final String STATIC_JOB_IDENTITY_KEY = "STATIC_JOB";
	
	@Override
	public void execute(final JobExecutionContext context) {
	  log.info("Static job is running. Current time is {}", LocalTime.now().format(dtf));
	}
}
```
Note the annotation `@DisallowConcurrentExecution` above the class. This annotation marks a Job class as one that must not have multiple instances executed concurrently.

Now let's describe the `JobDetails`:
```java
@Bean
public JobDetail jobDetail() {
	return JobBuilder.newJob().ofType(StaticJobExampleJob.class)
	.storeDurably()
	.withIdentity(STATIC_JOB_IDENTITY_KEY)
	.build();
}
```
And `Trigger`:
```java
@Bean
public Trigger trigger(
  @Qualifier("jobDetail") JobDetail jobDetail,
  @Value("${scheduler.staticJobIntervalInMilliseconds}") long intervalInMillis
) {
return TriggerBuilder.newTrigger().forJob(jobDetail)
	.withIdentity(STATIC_JOB_IDENTITY_KEY)
	.withSchedule(
		simpleSchedule()
			.repeatForever()
			.withIntervalInMilliseconds(intervalInMillis)
			.withMisfireHandlingInstructionIgnoreMisfires())
	.build();
}
```
The `@Bean` annotation enables Spring to discover this job and schedule it, but don't forget to annotate the entire class with the Spring annotation `@Configuration`. `@Value("${scheduler.staticJobIntervalInMilliseconds}") long intervalInMillis` argument in trigger allows us to configure trigger through application settings.
Finally, our class will look like this:
```java
@Configuration
public class StaticJobExampleConfig {

  @Bean
  public JobDetail jobDetail() {
    return JobBuilder.newJob().ofType(StaticJobExampleJob.class)
        .storeDurably()
        .withIdentity(STATIC_JOB_IDENTITY_KEY)
        .build();
  }

  @Bean
  public Trigger trigger(
      @Qualifier("jobDetail") JobDetail jobDetail,
      @Value("${scheduler.staticJobIntervalInMilliseconds}") long intervalInMillis
  ) {
    return TriggerBuilder.newTrigger().forJob(jobDetail)
        .withIdentity(STATIC_JOB_IDENTITY_KEY)
        .withSchedule(
            simpleSchedule()
                .repeatForever()
                .withIntervalInMilliseconds(intervalInMillis)
                .withMisfireHandlingInstructionIgnoreMisfires())
        .build();
  }

  @DisallowConcurrentExecution
  public static class StaticJobExampleJob implements Job {

    private static final Logger log = LoggerFactory.getLogger(StaticJobExampleJob.class);
    private static final DateTimeFormatter dtf = DateTimeFormatter.ofPattern("HH:mm:ss");
    static final String STATIC_JOB_IDENTITY_KEY = "STATIC_JOB";

    @Override
    public void execute(final JobExecutionContext context) {
      log.info("Static job is running. Current time is {}", LocalTime.now().format(dtf));
    }
  }
}
```
Done. Now we can run the application and make sure that our static job writes a message to the logs every 5 seconds. If we send a request to our rest controller, we will also see a message in the logs about the creation of a job in runtime.

What have we learned from this article? We've learned how to schedule jobs with Spring Boot and Quartz scheduler dynamically in application runtime and statically, and how application structure can look. I hope this article will be a good starting point for you in working with these technologies.