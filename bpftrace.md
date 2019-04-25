## The bpftrace language

In this lab we will work through some of the key language features of bpftrace that you will need to be familiar with. It is not an exhaustive treatment of the language but just the key concepts. I refer you to the [BPFTrace Reference Guide](https://github.com/iovisor/bpftrace/blob/master/docs/reference_guide.md) for details on all language features.

**bpftrace** is a language that is specifically designed for tracing user and kernel software. As its primary purpose is to facilitate observation of software behaviour it provides a number of key language primitives that enable us to gain detailed insights into the real runtime behaviour of the code we write (which is rarely what we think it actually is!). In this section we will look at the key primitives and techniques which enable us to obtain fresh insights.

## Dynamic Tracing - So What!??

A key attribute of bpftrace is its *dynamic* nature. To understand the myriad complexities and nuances of the execution profile of a modern software stack we tend to go through a cyclical process when reasoning about a systems behaviour (sometimes referred to as the 'Virtuous Circle'):

1. Form a hypothesis based upon our current understanding.
1. Gather data / evidence to prove or disprove our hypothesis
1. Analyse data.
1. Rinse and Repeat until satisfaction is achieved.

The problem with the above sequence is that modifying software to generate the trace data and re-running experiments tends to dominate the time. In production it is often impossible to install such debug binaries and even on anything but trivial development systems it can be painful to do this. In addition to this we rarely capture the data that we need the first time around and it often takes many iterations to gather all the data we ned to debug a problem.

bpftrace solves these problems by allowing us to trivially dynamically modify the system to capture arbitrary data without modifying any code. As modifying the system is so easy to do we can very quickly iterate through different hypothesis and gain novel insights about systemic behaviour in very short periods of time.

## Action Blocks

bpftrace scripts are made up of one or more *Action Blocks*. An action block has 3 parts:

* **A probe**: this is a place of interest where we interrupt the executing thread. There are numerous probe types but examples include the location of a function (e.g., strcmp(3)), an event such as a performance counter overflow event or a periodic timer. The key point here is that this is somewhere where we can collect data.
* **An optional predicate** (sometimes called a *filter*). This is a logical condition which allows us to decide if we are interested in recording data for this event. For example, is the current process named 'hhvm' or is the file we are writing to located in `/tmp`.
* **Actions to record data** . Actions are numerous and mostly capture data that we are interested in. Examples of such actions may be recording the contents of a buffer, capturing a stack trace or simply printing the current time to stdout.

### Starter Example

We'll start with the classic example of looking at system calls (see the [syscalls lab](syscalls.md) for further details).

1. First let's see what system calls are being made:

```
# bpftrace -e 'tracepoint:syscalls:sys_enter_*{@calls[probe] = count();}'
Attaching 314 probes...

^C
<output elided>
@calls[tracepoint:syscalls:sys_enter_pread64]: 119249
@calls[tracepoint:syscalls:sys_enter_munmap]: 152063
@calls[tracepoint:syscalls:sys_enter_mmap]: 156844
@calls[tracepoint:syscalls:sys_enter_close]: 302237
@calls[tracepoint:syscalls:sys_enter_read]: 311111
@calls[tracepoint:syscalls:sys_enter_futex]: 652906
```

Things to note:

* We use a wildcard (`*`) to pattern match all the system call entry probes.
* `@calls[]` declares a special type of associative array (known as a *map* or an *aggregation*). We didn't have to name the associative array as we only have one - we could have just used the `@` sign on its own (known as the `anonymous array`).
* We index the `@calls[]` array using the `probe` builtin variable. This expands to the name of the probe that has been fired (e.g., `tracepoint:syscalls:sys_enter_futex`).
* Each entry in a map can have one of a number of pre-defined functions associated with it. Here the `count()` function simply increments an associated counter every time we hit the probe and we therefore keep count of the number of times a probe has been hit.

**NOTE**: maps are a key data structure that you'll use very frequently!

1. Now let's iterate using the data we just acquired to drill down and discover who is making those `futex` syscalls!

```
# bpftrace -e 'tracepoint:syscalls:sys_enter_futex{@calls[comm] = count();}'
Attaching 1 probe...
^C
<output elided>
@calls[rs:main Q:Reg]: 2999
@calls[cfgator-sub]: 3866
@calls[LadDxChannel]: 5869
@calls[FS_DSSHander_GC]: 7586
```

Things to note:

* We changed are probe specification as we are only interested in futex calls now
* Instead of indexing by the probe name we now index by the name of the process making the futex syscall using the `comm` builtin.

1. Next let's see where in the code the `FS_DSSHander_GC` process is calling the `futex` from:

```
# bpftrace -e 'tracepoint:syscalls:sys_enter_futex/comm == "FS_DSSHander_GC"/{@calls[ustack()] = count();}'
Attaching 1 probe...
^C

@calls[
    __lll_unlock_wake+26
    execute_native_thread_routine+16
    start_thread+222
    __clone+63
]: 20
@calls[
    pthread_cond_timedwait@@GLIBC_2.3.2+870
    std::thread::_State_impl<std::thread::_Invoker<std::tuple<folly::FunctionScheduler::start()::$_2> > >::_M_run()+1625
    execute_native_thread_routine+16
    start_thread+222
    __clone+63
]: 20
@calls[
    syscall+25
    facebook::lad::LadMPSCQueue<folly::CPUThreadPoolExecutor::CPUTask>::add(folly::CPUThreadPoolExecutor::CPUTask)+1099
    folly::CPUThreadPoolExecutor::add(folly::Function<void ()>)+409
    void folly::detail::function::FunctionTraits<void ()>::callSmall<facebook::lad::DataSourceServiceHandler::init()::$_6>(folly::detail::function::Data&)+477
    std::thread::_State_impl<std::thread::_Invoker<std::tuple<folly::FunctionScheduler::start()::$_2> > >::_M_run()+1640
    execute_native_thread_routine+16
    start_thread+222
    __clone+63
]: 507
```

### Exercise

1 write a script to keep count of the number of system calls each process makes.(hint: use the sum() aggregating function)
1 expand the above script to display the per-process system call counts every 10 seconds (hint: use an `interval` timer)
1 add the ability to only display the top 10 per process counts (hint: use the `print` action)
1 delete all per-process syscall stats every 10 secs (hint: `clear`);
1 finally, exit the script after 3 iterations (or 30 seconds if you prefer it that way)


### Associative arrays
