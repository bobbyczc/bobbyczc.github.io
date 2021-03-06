---
layout: post
title: 'spring batch手册(一)——配置Step'
subtitle: '配置Step'
date: 2020-04-30
categories: SpringBatch
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-theme-h2o-postcover.jpg'
tags: SpringBatch
---

# 1. 配置Step

## 1.2 `TaskletStep`

面向Chunk的处理不是 `Step` 中唯一的处理方式，如果一个 `Step` 必须包括一个简单的存储过程调用该怎么办？你可以像 `ItemReader` 一样实现这个调用并在这个过程完成后返回空值。然而这样做有点不太自然，因为这样就需要有一个没有任何操作的 `ItemWriter` 。Spring Batch 为这个场景提供了 `TaskletStep` 。

`Tasklet` 是一个只有一个 `execute()` 方法的接口，这个方法会被 `Tasklet` 重复调用直到它返回 `RepeatStatus.FINISHED` 或者抛出异常表示失败。每个对 `Tasklet` 的调用都会包含在一个事务中。`Tasklet` 的实现可能会调用一个存储过程，一个脚本或者一个简单的SQL Update语句。

要创建一个 `TaskletStep` ，传递给 `StepBuilder` 的 `tasklet()` 方法的bean必须实现 `Tasklet` 接口，并且在创建 `TaskletStep` 时不能调用 `chunk`  方法。下面是一个简单的tasklet的例子：

```java
@Bean
public Step step1() {
    return this.stepBuilderFactory.get("step1")
                            .tasklet(myTasklet())
                            .build();
}
```

> 如果这个tasklet实现了 `StepListener` 接口， `TaskletStep` 会自动注册这个tasklet作为 `StepListener` 

### 1.2.1 `TaskletAdapter`

就像其他 `ItemReader` 和 `ItemWriter` 接口的适配器一样， `Tasklet` 接口有一个实现 `TaskletAdapter` 允许适配自身到任何已存在的类中。一个非常有用的情形是：已存在一个用来在一些记录中更新一个标签的DAO类，`TaskletAdapter` 可以用来调用这个类，我们甚至不需要为 `Tasklet` 接口写一个适配器，就像下面的例子一样：

```java
@Bean
public MethodInvokingTaskletAdapter myTasklet() {
        MethodInvokingTaskletAdapter adapter = new MethodInvokingTaskletAdapter();

        adapter.setTargetObject(fooDao());
        adapter.setTargetMethod("updateFoo");

        return adapter;
}
```

 ### 1.2.2 Example `Tasklet` Implementation

很多批处理工作包含了很多步骤，这些步骤必须在主要处理程序之前开始来让资源准备就绪或者在程序结束之后清理这些资源。在批处理工作需要处理大量文件的情况下，当文件被成功上传到另一个位置的时候删除本地的特定文件是非常有必要的。下面的例子就是一个 负责上述问题的`Tasklet` 实现。

```java
public class FileDeletingTasklet implements Tasklet, InitializingBean {

    private Resource directory;

    public RepeatStatus execute(StepContribution contribution,
                                ChunkContext chunkContext) throws Exception {
        File dir = directory.getFile();
        Assert.state(dir.isDirectory());

        File[] files = dir.listFiles();
        for (int i = 0; i < files.length; i++) {
            boolean deleted = files[i].delete();
            if (!deleted) {
                throw new UnexpectedJobExecutionException("Could not delete file " +
                                                          files[i].getPath());
            }
        }
        return RepeatStatus.FINISHED;
    }

    public void setDirectoryResource(Resource directory) {
        this.directory = directory;
    }

    public void afterPropertiesSet() throws Exception {
        Assert.notNull(directory, "directory must be set");
    }
}
```

上面的 `Tasklet` 实现删除给定目录的所有文件，需要注意的是，`execute` 方法只会被调用一次，剩下的只有在 `Step` 中引用 `Tasklet` ：

```java
@Bean
public Job taskletJob() {
        return this.jobBuilderFactory.get("taskletJob")
                                .start(deleteFilesInDir())
                                .build();
}

@Bean
public Step deleteFilesInDir() {
        return this.stepBuilderFactory.get("deleteFilesInDir")
                                .tasklet(fileDeletingTasklet())
                                .build();
}

@Bean
public FileDeletingTasklet fileDeletingTasklet() {
        FileDeletingTasklet tasklet = new FileDeletingTasklet();

        tasklet.setDirectoryResource(new FileSystemResource("target/test-outputs/test-dir"));

        return tasklet;
}
```

## 1.3 Controlling Step Flow

既然能将多个step放在同一个job中，那么就应该有能力去控制各个step的执行流程，一个 `Step` 的失败不意味着一整个 `Job` 应该失败。更有甚者，可能会有不止一种类型的情况来确认接下来哪个 `Step`  需要被执行， 取决于各个 `Step` 的配置情况，有些特定的 step甚至可能根本不执行。

### 1.3.1 Sequential Flow 

最简单的场景就是一个job的所有step全都串行执行，就像下图显示的一样

![Figure 3. Sequential Flow][Sequential Flow]

这可以通过使用step元素的'next'属性实现，像下面的代码所示：

```java
@Bean
public Job job() {
        return this.jobBuilderFactory.get("job")
                                .start(stepA())
                                .next(stepB())
                                .next(stepC())
                                .build();
}
```

在上面的场景中，‘step A’首先执行，因为它是第一个列出来的Step，如果Step A正常结束，Step B就会运行然后是Step C。然而，如果A失败了，那么B和C都不会执行。

### 1.3.2 Conditional Flow

在上面的例子中，只有两种可能的情况：

- Step 成功了，下一个Step应该被执行
- Step失败了，那么Job就应该失败了

在很多情况下，这可能满足要求。但是在一个step失败可能会触发不同的step而不是导致job失败的场景下，该怎么办呢？下面的图片显示了另一种流程

![][conditional flow]

为了处理更多复杂的场景，Spring Batch命名空间允许在step元素中定义transition元素，一个这样的transition是 `next` 元素，`next` 元素通知 `Job` 接下来执行哪个 `Step` 。然而和transition属性不一样，任意数量的 `next` 元素可以被添加到给定的 `Step` 上，而且在失败的情形下没有默认的行为。这意味着如果使用了transition元素，那么这个Step元素的所有行为都需要被显式的定义。注意，一个单独的Step不能同时拥有 `next` 和 `transition` 元素。

`transition` 元素声明了一个具体的pattern来匹配接下来哪个step会执行，如下面的例子所示：

```java
@Bean
public Job job() {
        return this.jobBuilderFactory.get("job")
                                .start(stepA())
                                .on("*").to(stepB())
                                .from(stepA()).on("FAILED").to(stepC())
                                .end()
                                .build();
}
```

当使用java配置，`on` 方法使用了一个简单模式匹配的方式来匹配一个 `Step` 执行后返回的`ExitStatus` ，只有两个特殊的字符能出现这个匹配串中：

- “*” 匹配0或者更多的字符
- “?" 仅匹配一个字符

例如，“c*t” 匹配 “cat” 和 ”count“ ，而 "c?t" 只能匹配 ”cat“ 不能匹配 ”count”。

由于 `Step` 上没有 transition元素的数量限制，如果 `Step` 执行导致了一个没有被这些元素提及的 `ExitStatus` ，那么框架会直接抛出异常，`Job` 失败结束，框架会自动将 transitions 按最具体到最不具体的顺序排序。这意味着，在上面的例子中，如果Step A返回了“FAILED”的 `ExitStatus` ，下一个执行的会是C而不是B，因为即使FAILED都能符合B和C的匹配，但是到C的匹配条件更为具体。

#### Batch Status Versus Exit Status

当给有条件的流程配置 `Job` 时，理解 `BatchStatus` 以及 `ExitStatus` 之间的不同是很重要的。`BatchStatus` 是一个枚举，是 `JobExecution` 以及 `StepExecution` 的属性，被框架用来记录一个 `Job` 或者 `Step` 的状态，它可以是 `COMPLETED` , `STARTING` ,`STARTED` ,`STOPING` , `FAILED` ,`ABANDONED` 或者 `UNKNOWN` 其中的任何一个值。大多数都是具有自我解释性，`COMPLETED` 表示 step 或者 job 成功完成，`FAILED` 表示失败。

当使用java配置的时候，下面的例子包含了 ‘on’ 元素：

```java
...
.from(stepA()).on("FAILED").to(stepB())
...
```

乍看一下，‘on’ 元素似乎引用了 `Step` 的 `BatchStatus` ，但是实际上它表示的是 `Step` 的 `ExitStatus` ，就像名字显示的一样，`ExitStatus` 表示一个 `Step` 完成执行之后的状态。

在英语中可以这么说，“如果退出码是 `FAILED` 流程走向Step B”。默认情况下，退出码总是和 `Step` 的 `BatchStatus` 的值是一样的 ，这也是上面的方式有效的原因。但是如果我们的退出码需要不一样呢？下面是一个很好的例子：

```java
@Bean
public Job job() {
	return this.jobBuilderFactory.get("job")
			.start(step1()).on("FAILED").end()
			.from(step1()).on("COMPLETED WITH SKIPS").to(errorPrint1())
			.from(step1()).on("*").to(step2())
			.end()
			.build();
}
```

`step1` 有三种可能：

1. 失败，这种情况下，job直接失败
2. 成功完成
3. 成功完成，但是退出码为 ‘COMPLETED WITH SKIPS’，在这种情况下，一个不一样的步骤会来处理这些错误

上面的配置是有效的。但是在执行跳过了一些记录的情况下，我们需要有一种机制来改变退出码，就像下面的例子一样：

```java
public class SkipCheckingListener extends StepExecutionListenerSupport {
    public ExitStatus afterStep(StepExecution stepExecution) {
        String exitCode = stepExecution.getExitStatus().getExitCode();
        if (!exitCode.equals(ExitStatus.FAILED.getExitCode()) &&
              stepExecution.getSkipCount() > 0) {
            return new ExitStatus("COMPLETED WITH SKIPS");
        }
        else {
            return null;
        }
    }
}
```

上面的代码是一个 `stepExecutionListener` ，它会首先确认 `Step` 成功完成，然后检查是否 `StepExecution` 的跳过的数量大于0；如果两个条件都满足的话，就会返回一个新的 `ExitStatus` 退出码为 `COMPLETED WITH SKIPS` 。

### 1.3.3 Configuring for Stop

讨论完 BatchStatus 和ExitStatus 之后，有人可能会好奇如何为 `Job` 定义 `BatchStatus` 和 `ExitStatus` 。`Step` 返回的状态是由具体的执行代码决定的，`Job` 的状态是基于配置来确定的。

目前为止，所有讨论到的job配置都至少有一个固定的没有transition的 `Step` 。像下面的例子一样，step执行完，job就结束了：

```java
@Bean
public Job job() {
	return this.jobBuilderFactory.get("job")
				.start(step1())
				.build();
}
```

如果一个 `Step` 没有定义事务（条件逻辑），那么 `Job` 的状态有下面几种可能：

- 如果 `Step` 返回 `FAILED` 的 `ExitStatus` ，那么 `Job` 的 `BatchStatus` 和 `ExitStatus` 都是 `FAILED`。
- 否则 `Job` 的 `BatchStatus` 和 `ExitStatus` 都是 `COMPLETED` 。

虽然这种结束批处理作业的方法能满足一些场景，但是，我们仍可能需要自定义的作业停止场景。为了达成这种目的，Spring Batch 提供了三种 stopping elements来停止一个 `Job`（上面已经提及的 `next` 元素除外）。这些 stopping elements 中的每一个都会用一个特殊的 `BatchStatus` 停止一个 `Job` 。特别值得注意的是，这些stop elements对这个 `Job` 中的任意 `Steps` 的 `BatchStatus` 和 `ExitStatus` 没有任何影响，它们只影响这个 `Job` 的最终状态。比如，就算job中的每个step都是 `FAILED` 的状态，但是Job也有可能是 `COMPLETED` 的状态。

#### Ending at a Step

配置一个 step end 会让一个 `Job` 停止并返回 `COMPLETED` 的 `BatchStatus` 。一个以 `COMPLETED` 状态结束的 `Job` 不可以被重新启动（框架会抛出 `JobInstanceAlreadyCompleteException` 异常）。

当使用Java配置时，可以使用 ‘end’ 方法。`end` 方法同样允许可选的 'exitStatus' 参数来自定义 `Job` 的 `ExitStatus` 。如果没有提供 ‘exitStatus’ 那么默认的 `ExitStatus` 为 `COMPLETED` 来匹配 `BatchStatus` 。

在下面的场景中，如果 `step2` 失败了，那么 `Job` 会停止，`BatchStatus` 以及 `ExitStatus` 都为 `COMPLETED` ，并且 `step3` 不会执行。否则，`step3` 执行。注意如果 `step2` 失败了，那么 `Job` 是不能重启的，因为状态是 `COMPLETED` 。

```java
@Bean
public Job job() {
	return this.jobBuilderFactory.get("job")
				.start(step1())
				.next(step2())
				.on("FAILED").end()
				.from(step2()).on("*").to(step3())
				.end()
				.build();
}
```

#### Failing a Step

配置 step 在某个点失败会让一个 `Job` 以 `FAILED` 的 `BatchStatus` 停止，和end不一样，失败的 `Job` 可以重启。

在下面的场景值，如果 `step2` 失败，那么 `Job` 停止，`BatchStatus` 为 `FAILED` ，`ExitStatus` 为 `EARLY TERMINATION` ，`step3` 不会执行；否则，`step3` 执行。此外，如果 `step2` 失败并且 `Job` 重启，那么会从 `step2` 开始执行。

```java
@Bean
public Job job() {
	return this.jobBuilderFactory.get("job")
			.start(step1())
			.next(step2()).on("FAILED").fail()
			.from(step2()).on("*").to(step3())
			.end()
			.build();
}
```

#### Stopping a Job at a Given Step

配置job在一个特定step停止会让 `Job` 停止并且 `BatchStatus` 为 `STOPPED` 。停止一个 `Job` 可以让处理过程有个短暂的间隙，这样操作符可以在重启 `Job` 之前执行一些操作。

当使用Java配置时，`stopAndRestart` 方法需要有一个 ’restart‘ 参数来指定 job 重启时从哪个step开始执行。

在下面的场景中，如果 `step1` 以 `COMPLETED` 结束，那么job停止。一旦它重新启动，会从 `step2` 开始执行。

```java
@Bean
public Job job() {
	return this.jobBuilderFactory.get("job")
			.start(step1()).on("COMPLETED").stopAndRestart(step2())
			.end()
			.build();
}
```

### 1.3.4 Programmatic Flow Decisions

在一些情况下，可能需要除了 `ExitStatus` 之外的更多信息来决定下一步执行哪个step。在这种情况下，需要使用 `JobExecutionDecider` ，和下面的例子一样：

```java
public class MyDecider implements JobExecutionDecider {
    public FlowExecutionStatus decide(JobExecution jobExecution, StepExecution stepExecution) {
        String status;
        if (someCondition()) {
            status = "FAILED";
        }
        else {
            status = "COMPLETED";
        }
        return new FlowExecutionStatus(status);
    }
}
```

在下面的例子中，`JobExecutionDecider` 的bean实现被直接传递到directly方法中。

```java
@Bean
public Job job() {
	return this.jobBuilderFactory.get("job")
			.start(step1())
			.next(decider()).on("FAILED").to(step2())
			.from(decider()).on("COMPLETED").to(step3())
			.end()
			.build();
}
```

### 1.3.5 Split Flows

目前为止描述的每个场景都 `Job` 以线性的方式每次执行一个step。除了这种典型的形式之外，Spring Batch也允许job被配置为并行的流程。

基于Java的配置方式允许我们通过提供的builders来配置splits，就像下面的例子显示的一样，split元素包含了一个或多个flow元素，在每个flow元素中可以定义整个分离的流程。split也可以包含任意上面已讨论过的transition元素，就像next，end，fail等元素一样。

```java
@Bean
public Job job() {
	Flow flow1 = new FlowBuilder<SimpleFlow>("flow1")
			.start(step1())
			.next(step2())
			.build();
	Flow flow2 = new FlowBuilder<SimpleFlow>("flow2")
			.start(step3())
			.build();

	return this.jobBuilderFactory.get("job")
				.start(flow1)
				.split(new SimpleAsyncTaskExecutor())
				.add(flow2)
				.next(step4())
				.end()
				.build();
}
```

### 1.3.6 Externalizing Flow Definitions and Dependencies Between Jobs

有一部分flow可以在job的外面定义为一个独立的bean定义，可以被复用。有两种方式可以实现。第一个是简单地声明flow作为其他地方定义的引用，就像下面的例子一样：

```java
@Bean
public Job job() {
	return this.jobBuilderFactory.get("job")
				.start(flow1())
				.next(step3())
				.end()
				.build();
}

@Bean
public Flow flow1() {
	return new FlowBuilder<SimpleFlow>("flow1")
			.start(step1())
			.next(step2())
			.build();
}
```

像上面的例子一样在外面定义flow的影响就是从外面的flow讲steps插入到job中，就像他们已经在job内部定义了一样。通过这种方式，一些jobs可以一用同样的模板flow然后组合这些模板成不同的逻辑flows。这也是一个很好的方式来分离每个flow的集成测试。

另一种外部flow的声明方式是使用 `JobStep` ，`JobStep` 和 `FlowStep` 相似，但是实际上会为flow中得到steps创建并且运行一个独立的job执行，像下面的例子一样：

```java
@Bean
public Job jobStepJob() {
	return this.jobBuilderFactory.get("jobStepJob")
				.start(jobStepJobStep1(null))
				.build();
}

@Bean
public Step jobStepJobStep1(JobLauncher jobLauncher) {
	return this.stepBuilderFactory.get("jobStepJobStep1")
				.job(job())
				.launcher(jobLauncher)
				.parametersExtractor(jobParametersExtractor())
				.build();
}

@Bean
public Job job() {
	return this.jobBuilderFactory.get("job")
				.start(step1())
				.build();
}

@Bean
public DefaultJobParametersExtractor jobParametersExtractor() {
	DefaultJobParametersExtractor extractor = new DefaultJobParametersExtractor();

	extractor.setKeys(new String[]{"input.file"});

	return extractor;
}
```

job parameters extractor是一个决定 `Step` 的 `ExecutionContext` 如何转变成要执行 `Job` 的 `JobParameters` 的策略。 当你想要关于监控和报告jobs和steps的一些更细粒度的选择时，`JobStep` 会很有用。使用 `JobStep` 也是 ‘我如何创建jobs之间的依赖’ 这个问题的一个很好的答案。将一个大系统拆分为更小一些的模块来控制jobs的flow是一个很好的方式。

## 1.4 Late Binding of `Job` and `Step` Attributes

前面已经介绍的xml和平面文件都使用 `Resource` 来获取一个文件，这个能生效是因为 `Resource` 有一个 `getFile` 方法，这个方法返回一个 `java.io.File` 对象。xml或者flat文件都能使用标准的Spring方法配置，就像下面的例子一样：

```java
@Bean
public FlatFileItemReader flatFileItemReader() {
        FlatFileItemReader<Foo> reader = new FlatFileItemReaderBuilder<Foo>()
                        .name("flatFileItemReader")
                        .resource(new FileSystemResource("file://outputs/file.txt"))
                        ...
}
```

上面的 `Resource` 从具体的文件系统位置来获取文件。注意这个文件的绝对路径由双斜线（ `//` ）开头。在绝大多数Spring应用中，这种方式已经足够好了，因为这些资源的名字是在编译时可见的。但是，在批处理场景中，文件的名字可能是需要在运行时作为job的参数传入的。这个可以使用 ‘-D’参数来读取系统属性来解决。

下面的例子显示了如何从系统属性中获取文件名：

```java
@Bean
public FlatFileItemReader flatFileItemReader(@Value("${input.file.name}") String name) {
        return new FlatFileItemReaderBuilder<Foo>()
                        .name("flatFileItemReader")
                        .resource(new FileSystemResource(name))
                        ...
}
```

实现这个只需要一个系统参数（比如：`-Dinput.file.name="file://outputs/file.txt"` ）

> 虽然这里可以使用 `PropertyPlaceholderConfigurer` ，但是如果当系统属性一直存在时，是没必要使用的。因为Spring的 `ResourceEditor` 已经过滤并完成了系统属性的替换 。

通常情况下，在批处理设置中，更倾向于将文件名配置在job的 `JobParameters` 中，而不是通过系统属性来获取。要实现这个，Spring Batch允许 `Job` 和 `Step` 属性的后期绑定，就像下面的例子一样：

```java
@StepScope
@Bean
public FlatFileItemReader flatFileItemReader(@Value("#{jobParameters['input.file.name']}") String name) {
        return new FlatFileItemReaderBuilder<Foo>()
                        .name("flatFileItemReader")
                        .resource(new FileSystemResource(name))
                        ...
}
```

不管是在 `JobExecution` 还是 `StepExecution` 层面，`ExecutionContext` 都能通过一样的方式访问，如下所示：

```java
@StepScope
@Bean
public FlatFileItemReader flatFileItemReader(@Value("#{jobExecutionContext['input.file.name']}") String name) {
        return new FlatFileItemReaderBuilder<Foo>()
                        .name("flatFileItemReader")
                        .resource(new FileSystemResource(name))
                        ...
}
```

```java
@StepScope
@Bean
public FlatFileItemReader flatFileItemReader(@Value("#{stepExecutionContext['input.file.name']}") String name) {
        return new FlatFileItemReaderBuilder<Foo>()
                        .name("flatFileItemReader")
                        .resource(new FileSystemResource(name))
                        ...
}
```

> 所有使用后期绑定的bean都必须被声明为scope=‘step’  `@StepScope`

### 1.4.1.Step Scope

上面所有使用了后期绑定的例子都在bean的声明上使用了 `@StepScope` .

要使用后期绑定来发现属性，使用 `Step` 的范围是必须的，因为bean在 `Step` 开始之前实际上是不会被实例化的。因为默认情况下它不是Spring容器的一部分，所以必须显式添加scope，通过 `batch` 的namespace或者为 `StepScope` 显式添加一个bean,  或者使用 `EnableBatchProcessing` 注解。

`batch` 的namespace：

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:batch="http://www.springframework.org/schema/batch"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="...">
<batch:job .../>
...
</beans>
```

显式包含 `StepScope` ：

```xml
<bean class="org.springframework.batch.core.scope.StepScope" />
```

### 1.4.2.Job Scope

Spring Batch 3.0中引入了 `job` scope ，在配置上和 `Step` scope相似但是属于 `Job` 上下文的范围，因此每一次运行中的job只会有一个这个bean的实例。除此之外，还为使用 `#{..}` 的 `JobContext` 引用的后期绑定提供的了支持。使用这个特性，bean的属性可以从job或者job execution context 和job parameters 获取，像下面的例子一样：

```java
@JobScope
@Bean
public FlatFileItemReader flatFileItemReader(@Value("#{jobParameters[input]}") String name) {
        return new FlatFileItemReaderBuilder<Foo>()
                        .name("flatFileItemReader")
                        .resource(new FileSystemResource(name))
                        ...
}
```

```java
@JobScope
@Bean
public FlatFileItemReader flatFileItemReader(@Value("#{jobParameters[input]}") String name) {
        return new FlatFileItemReaderBuilder<Foo>()
                        .name("flatFileItemReader")
                        .resource(new FileSystemResource(name))
                        ...
}
```

同样的，因为默认情况下它不是Spring容器的一部分，所以必须显式添加scope，通过 `batch` 的namespace或者为 `StepScope` 显式添加一个bean,  或者使用 `EnableBatchProcessing` 注解。

`batch` 的namespace：

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
                  xmlns:batch="http://www.springframework.org/schema/batch"
                  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                  xsi:schemaLocation="...">

<batch:job .../>
...
</beans>
```

 显式包含 `StepScope` ：

```xml
<bean class="org.springframework.batch.core.scope.JobScope" />
```

[Sequential Flow]: data:image/png;base64,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L1687777amuv3n77bZyrqvd47bXXaijchx56qPkORRPqXr169cEHH9xq+Z48eRK6jREffvjhlgp3dHS0KY9D0/5G9Oijj26yfFtbW7/77jvoNlJ88cUXm1n7trS0PPPMM816EJr5990nn3zyb4E7Ojp+/fVX6DZenD17dv/+/dV1X3jhhSY+Ak3+txnBYLBKM3X//ff/+eef0G3UOH/+/N13311pfH799deb++M3/92mXn75ZdHC1Wg0165dg25jx5UrV1Qq1Y2ro48++qjpP/ueuFPc8ePHywq36e8itod0r1+/bjKZ+PKVSCRffvkldJsnPv/8c/70xVNPPbVHPvUeuofn+Ph48S/Uz549C91mizNnzrS1tT333HN75yPvrfvvvvjii+fPn4cuAroI6CKgi4AuAroI6EIXAV0EdBF1o3vhwoV1RCNElf+SKq57/vx5giDab+1A1n8SBHHixIkt6K6trXXcIX04dgxZ/6kYfOTo0aPQhS50oYuELhK6SOgioQtd6EIXutBFQhcJXSR0kdCFLnShi4TuLmaO9kV6HR7VsFtpc1OO+YFA5v/z3Y2zkYPTkf6NaZhdppdy0N1eRleUlMiViEjbvEUAMDDhJghdf2g39iGjIiteyUw+GoNu7XWj0Vc8sjJnorBNtpfdZrd0KarapeqUkyno1la4iS72GOp6phPWaM4aWukxcgdb4TYXdLmjrzNEd1WX6p1ZoedWjP7EoC+i1MqLj0qHI9CtKcMxblC06/lJLhSU5odmokPvtcSy2iEdX0ZSipYPL1oLm9HTXpmCA6BGtL40r6UZYkgtrXTFDBPuzuIbkDq1J/F3urQhLHjcP1V8tHM4Bt3aMtktGAM79WOUa9Ewl2IEh16tKJuQ8wU94LCLDKGeJPuUCiMtORqvXrsaX8q8kDIFkkZ/TKWnNk4Q0N160p4xMQed0hkrGGf7DpUgO4125WiQWSoUd6Hp6XbMax38KzAD0bJ5VKd2+tVmvvqpg0tbnHdJjwVd1baAp72kWNcqtQULwLnSyFmYdwcdDFeLy8VXMBxi/VQzaYGWvNef4/oyObdBZotdFdUzg65q282zaS7e55rq0uuER7bQJPMjLdsz85YdpLyY/BPUeTxOi5riy84y4648OJe+DaqJYP908OBkUOvydet5843zMXQ3maYZr0xLS0miy7VSejwU57k2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

[conditional flow]: data:image/png;base64,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