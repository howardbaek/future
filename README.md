

<div id="badges"><!-- pkgdown markup -->
<a href="https://CRAN.R-project.org/web/checks/check_results_future.html"><img border="0" src="https://www.r-pkg.org/badges/version/future" alt="CRAN check status"/></a> <a href="https://github.com/HenrikBengtsson/future/actions?query=workflow%3AR-CMD-check"><img border="0" src="https://github.com/HenrikBengtsson/future/workflows/R-CMD-check/badge.svg?branch=develop" alt="Build status"/></a>  <a href="https://ci.appveyor.com/project/HenrikBengtsson/future"><img border="0" src="https://ci.appveyor.com/api/projects/status/github/HenrikBengtsson/future?svg=true" alt="Build status"/></a> <a href="https://codecov.io/gh/HenrikBengtsson/future"><img border="0" src="https://codecov.io/gh/HenrikBengtsson/future/branch/develop/graph/badge.svg" alt="Coverage Status"/></a> 
</div>

# future: Unified Parallel and Distributed Processing in R for Everyone <img border="0" src="man/figures/logo.png" alt="The 'future' hexlogo" align="right"/>

## Introduction
The purpose of the [future] package is to provide a very simple and uniform way of evaluating R expressions asynchronously using various resources available to the user.

In programming, a _future_ is an abstraction for a _value_ that may be available at some point in the future.  The state of a future can either be _unresolved_ or _resolved_.  As soon as it is resolved, the value is available instantaneously.  If the value is queried while the future is still unresolved, the current process is _blocked_ until the future is resolved.  It is possible to check whether a future is resolved or not without blocking.  Exactly how and when futures are resolved depends on what strategy is used to evaluate them.  For instance, a future can be resolved using a sequential strategy, which means it is resolved in the current R session.  Other strategies may be to resolve futures asynchronously, for instance, by evaluating expressions in parallel on the current machine or concurrently on a compute cluster.

Here is an example illustrating how the basics of futures work.  First, consider the following code snippet that uses plain R code:
```r
> v <- {
+   cat("Hello world!\n")
+   3.14
+ }
Hello world!
> v
[1] 3.14
```
It works by assigning the value of an expression to variable `v` and we then print the value of `v`.  Moreover, when the expression for `v` is evaluated we also print a message.

Here is the same code snippet modified to use futures instead:
```r
> library("future")
> v %<-% {
+   cat("Hello world!\n")
+   3.14
+ }
> v
Hello world!
[1] 3.14
```
The difference is in how `v` is constructed; with plain R we use `<-` whereas with futures we use `%<-%`.  The other difference is that output is relayed _after_ the future is resolved (not during) and when the value is queried (see Vignette 'Outputting Text').

So why are futures useful?  Because we can choose to evaluate the future expression in a separate R process asynchronously by simply switching settings as:
```r
> library("future")
> plan(multisession)
> v %<-% {
+   cat("Hello world!\n")
+   3.14
+ }
> v
Hello world!
[1] 3.14
```
With asynchronous futures, the current/main R process does _not_ block, which means it is available for further processing while the futures are being resolved
in separate processes running in the background.  In other words, futures provide a simple but yet powerful construct for parallel and / or distributed processing in R.


Now, if you cannot be bothered to read all the nitty-gritty details about futures, but just want to try them out, then skip to the end to play with the Mandelbrot demo using both parallel and non-parallel evaluation.



## Implicit or Explicit Futures

Futures can be created either _implicitly_ or _explicitly_.  In the introductory example above we used _implicit futures_ created via the `v %<-% { expr }` construct.  An alternative is _explicit futures_ using the `f <- future({ expr })` and `v <- value(f)` constructs.  With these, our example could alternatively be written as:
```r
> library("future")
> f <- future({
+   cat("Hello world!\n")
+   3.14
+ })
> v <- value(f)
Hello world!
> v
[1] 3.14
```

Either style of future construct works equally(*) well.  The implicit style is most similar to how regular R code is written.  In principle, all you have to do is to replace `<-` with a `%<-%` to turn the assignment into a future assignment.  On the other hand, this simplicity can also be deceiving, particularly when asynchronous futures are being used.  In contrast, the explicit style makes it much clearer that futures are being used, which lowers the risk for mistakes and better communicates the design to others reading your code.

(*) There are cases where `%<-%` cannot be used without some (small) modifications.  We will return to this in Section 'Constraints when using Implicit Futures' near the end of this document.



To summarize, for explicit futures, we use:

* `f <- future({ expr })` - creates a future
* `v <- value(f)` - gets the value of the future (blocks if not yet resolved)

For implicit futures, we use:

* `v %<-% { expr }` - creates a future and a promise to its value

To keep it simple, we will use the implicit style in the rest of this document, but everything discussed will also apply to explicit futures.



## Controlling How Futures are Resolved
The future package implements the following types of futures:

| Name            | OSes        | Description
|:----------------|:------------|:-----------------------------------------------------
| _synchronous:_  |             | _non-parallel:_
| `sequential`    | all         | sequentially and in the current R process
| `transparent`   | all         | as sequential w/ early signaling and w/out local (for debugging)
| _asynchronous:_ |             | _parallel_:
| `multisession`  | all         | background R sessions (on current machine)
| `multicore`     | not Windows/not RStudio | forked R processes (on current machine)
| `cluster`       | all         | external R sessions on current, local, and/or remote machines
| `remote`        | all         | Simple access to remote R sessions

_Comment:_ The alias strategy `multiprocess` was deprecated in future (>= 1.20.0) in favor of `multisession` and `multicore`.

The future package is designed such that support for additional strategies can be implemented as well.  For instance, the [future.callr] package provides future backends that evaluates futures in a background R process utilizing the [callr] package - they work similarly to `multisession` futures but has a few advantages.  Continuing, the [future.batchtools] package provides futures for all types of _cluster functions_ ("backends") that the [batchtools] package supports.  Specifically, futures for evaluating R expressions via job schedulers such as Slurm, TORQUE/PBS, Oracle/Sun Grid Engine (SGE) and Load Sharing Facility (LSF) are also available.

By default, future expressions are evaluated eagerly (= instantaneously) and synchronously (in the current R session).  This evaluation strategy is referred to as "sequential".  In this section, we will go through each of these strategies and discuss what they have in common and how they differ.


### Consistent Behavior Across Futures
Before going through each of the different future strategies, it is probably helpful to clarify the objectives of the Future API (as defined by the future package).  When programming with futures, it should not really matter what future strategy is used for executing code.  This is because we cannot really know what computational resources the user has access to so the choice of evaluation strategy should be in the hands of the user and not the developer.  In other words, the code should not make any assumptions on the type of futures used, e.g. synchronous or asynchronous.

One of the designs of the Future API was to encapsulate any differences such that all types of futures will appear to work the same.  This despite expressions may be evaluated locally in the current R session or across the world in remote R sessions.  Another obvious advantage of having a consistent API and behavior among different types of futures is that it helps while prototyping.  Typically one would use sequential evaluation while building up a script and, later, when the script is fully developed, one may turn on asynchronous processing.

Because of this, the defaults of the different strategies are such that the results and side effects of evaluating a future expression are as similar as possible.  More specifically, the following is true for all futures:

* All _evaluation is done in a local environment_ (i.e. `local({ expr })`) so that assignments do not affect the calling environment.  This is natural when evaluating in an external R process, but is also enforced when evaluating in the current R session.

* When a future is constructed, _global variables are identified_.  For asynchronous evaluation, globals are exported to the R process/session that will be evaluating the future expression.  For sequential futures with lazy evaluation (`lazy = TRUE`), globals are "frozen" (cloned to a local environment of the future).  Also, in order to protect against exporting too large objects by mistake, there is a built-in assertion that the total size of all globals is less than a given threshold (controllable via an option, cf. `help("future.options")`).  If the threshold is exceeded, an informative error is thrown.

* Future _expressions are only evaluated once_.  As soon as the value (or an error) has been collected it will be available for all succeeding requests.

Here is an example illustrating that all assignments are done to a local environment:
```r
> plan(sequential)
> a <- 1
> x %<-% {
+     a <- 2
+     2 * a
+ }
> x
[1] 4
> a
[1] 1
```


Now we are ready to explore the different future strategies.


### Synchronous Futures

Synchronous futures are resolved one after another and most commonly by the R process that creates them.  When a synchronous future is being resolved it blocks the main process until resolved.  There are two types of synchronous futures in the future package, _sequential_ and _transparent_.


#### Sequential Futures
Sequential futures are the default unless otherwise specified.  They were designed to behave as similar as possible to regular R evaluation while still fulfilling the Future API and its behaviors.  Here is an example illustrating their properties:
```r
> plan(sequential)
> pid <- Sys.getpid()
> pid
[1] 8000
> a %<-% {
+     pid <- Sys.getpid()
+     cat("Future 'a' ...\n")
+     3.14
+ }
> b %<-% {
+     rm(pid)
+     cat("Future 'b' ...\n")
+     Sys.getpid()
+ }
> c %<-% {
+     cat("Future 'c' ...\n")
+     2 * a
+ }
Future 'a' ...
> b
Future 'b' ...
[1] 8000
> c
Future 'c' ...
[1] 6.28
> a
[1] 3.14
> pid
[1] 8000
```
Since eager sequential evaluation is taking place, each of the three futures is resolved instantaneously in the moment it is created.  Note also how `pid` in the calling environment, which was assigned the process ID of the current process, is neither overwritten nor removed.  This is because futures are evaluated in a local environment.  Since synchronous (uni-)processing is used, future `b` is resolved by the main R process (still in a local environment), which is why the value of `b` and `pid` are the same.


#### Transparent Futures
For troubleshooting, _transparent_ futures can be used by specifying `plan(transparent)`.  A transparent future is technically a sequential future with instant signaling of conditions (including errors and warnings) and where evaluation, and therefore also assignments, take place in the calling environment.  Transparent futures are particularly useful for troubleshooting errors that are otherwise hard to narrow down.


### Asynchronous Futures
Next, we will turn to asynchronous futures, which are futures that are resolved in the background.  By design, these futures are non-blocking, that is, after being created the calling process is available for other tasks including creating additional futures.  It is only when the calling process tries to access the value of a future that is not yet resolved, or trying to create another asynchronous future when all available R processes are busy serving other futures, that it blocks.


#### Multisession Futures
We start with multisession futures because they are supported by all operating systems.  A multisession future is evaluated in a background R session running on the same machine as the calling R process.  Here is our example with multisession evaluation:
```r
> plan(multisession)
> pid <- Sys.getpid()
> pid
[1] 8000
> a %<-% {
+     pid <- Sys.getpid()
+     cat("Future 'a' ...\n")
+     3.14
+ }
> b %<-% {
+     rm(pid)
+     cat("Future 'b' ...\n")
+     Sys.getpid()
+ }
> c %<-% {
+     cat("Future 'c' ...\n")
+     2 * a
+ }
Future 'a' ...
> b
Future 'b' ...
[1] 8079
> c
Future 'c' ...
[1] 6.28
> a
[1] 3.14
> pid
[1] 8000
```
The first thing we observe is that the values of `a`, `c` and `pid` are the same as previously.  However, we notice that `b` is different from before.  This is because future `b` is evaluated in a different R process and therefore it returns a different process ID.


When multisession evaluation is used, the package launches a set of R sessions in the background that will serve multisession futures by evaluating their expressions as they are created.  If all background sessions are busy serving other futures, the creation of the next multisession future is _blocked_ until a background session becomes available again.  The total number of background processes launched is decided by the value of `availableCores()`, e.g.
```r
> availableCores()
mc.cores 
       2 
```
This particular result tells us that the `mc.cores` option was set such that we are allowed to use in total two (2) processes including the main process.  In other words, with these settings, there will be two (2) background processes serving the multisession futures.  The `availableCores()` is also agile to different options and system environment variables.  For instance, if compute cluster schedulers are used (e.g. TORQUE/PBS and Slurm), they set specific environment variable specifying the number of cores that was allotted to any given job; `availableCores()` acknowledges these as well.  If nothing else is specified, all available cores on the machine will be utilized, cf. `parallel::detectCores()`.  For more details, please see `help("availableCores", package = "parallelly")`.


#### Multicore Futures
On operating systems where R supports _forking_ of processes, which is basically all operating system except Windows, an alternative to spawning R sessions in the background is to fork the existing R process.  To use multicore futures, when supported, specify:

```r
plan(multicore)
```

Just like for multisession futures, the maximum number of parallel processes running will be decided by `availableCores()`, since in both cases the evaluation is done on the local machine.

Forking an R process can be faster than working with a separate R session running in the background.  One reason is that the overhead of exporting large globals to the background session can be greater than when forking, and therefore shared memory, is used.  On the other hand, the shared memory is _read only_, meaning any modifications to shared objects by one of the forked processes ("workers") will cause a copy by the operating system.  This can also happen when the R garbage collector runs in one of the forked processes.

On the other hand, process forking is also considered unstable in some R environments.  For instance, when running R from within RStudio process forking may resulting in crashed R sessions.  Because of this, the future package disables multicore futures by default when running from RStudio.  See `help("supportsMulticore")` for more details.


#### Cluster Futures
Cluster futures evaluate expressions on an ad-hoc cluster (as implemented by the parallel package).  For instance, assume you have access to three nodes `n1`, `n2` and `n3`, you can then use these for asynchronous evaluation as:
```r
> plan(cluster, workers = c("n1", "n2", "n3"))
> pid <- Sys.getpid()
> pid
[1] 8000
> a %<-% {
+     pid <- Sys.getpid()
+     cat("Future 'a' ...\n")
+     3.14
+ }
> b %<-% {
+     rm(pid)
+     cat("Future 'b' ...\n")
+     Sys.getpid()
+ }
> c %<-% {
+     cat("Future 'c' ...\n")
+     2 * a
+ }
Future 'a' ...
> b
Future 'b' ...
[1] 8164
> c
Future 'c' ...
[1] 6.28
> a
[1] 3.14
> pid
[1] 8000
```


Any types of clusters that `parallel::makeCluster()` creates can be used for cluster futures.  For instance, the above cluster can be explicitly set up as:
```r
cl <- parallel::makeCluster(c("n1", "n2", "n3"))
plan(cluster, workers = cl)
```
Also, it is considered good style to shut down cluster `cl` when it is no longer needed, that is, calling `parallel::stopCluster(cl)`.  However, it will shut itself down if the main process is terminated.  For more information on how to set up and manage such clusters, see `help("makeCluster", package = "parallel")`.
Clusters created implicitly using `plan(cluster, workers = hosts)` where `hosts` is a character vector will also be shut down when the main R session terminates, or when the future strategy is changed, e.g. by calling `plan(sequential)`.

Note that with automatic authentication setup (e.g. SSH key pairs), there is nothing preventing us from using the same approach for using a cluster of remote machines.




### Nested Futures and Evaluation Topologies
This far we have discussed what can be referred to as "flat topology" of futures, that is, all futures are created in and assigned to the same environment.  However, there is nothing stopping us from using a "nested topology" of futures, where one set of futures may, in turn, create another set of futures internally and so on.

For instance, here is an example of two "top" futures (`a` and `b`) that uses multisession evaluation and where the second future (`b`) in turn uses two internal futures:
```r
> plan(multisession)
> pid <- Sys.getpid()
> a %<-% {
+     cat("Future 'a' ...\n")
+     Sys.getpid()
+ }
> b %<-% {
+     cat("Future 'b' ...\n")
+     b1 %<-% {
+         cat("Future 'b1' ...\n")
+         Sys.getpid()
+     }
+     b2 %<-% {
+         cat("Future 'b2' ...\n")
+         Sys.getpid()
+     }
+     c(b.pid = Sys.getpid(), b1.pid = b1, b2.pid = b2)
+ }
> pid
[1] 8000
> a
Future 'a' ...
[1] 8220
> b
Future 'b' ...
Future 'b1' ...
Future 'b2' ...
 b.pid b1.pid b2.pid 
  8251   8251   8251 
```
By inspection the process IDs, we see that there are in total three different processes involved for resolving the futures.  There is the main R process
(pid 8000),
and there are the two processes used by `a`
(pid 8220)
and `b`
(pid 8251).
However, the two futures (`b1` and `b2`) that is nested by `b` are evaluated by the same R process as `b`.  This is because nested futures use sequential evaluation unless otherwise specified.  There are a few reasons for this, but the main reason is that it protects us from spawning off a large number of background processes by mistake, e.g. via recursive calls.



To specify a different type of _evaluation topology_, other than the first level of futures being resolved by multisession evaluation and the second level by sequential evaluation, we can provide a list of evaluation strategies to `plan()`.  First, the same evaluation strategies as above can be explicitly specified as:
```r
plan(list(multisession, sequential))
```
We would actually get the same behavior if we try with multiple levels of multisession evaluations;
```r
> plan(list(multisession, multisession))
[...]
> pid
[1] 8000
> a
Future 'a' ...
[1] 8310
> b
Future 'b' ...
Future 'b1' ...
Future 'b2' ...
 b.pid b1.pid b2.pid 
  8341   8341   8341 
```
The reason for this is, also here, to protect us from launching more processes than what the machine can support.  Internally, this is done by setting `mc.cores = 1` such that functions like `parallel::mclapply()` will fall back to run sequentially.  This is the case for both multisession and multicore evaluation.


Continuing, if we start off by sequential evaluation and then use multisession evaluation for any nested futures, we get:
```r
> plan(list(sequential, multisession))
[...]
> pid
[1] 8000
> a
Future 'a' ...
[1] 8000
> b
Future 'b' ...
Future 'b1' ...
Future 'b2' ...
 b.pid b1.pid b2.pid 
  8000   8418   8449 
```
which clearly show that `a` and `b` are resolved in the calling process
(pid 8000)
whereas the two nested futures (`b1` and `b2`) are resolved in two separate R processes
(pids 8418 and 8449).



Having said this, it is indeed possible to use nested multisession evaluation strategies, if we explicitly specify (read _force_) the number of cores available at each level.  In order to do this we need to "tweak" the default settings, which can be done as follows:
```r
> plan(list(tweak(multisession, workers = 2), tweak(multisession, 
+     workers = 2)))
[...]
> pid
[1] 8000
> a
Future 'a' ...
[1] 8500
> b
Future 'b' ...
Future 'b1' ...
Future 'b2' ...
 b.pid b1.pid b2.pid 
  8531   8589   8620 
```
First, we see that both `a` and `b` are resolved in different processes
(pids 8500 and 8531)
than the calling process
(pid 8000).
Second, the two nested futures (`b1` and `b2`) are resolved in yet two other R processes
(pids 8589 and 8620).


For more details on working with nested futures and different evaluation strategies at each level, see Vignette '[Futures in R: Future Topologies]'.


### Checking A Future without Blocking
It is possible to check whether a future has been resolved or not without blocking.  This can be done using the `resolved(f)` function, which takes an explicit future `f` as input.  If we work with implicit futures (as in all the examples above), we can use the `f <- futureOf(a)` function to retrieve the explicit future from an implicit one.  For example,
```r
> plan(multisession)
> a %<-% {
+     cat("Future 'a' ...")
+     Sys.sleep(2)
+     cat("done\n")
+     Sys.getpid()
+ }
> cat("Waiting for 'a' to be resolved ...\n")
Waiting for 'a' to be resolved ...
> f <- futureOf(a)
> count <- 1
> while (!resolved(f)) {
+     cat(count, "\n")
+     Sys.sleep(0.2)
+     count <- count + 1
+ }
1 
2 
3 
4 
5 
> cat("Waiting for 'a' to be resolved ... DONE\n")
Waiting for 'a' to be resolved ... DONE
> a
Future 'a' ...done
[1] 8657
```


## Failed Futures
Sometimes the future is not what you expected.  If an error occurs while evaluating a future, the error is propagated and thrown as an error in the calling environment _when the future value is requested_.  For example, if we use lazy evaluation on a future that generates an error, we might see something like
```r
> plan(sequential)
> b <- "hello"
> a %<-% {
+     cat("Future 'a' ...\n")
+     log(b)
+ } %lazy% TRUE
> cat("Everything is still ok although we have created a future that will fail.\n")
Everything is still ok although we have created a future that will fail.
> a
Future 'a' ...
Error in log(b) : non-numeric argument to mathematical function
```
The error is thrown each time the value is requested, that is, if we try to get the value again will generate the same error (and output):
```r
> a
Future 'a' ...
Error in log(b) : non-numeric argument to mathematical function
In addition: Warning message:
restarting interrupted promise evaluation
```
To see the _last_ call in the call stack that gave the error, we can use the `backtrace()` function(\*) on the future, i.e.
```r
> backtrace(a)
[[1]]
log(a)
```

(\*\) The commonly used `traceback()` does not provide relevant information in the context of futures.  Furthermore, it is unfortunately not possible to see the list of calls (evaluated expressions) that led up to the error; only the call that gave the error (this is due to a limitation in `tryCatch()` used internally).


## Globals
Whenever an R expression is to be evaluated asynchronously (in parallel) or sequentially via lazy evaluation, global (aka "free") objects have to be identified and passed to the evaluator.  They need to be passed exactly as they were at the time the future was created, because, for lazy evaluation, globals may otherwise change between when it is created and when it is resolved.  For asynchronous processing, the reason globals need to be identified is so that they can be exported to the process that evaluates the future.

The future package tries to automate these tasks as far as possible.  It does this with help of the [globals] package, which uses static-code inspection to identify global variables.  If a global variable is identified, it is captured and made available to the evaluating process.
Moreover, if a global is defined in a package, then that global is not exported.  Instead, it is made sure that the corresponding package is attached when the future is evaluated.  This not only better reflects the setup of the main R session, but it also minimizes the need for exporting globals, which saves not only memory but also time and bandwidth, especially when using remote compute nodes.

Finally, it should be clarified that identifying globals from static code inspection alone is a challenging problem.  There will always be corner cases where automatic identification of globals fails so that either false globals are identified (less of a concern) or some of the true globals are missing (which will result in a run-time error or possibly the wrong results).  Vignette '[Futures in R: Common Issues with Solutions]' provides examples of common cases and explains how to avoid them as well as how to help the package to identify globals or to ignore falsely identified globals.  If that does not suffice, it is always possible to manually specify the global variables by their names (e.g. `globals = c("a", "slow_sum")`) or as name-value pairs (e.g. `globals = list(a = 42, slow_sum = my_sum)`).


## Constraints when using Implicit Futures

There is one limitation with implicit futures that does not exist for explicit ones.  Because an explicit future is just like any other object in R it can be assigned anywhere/to anything.  For instance, we can create several of them in a loop and assign them to a list, e.g.
```r
> plan(multisession)
> f <- list()
> for (ii in 1:3) {
+     f[[ii]] <- future({
+         Sys.getpid()
+     })
+ }
> v <- lapply(f, FUN = value)
> str(v)
List of 3
 $ : int 8742
 $ : int 8773
 $ : int 8742
```
This is _not_ possible to do when using implicit futures.  This is because the `%<-%` assignment operator _cannot_ be used in all cases where the regular `<-` assignment operator can be used.  It can only be used to assign future values to _environments_ (including the calling environment) much like how `assign(name, value, envir)` works.  However, we can assign implicit futures to environments using _named indices_, e.g.
```r
> plan(multisession)
> v <- new.env()
> for (name in c("a", "b", "c")) {
+     v[[name]] %<-% {
+         Sys.getpid()
+     }
+ }
> v <- as.list(v)
> str(v)
List of 3
 $ a: int 8832
 $ b: int 8864
 $ c: int 8832
```
Here `as.list(v)` blocks until all futures in the environment `v` have been resolved.  Then their values are collected and returned as a regular list.

If _numeric indices_ are required, then _list environments_ can be used.  List environments, which are implemented by the [listenv] package, are regular environments with customized subsetting operators making it possible to index them much like how lists can be indexed.  By using list environments where we otherwise would use lists, we can also assign implicit futures to list-like objects using numeric indices.  For example,
```r
> library("listenv")
> plan(multisession)
> v <- listenv()
> for (ii in 1:3) {
+     v[[ii]] %<-% {
+         Sys.getpid()
+     }
+ }
> v <- as.list(v)
> str(v)
List of 3
 $ : int 8929
 $ : int 8960
 $ : int 8929
```
As previously, `as.list(v)` blocks until all futures are resolved.



## Demos
To see a live illustration how different types of futures are evaluated, run the Mandelbrot demo of this package.  First, try with the sequential evaluation,
```r
library("future")
plan(sequential)
demo("mandelbrot", package = "future", ask = FALSE)
```
which resembles how the script would run if futures were not used.  Then, try multisession evaluation, which calculates the different Mandelbrot planes using parallel R processes running in the background.  Try,
```r
plan(multisession)
demo("mandelbrot", package = "future", ask = FALSE)
```
Finally, if you have access to multiple machines you can try to set up a cluster of workers and use them, e.g.
```r
plan(cluster, workers = c("n2", "n5", "n6", "n6", "n9"))
demo("mandelbrot", package = "future", ask = FALSE)
```


[batchtools]: https://cran.r-project.org/package=batchtools
[callr]: https://cran.r-project.org/package=callr
[future]: https://cran.r-project.org/package=future
[future.callr]: https://cran.r-project.org/package=future.callr
[future.batchtools]: https://cran.r-project.org/package=future.batchtools
[globals]: https://cran.r-project.org/package=globals
[listenv]: https://cran.r-project.org/package=listenv
[Futures in R: Common Issues with Solutions]: https://cran.r-project.org/web/packages/future/vignettes/future-4-issues.html
[Futures in R: Future Topologies]: https://cran.r-project.org/web/packages/future/vignettes/future-3-topologies.html

## Installation
R package future is available on [CRAN](https://cran.r-project.org/package=future) and can be installed in R as:
```r
install.packages("future")
```


### Pre-release version

To install the pre-release version that is available in Git branch `develop` on GitHub, use:
```r
remotes::install_github("HenrikBengtsson/future", ref="develop")
```
This will install the package from source.  

<!-- pkgdown-drop-below -->


## Contributing

To contribute to this package, please see [CONTRIBUTING.md](CONTRIBUTING.md).

