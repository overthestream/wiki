---
title: "SimpleAsyncTaskExecutor (Spring Framework 7.0.6 API)"
source: "https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/task/SimpleAsyncTaskExecutor.html"
author:
published:
created: 2026-04-15
description: "declaration: package: org.springframework.core.task, class: SimpleAsyncTaskExecutor"
tags:
  - "clippings"
---
public class SimpleAsyncTaskExecutor extends [CustomizableThreadCreator](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/util/CustomizableThreadCreator.html "class in org.springframework.util") implements [AsyncTaskExecutor](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/task/AsyncTaskExecutor.html "interface in org.springframework.core.task"), [Serializable](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/io/Serializable.html "class or interface in java.io"), [AutoCloseable](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/AutoCloseable.html "class or interface in java.lang")

[`TaskExecutor`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/task/TaskExecutor.html "interface in org.springframework.core.task") implementation that fires up a new Thread for each task. Provides a [`virtual threads`](#setVirtualThreads\(boolean\)) option on JDK 21+.

Supports a graceful shutdown through [`setTaskTerminationTimeout(long)`](#setTaskTerminationTimeout\(long\)), at the expense of task tracking overhead per execution thread at runtime. Supports limiting concurrent threads through [`setConcurrencyLimit(int)`](#setConcurrencyLimit\(int\)); by default, the number of concurrent task executions is unlimited. Can be combined with [`RetryTask`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/retry/support/RetryTask.html "class in org.springframework.core.retry.support") for re-executing submitted tasks after failure, according to a retry policy.

**NOTE: This implementation does not reuse threads!** Consider a thread-pooling TaskExecutor implementation instead, in particular for executing a large number of short-lived tasks. Alternatively, on JDK 21, consider setting [`setVirtualThreads(boolean)`](#setVirtualThreads\(boolean\)) to `true`.

**NOTE: This executor does not participate in context-level lifecycle management.** Tasks on handed-off execution threads cannot be centrally stopped and restarted; if such tight lifecycle management is necessary, consider a common `ThreadPoolTaskExecutor` setup instead.

Since:

2.0

Author:

Juergen Hoeller

See Also:

- ## Field Summary
	Fields
	Modifier and Type
	Field
	Description
	`static final int`
	`NO_CONCURRENCY`
	Switch concurrency 'off': that is, don't allow any concurrent invocations.
	`static final int`
	`UNBOUNDED_CONCURRENCY`
	Permit any number of concurrent invocations: that is, don't throttle concurrency.
- ## Constructor Summary
	Constructors
	Constructor
	Description
	`SimpleAsyncTaskExecutor()`
	Create a new SimpleAsyncTaskExecutor with default thread name prefix.
	`SimpleAsyncTaskExecutor(String threadNamePrefix)`
	Create a new SimpleAsyncTaskExecutor with the given thread name prefix.
	`SimpleAsyncTaskExecutor(ThreadFactory threadFactory)`
	Create a new SimpleAsyncTaskExecutor with the given external thread factory.
- ## Method Summary
	Modifier and Type
	Method
	Description
	`void`
	`close()`
	This close method tracks the termination of active threads if a concrete [`task termination timeout`](#setTaskTerminationTimeout\(long\)) has been set.
	`protected void`
	`doExecute(Runnable task)`
	Template method for the actual execution of a task.
	`void`
	`execute(Runnable task)`
	Executes the given task, within a concurrency throttle if configured (through the superclass's settings).
	`void`
	`execute(Runnable task,  long startTimeout)`
	Deprecated.
	`final int`
	`getConcurrencyLimit()`
	Return the maximum number of parallel task executions allowed.
	`final @Nullable ThreadFactory`
	`getThreadFactory()`
	Return the external factory to use for creating new Threads, if any.
	`boolean`
	`isActive()`
	Return whether this executor is still active, i.e.
	`final boolean`
	`isThrottleActive()`
	Return whether the concurrency throttle is currently active.
	`protected Thread`
	`newThread(Runnable task)`
	Create a new Thread for the given task.
	`void`
	`setCancelRemainingTasksOnClose(boolean cancelRemainingTasksOnClose)`
	Specify whether to cancel remaining tasks on close: that is, whether to interrupt any active threads at the time of the [`close()`](#close\(\)) call.
	`void`
	`setConcurrencyLimit(int concurrencyLimit)`
	Set the maximum number of parallel task executions allowed.
	`void`
	`setRejectTasksWhenLimitReached(boolean rejectTasksWhenLimitReached)`
	Specify whether to reject tasks when the concurrency limit has been reached, throwing [`TaskRejectedException`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/task/TaskRejectedException.html "class in org.springframework.core.task") (which extends the common [`RejectedExecutionException`](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/RejectedExecutionException.html "class or interface in java.util.concurrent")) on any further execution attempts.
	`void`
	`setTaskDecorator(TaskDecorator taskDecorator)`
	Specify a custom [`TaskDecorator`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/task/TaskDecorator.html "interface in org.springframework.core.task") to be applied to any [`Runnable`](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Runnable.html "class or interface in java.lang") about to be executed.
	`void`
	`setTaskTerminationTimeout(long timeout)`
	Specify a timeout (in milliseconds) for task termination when closing this executor.
	`void`
	`setThreadFactory(@Nullable ThreadFactory threadFactory)`
	Specify an external factory to use for creating new Threads, instead of relying on the local properties of this executor.
	`void`
	`setVirtualThreads(boolean virtual)`
	Switch this executor to virtual threads.
	`Future<?>`
	`submit(Runnable task)`
	Submit a Runnable task for execution, receiving a Future representing that task.
	`<T> Future<T>`
	`submit(Callable<T> task)`
	Submit a Callable task for execution, receiving a Future representing that task.

- ## Field Details
	- ### UNBOUNDED\_CONCURRENCY
		public static final int UNBOUNDED\_CONCURRENCY
		Permit any number of concurrent invocations: that is, don't throttle concurrency.
		See Also:
	- ### NO\_CONCURRENCY
		public static final int NO\_CONCURRENCY
		Switch concurrency 'off': that is, don't allow any concurrent invocations.
		See Also:
- ## Constructor Details
	- ### SimpleAsyncTaskExecutor
		public SimpleAsyncTaskExecutor()
		Create a new SimpleAsyncTaskExecutor with default thread name prefix.
	- ### SimpleAsyncTaskExecutor
		public SimpleAsyncTaskExecutor([String](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/String.html "class or interface in java.lang") threadNamePrefix)
		Create a new SimpleAsyncTaskExecutor with the given thread name prefix.
		Parameters:
		`threadNamePrefix` - the prefix to use for the names of newly created threads
	- ### SimpleAsyncTaskExecutor
		public SimpleAsyncTaskExecutor([ThreadFactory](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/ThreadFactory.html "class or interface in java.util.concurrent") threadFactory)
		Create a new SimpleAsyncTaskExecutor with the given external thread factory.
		Parameters:
		`threadFactory` - the factory to use for creating new Threads
- ## Method Details
	- ### setVirtualThreads
		public void setVirtualThreads(boolean virtual)
		Switch this executor to virtual threads. Requires Java 21 or higher.
		The default is `false`, indicating platform threads. Set this flag to `true` in order to create virtual threads instead.
		Since:
		6.1
	- ### setThreadFactory
		public void setThreadFactory([@Nullable](https://jspecify.dev/docs/api/org/jspecify/annotations/Nullable.html "class or interface in org.jspecify.annotations") [ThreadFactory](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/ThreadFactory.html "class or interface in java.util.concurrent") threadFactory)
		Specify an external factory to use for creating new Threads, instead of relying on the local properties of this executor.
		You may specify an inner ThreadFactory bean or also a ThreadFactory reference obtained from JNDI (on a Jakarta EE server) or some other lookup mechanism.
		See Also:
	- ### getThreadFactory
		Return the external factory to use for creating new Threads, if any.
	- ### setTaskDecorator
		public void setTaskDecorator([TaskDecorator](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/task/TaskDecorator.html "interface in org.springframework.core.task") taskDecorator)
		Specify a custom [`TaskDecorator`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/task/TaskDecorator.html "interface in org.springframework.core.task") to be applied to any [`Runnable`](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Runnable.html "class or interface in java.lang") about to be executed.
		Note that such a decorator is not necessarily being applied to the user-supplied `Runnable` / `Callable` but rather to the actual execution callback (which may be a wrapper around the user-supplied task).
		The primary use case is to set some execution context around the task's invocation, or to provide some monitoring/statistics for task execution.
		**NOTE:** Exception handling in `TaskDecorator` implementations is limited to plain `Runnable` execution via `execute` calls. In case of `#submit` calls, the exposed `Runnable` will be a `FutureTask` which does not propagate any exceptions; you might have to cast it and call `Future#get` to evaluate exceptions.
		Since:
		4.3
	- ### setTaskTerminationTimeout
		public void setTaskTerminationTimeout(long timeout)
		Specify a timeout (in milliseconds) for task termination when closing this executor. The default is 0, not waiting for task termination at all.
		Note that a concrete >0 timeout specified here will lead to the wrapping of every submitted task into a task-tracking runnable which involves considerable overhead in case of a high number of tasks. However, for a modest level of submissions with longer-running tasks, this is feasible in order to arrive at a graceful shutdown.
		Note that `SimpleAsyncTaskExecutor` does not participate in a coordinated lifecycle stop but rather just awaits task termination on [`close()`](#close\(\)).
		Parameters:
		`timeout` - the timeout in milliseconds
		Since:
		6.1
		See Also:
	- ### setCancelRemainingTasksOnClose
		public void setCancelRemainingTasksOnClose(boolean cancelRemainingTasksOnClose)
		Specify whether to cancel remaining tasks on close: that is, whether to interrupt any active threads at the time of the [`close()`](#close\(\)) call.
		The default is `false`, not tracking active threads at all or just interrupting any remaining threads that still have not finished after the specified [`taskTerminationTimeout`](#setTaskTerminationTimeout\(long\)). Switch this to `true` for immediate interruption on close, either in combination with a subsequent termination timeout or without any waiting at all, depending on whether a `taskTerminationTimeout` has been specified as well.
		Since:
		6.2.11
		See Also:
	- ### setRejectTasksWhenLimitReached
		public void setRejectTasksWhenLimitReached(boolean rejectTasksWhenLimitReached)
		Specify whether to reject tasks when the concurrency limit has been reached, throwing [`TaskRejectedException`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/task/TaskRejectedException.html "class in org.springframework.core.task") (which extends the common [`RejectedExecutionException`](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/RejectedExecutionException.html "class or interface in java.util.concurrent")) on any further execution attempts.
		The default is `false`, blocking the caller until the submission can be accepted. Switch this to `true` for immediate rejection instead.
		Since:
		6.2.6
		See Also:
	- ### setConcurrencyLimit
		public void setConcurrencyLimit(int concurrencyLimit)
		Set the maximum number of parallel task executions allowed. The default of -1 indicates no concurrency limit at all.
		This is the equivalent of a maximum pool size in a thread pool, preventing temporary overload of the thread management system. However, in contrast to a thread pool with a managed task queue, this executor will block the submitter until the task can be accepted when the configured concurrency limit has been reached. If you prefer queue-based task hand-offs without such blocking, consider using a `ThreadPoolTaskExecutor` instead.
		See Also:
	- ### getConcurrencyLimit
		public final int getConcurrencyLimit()
		Return the maximum number of parallel task executions allowed.
	- ### isThrottleActive
		public final boolean isThrottleActive()
		Return whether the concurrency throttle is currently active.
		Returns:
		`true` if the concurrency limit for this instance is active
		See Also:
	- ### isActive
		public boolean isActive()
		Return whether this executor is still active, i.e. not closed yet, and therefore accepts further task submissions. Otherwise, it is either in the task termination phase or entirely shut down already.
		Since:
		6.1
		See Also:
	- ### execute
		public void execute([Runnable](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Runnable.html "class or interface in java.lang") task)
		Executes the given task, within a concurrency throttle if configured (through the superclass's settings).
		Specified by:
		`execute` in interface `Executor`
		Specified by:
		`execute` in interface `TaskExecutor`
		Parameters:
		`task` - the `Runnable` to execute (never `null`)
		See Also:
	- ### execute
		Deprecated.
		Executes the given task, within a concurrency throttle if configured (through the superclass's settings).
		Executes urgent tasks (with 'immediate' timeout) directly, bypassing the concurrency throttle (if active). All other tasks are subject to throttling.
		Specified by:
		`execute` in interface `AsyncTaskExecutor`
		Parameters:
		`task` - the `Runnable` to execute (never `null`)
		`startTimeout` - the time duration (milliseconds) within which the task is supposed to start. This is intended as a hint to the executor, allowing for preferred handling of immediate tasks. Typical values are [`AsyncTaskExecutor.TIMEOUT_IMMEDIATE`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/task/AsyncTaskExecutor.html#TIMEOUT_IMMEDIATE) or [`AsyncTaskExecutor.TIMEOUT_INDEFINITE`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/task/AsyncTaskExecutor.html#TIMEOUT_INDEFINITE) (the default as used by [`TaskExecutor.execute(Runnable)`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/task/TaskExecutor.html#execute\(java.lang.Runnable\))).
		See Also:
	- ### submit
		public [Future](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/Future.html "class or interface in java.util.concurrent") <?> submit([Runnable](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Runnable.html "class or interface in java.lang") task)
		Description copied from interface: `AsyncTaskExecutor`
		Submit a Runnable task for execution, receiving a Future representing that task. The Future will return a `null` result upon completion.
		As of 6.1, this method comes with a default implementation that delegates to [`TaskExecutor.execute(Runnable)`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/task/TaskExecutor.html#execute\(java.lang.Runnable\)).
		Specified by:
		`submit` in interface `AsyncTaskExecutor`
		Parameters:
		`task` - the `Runnable` to execute (never `null`)
		Returns:
		a Future representing pending completion of the task
	- ### submit
		public <T> [Future](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/Future.html "class or interface in java.util.concurrent") <T> submit([Callable](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/Callable.html "class or interface in java.util.concurrent") <T> task)
		Description copied from interface: `AsyncTaskExecutor`
		Submit a Callable task for execution, receiving a Future representing that task. The Future will return the Callable's result upon completion.
		As of 6.1, this method comes with a default implementation that delegates to [`TaskExecutor.execute(Runnable)`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/task/TaskExecutor.html#execute\(java.lang.Runnable\)).
		Specified by:
		`submit` in interface `AsyncTaskExecutor`
		Parameters:
		`task` - the `Callable` to execute (never `null`)
		Returns:
		a Future representing pending completion of the task
	- ### doExecute
		protected void doExecute([Runnable](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Runnable.html "class or interface in java.lang") task)
		Template method for the actual execution of a task.
		The default implementation creates a new Thread and starts it.
	- ### newThread
		protected [Thread](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Thread.html "class or interface in java.lang") newThread([Runnable](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Runnable.html "class or interface in java.lang") task)
		Create a new Thread for the given task.
		Parameters:
		`task` - the Runnable to create a Thread for
		Returns:
		the new Thread instance
		Since:
		6.1
		See Also:
	- ### close
		public void close()
		This close method tracks the termination of active threads if a concrete [`task termination timeout`](#setTaskTerminationTimeout\(long\)) has been set. Otherwise, it is not necessary to close this executor.