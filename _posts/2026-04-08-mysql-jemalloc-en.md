---
title: "Diagnosing MySQL Memory Problems with Jemalloc"
description: Using Jemalloc's sampling profiler to locate MySQL memory leaks and study its memory usage — two real cases, how to enable it, how to take snapshots via gdb or a UDF, and how to generate call graphs.
date: 2026-04-08 10:00:00 +0800
categories: [Database, MySQL]
tags: [MySQL, Memory, Jemalloc, Performance]
toc: true
lang: en
hidden: true
published: false
---

> This article is also available in Chinese: [中文版](/posts/mysql-jemalloc/). Browse [all English articles](/english/).
{: .prompt-tip }

Memory leaks and high memory usage are common problems in MySQL, and diagnosing them depends heavily on good memory-monitoring data. MySQL's Performance Schema provides memory monitoring, but the granularity of PFS's memory statistics is coarse, which makes it hard to pinpoint the exact code responsible.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/ibf4J5w9SFz4qibCyCxtwSASicXk2rlzpar3tmHauB4moaicvcSw0DuonIgtfPWXqStITXNcBVfw9fPefnPVkRBarg/640?wx_fmt=png&from=appmsg)

The figure above shows the memory information monitored by PFS. Although the monitoring shows that a lot of memory is allocated on `String::value` and `thd::main_mem_root`, these are basic objects used in many places throughout MySQL, so we cannot infer from this which operation actually caused the memory usage. In addition, PFS itself has relatively high memory overhead and some performance impact, so many users do not enable it on their instances.

[Jemalloc](https://github.com/jemalloc/jemalloc) is an efficient memory allocator that improves allocation and deallocation performance — and reduces fragmentation — through process- and thread-level memory caching. It also provides a memory monitoring and analysis facility with several features:

- It records allocations and frees by sampling, to minimize the impact on the running process.
- Each sample records the call stack at the point of allocation, so the call stack can pinpoint the allocation precisely.
- The sampling data can be dumped to a file.
- The `jeprof` tool can present the sampled data graphically.
- It can diff the samples from two points in time, so you can focus on the memory allocated during a given interval, which makes analysis easier.

Jemalloc is well suited both to diagnosing memory problems in MySQL and to studying MySQL's memory usage. The two cases below show its memory-analysis capability.

## Case 1 — A Memory Leak in Clone_persist_gtid

[Bug#107991](https://bugs.mysql.com/bug.php?id=107991) is a memory leak that the AliSQL team located using Jemalloc. `Clone_persist_gtid` is a thread introduced in MySQL 8.0 that persists a transaction's GTID into the `mysql.gtid_executed` table. On an instance with very frequent write transactions, this leak appears. Each leak is small, but after several or a dozen days the leaked memory adds up. The figure below is part of the call stack for this bug; the full snapshot file is on the Bug#107991 page for anyone who wants it.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/ibf4J5w9SFz4qibCyCxtwSASicXk2rlzparWibCDhQqeLp8UZj6loNYmABgB6KFTfibrgsVo3ibTvJhUBr3AqQ4VXBlQ/640?wx_fmt=png&from=appmsg)

Because the leaked memory is a very small fraction of the total, this figure is produced by diffing the samples from two points in time. From the call stack, we can clearly see that the memory is allocated when the `Clone_persist_gtid` thread opens a system table. The call stack quickly leads to the cause: the thread's `THD::mem_root` is never cleared, which leaks memory (see Bug#107991 for details). The fix is simple — a single line of code. We fixed it in AliSQL soon after and submitted the patch to the official team, which fixed the issue in the latest release, MySQL 8.0.42.

## Case 2 — Wasted Memory in the AHI

Besides analyzing leaks, Jemalloc is also very useful for studying MySQL's memory usage. For example, right after starting an instance, you can take a snapshot of memory allocation to analyze how memory is allocated at startup. The figure below is part of the memory snapshot of a 64 GB buffer-pool instance right after startup.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/ibf4J5w9SFz4z3HlTS49TWXOKL1jv2zMqWbba8icP1sPibEf2aOmWTPldvOlZp23rSKuNgqGicesZrSDpBpVibgXaQA/640?wx_fmt=png&from=appmsg)

We can see that during buffer-pool initialization, the allocated memory falls into two parts:

- `btr_search_sys_create` allocated 1280 MB for the adaptive hash index (AHI) structures.
- `buf_block_init` allocated the `os_event` structures for each page's rw-lock and mutex in the buffer pool, totaling 1918.8 MB.

Because the AHI has serious stability problems, we disable it by default, and the official team also changed the default to OFF starting in MySQL 8.4. Even so, with the AHI disabled it still uses 1 GB of memory. Reading the code shows that this memory is `1/64` of the buffer-pool size (see [Bug#112223](https://bugs.mysql.com/bug.php?id=112223) for the detailed analysis). That is a fair amount of wasted memory, so AliSQL fixed it: with the AHI disabled, this memory is no longer allocated.

*Why doesn't the figure show the 64 GB used by the buffer pool? Because the buffer pool is allocated with `mmap`, not `malloc`, so it is not tracked by jemalloc.*

## Using Jemalloc with MySQL

After installing libjemalloc on the system, configure it as follows:

```shell
# Add the jemalloc library to LD_PRELOAD
export LD_PRELOAD=</path/jemalloc.so.2>
# Start mysqld
```

If you start MySQL with mysqld_safe, you can configure it in the MySQL config file instead:

```ini
[mysqld_safe]
malloc-lib = </path/jemalloc.so.2>
```

You can check whether mysqld is using jemalloc like this:

```shell
lsof -p <pid> | grep jemalloc
```

## Enabling Jemalloc Profiling

Before starting mysqld, set the following environment variable to enable jemalloc profiling:

```shell
export MALLOC_CONF="prof:true,prof_active:true,prof_prefix:/tmp/mysqld.jedump"
```

- `prof`: enables profiling; can only be set before starting mysqld.
- `prof_active`: sets the profiling state to active; if set to false, allocation information is not recorded.
- `prof_prefix`: sets the directory and filename prefix for the snapshot files. jemalloc appends its own suffix to this prefix (the exact format is shown in the section on generating a call graph below), so do not add a trailing dot yourself, or the filenames get a doubled dot. If `prof_prefix` is not set it defaults to `jeprof`; if it is set to an empty string, `prof.dump` returns failure without writing a file, so always give it a non-empty value.

*Profiling must be enabled at compile time.* If the jemalloc library does not support profiling, you get the following error:

![](https://mmbiz.qpic.cn/sz_mmbiz_png/ibf4J5w9SFz7NRR7TKZ1xDE4uMGDm1rDvw68SkGn25xAD6zm5XDx7MZemPFYrZYlSibFeHqYKuMbEWZdDlBnd7Vw/640?wx_fmt=png&from=appmsg)

In that case you need to recompile a version that supports profiling, adding the `--enable-prof` option at build time.

### Performance Impact

Based on sysbench benchmarks, we can draw roughly the following conclusions:

- With Jemalloc 5.2, `prof:true,prof_active:false` has essentially no performance impact; `prof:true,prof_active:true` causes about a 4% drop under high concurrency.
- With Jemalloc 5.3, neither case has a noticeable performance impact.

These figures are for the default sampling rate. jemalloc records one sample per `2^lg_prof_sample` bytes allocated on average; `lg_prof_sample` defaults to `19`, which is about one sample per 512 KB. Lowering it samples more frequently, giving finer-grained data at the cost of higher overhead; raising it does the opposite. Adjust this knob when you need more accurate profiles or lower overhead.

## Generating Snapshots Automatically

jemalloc offers two ways to generate snapshots automatically:

- `lg_prof_interval`: dump once per amount of memory allocated. The value is a power of two — for example, 30 means 2^30 = 1 GB, so it dumps once every 1 GB.
- `prof_gdump`: dump each time total memory reaches a new high.

```shell
export MALLOC_CONF="prof:true,prof_active:true,lg_prof_interval:30,prof_prefix:/tmp/mysqld.jedump"
export MALLOC_CONF="prof:true,prof_active:true,prof_gdump:true,prof_prefix:/tmp/mysqld.jedump"
```

## Generating Snapshots Manually

Automatic snapshots are simple, but for a system like MySQL that allocates and frees memory frequently, we prefer to control snapshot generation ourselves. Generating a snapshot manually requires calling the relevant function inside the process. For example, [Percona](https://docs.percona.com/percona-server/8.0/jemalloc-profiling.html#use-percona-server-for-mysql-with-jemalloc-with-profiling-enabled) integrates a command into the server for this.

If you are on community MySQL, how do you generate a snapshot? Here are two manual approaches: one via gdb, suited to development environments or ad-hoc use, and one via a UDF, which is better suited to production.

## Generating a Snapshot with gdb

This approach uses gdb to call the `mallctl` function to generate a snapshot. In this case you do not need to set `lg_prof_interval`.

> Every `mallctl` call in the gdb scripts below is cast to `(int)`. This is required when jemalloc is the stripped library installed from a distribution package (`apt install libjemalloc2`, `yum install jemalloc`): gdb then has only the minimal symbols from `.dynsym` and no DWARF debug information, so it cannot determine `mallctl`'s return type and refuses to call it with the error `'mallctl' has unknown return type; cast the call to its declared return type`. The explicit `(int)` cast supplies the return type. A jemalloc built from source with `-g` and left unstripped already carries the type information, but the cast is harmless there, so the scripts include it unconditionally.
{: .prompt-warning }

```shell
define jeprof_dump
  p (int) mallctl("prof.dump", 0, 0, 0, 0)
end

jeprof_dump
```

Copy the above into a file (jemalloc.gdb), then run the script with gdb:

```shell
gdb -p <pid> -x jemalloc.gdb -batch
```

You can also control the `prof_active` state dynamically and check its status with the following scripts:

```shell
define jeprof_status
  set $backup_opt_help = opt_help
  set $backup_opt_tc_log_size = opt_tc_log_size
  set opt_tc_log_size = sizeof(opt_help)

  call (int) mallctl("opt.prof", &opt_help, &opt_tc_log_size, 0, 0)
  printf "opt.prof is %d\n", opt_help

  call (int) mallctl("prof.active", &opt_help, &opt_tc_log_size, 0, 0)
  printf "prof.active is %d\n", opt_help

  set opt_help = $backup_opt_help
  set opt_tc_log_size = $backup_opt_tc_log_size
end
jeprof_status
```

```shell
define jeprof_off
  set $backup_opt_help = opt_help
  set opt_help = 0

  p (int) mallctl("prof.active", 0, 0, &opt_help, sizeof(bool))

  set opt_help = $backup_opt_help
end
jeprof_off
```

```shell
define jeprof_on
  set $backup_opt_help = opt_help
  set opt_help = 1

  p (int) mallctl("prof.active", 0, 0, &opt_help, sizeof(bool))

  set opt_help = $backup_opt_help
end
jeprof_on
```

These scripts borrow the process variables `opt_help` and `opt_tc_log_size` as scratch space; any writable variables of a suitable type would do. Both `jeprof_on` and `jeprof_off` pass `&opt_help` as the new value (`newp`) for `prof.active`, so `opt_help` must be set to the intended value first — `1` to enable, `0` to disable — and then restored. `jeprof_on` sets it to `1` and `jeprof_off` sets it to `0`; do not rely on the variable's current value, or the command may enable profiling when you meant to disable it.

## Generating a Snapshot with a UDF

The gdb approach is somewhat hacky and not suitable for automation. It also interrupts the process, which can cause the instance to stall. MySQL provides a loadable-function mechanism, commonly called a [UDF](https://dev.mysql.com/doc/extending-mysql/8.4/en/adding-loadable-function.html). Through it, we can implement the functionality above in C in a shared library and load it dynamically into a running instance. The code is as follows:

```cpp
#include <jemalloc/jemalloc.h>
#include <string.h>
#include <stdbool.h>
/* The following is for user defined functions */

#define PLUGIN_EXPORT
#define longlong long
#define my_bool bool

enum Item_result {STRING_RESULT=0, REAL_RESULT, INT_RESULT, ROW_RESULT,
                  DECIMAL_RESULT};

typedef struct st_udf_args
{
  unsigned int arg_count;		/* Number of arguments */
  enum Item_result *arg_type;		/* Pointer to item_results */
  char **args;				/* Pointer to argument */
  unsigned long *lengths;		/* Length of string arguments */
  char *maybe_null;			/* Set to 1 for all maybe_null args */
  char **attributes;                    /* Pointer to attribute name */
  unsigned long *attribute_lengths;     /* Length of attribute arguments */
  void *extension;
} UDF_ARGS;

/* This holds information about the result */

typedef struct st_udf_init
{
  my_bool maybe_null;          /* 1 if function can return NULL */
  unsigned int decimals;       /* for real functions */
  unsigned long max_length;    /* For string functions */
  char *ptr;                   /* free pointer for function data */
  my_bool const_item;          /* 1 if function always returns the same value */
  void *extension;
} UDF_INIT;

// Return whether profiling was compiled in and enabled (opt.prof): 1 if on, 0 if off; NULL on error
PLUGIN_EXPORT my_bool
jeprof_prof_status_init(UDF_INIT *initid, UDF_ARGS *args, char *message)
{
    if (args->arg_count != 0) {
        strcpy(message, "Usage: prof_opt_status()");
        return 1;
    }
    return 0;
}

PLUGIN_EXPORT longlong
jeprof_prof_status(UDF_INIT *initid, UDF_ARGS *args, char *is_null, char *error)
{
    bool enabled;
    size_t len = sizeof(enabled);

    if (mallctl("opt.prof", &enabled, &len, NULL, 0)) {
        *error = 1;
        return -1;
    }
    return enabled ? 1 : 0;
}

// Return the prof.active state: 1 if active, 0 if not; NULL on error
PLUGIN_EXPORT my_bool
jeprof_active_status_init(UDF_INIT *initid, UDF_ARGS *args, char *message)
{
    if (args->arg_count != 0) {
        strcpy(message, "Usage: prof_active_status()");
        return 1;
    }
    return 0;
}

PLUGIN_EXPORT longlong
jeprof_active_status(UDF_INIT *initid, UDF_ARGS *args, char *is_null, char *error)
{
    bool active;
    size_t len = sizeof(active);

    if (mallctl("prof.active", &active, &len, NULL, 0)) {
        *error = 1;
        return -1;
    }
    return active ? 1 : 0;
}

// Enable profiling: set prof.active to true; returns 0 on success, NULL on error
PLUGIN_EXPORT my_bool
jeprof_enable_init(UDF_INIT *initid, UDF_ARGS *args, char *message)
{
    if (args->arg_count != 0) {
        strcpy(message, "Usage: prof_enable()");
        return 1;
    }
    return 0;
}

PLUGIN_EXPORT longlong
jeprof_enable(UDF_INIT *initid, UDF_ARGS *args, char *is_null, char *error)
{
    bool enable = true;

    if (mallctl("prof.active", NULL, NULL, &enable, sizeof(enable))) {
        *error = 1;
        return -1;
    }
    return 0;
}

// Disable profiling: set prof.active to false; returns 0 on success, NULL on error
PLUGIN_EXPORT my_bool
jeprof_disable_init(UDF_INIT *initid, UDF_ARGS *args, char *message)
{
    if (args->arg_count != 0) {
        strcpy(message, "Usage: prof_disable()");
        return 1;
    }
    return 0;
}

PLUGIN_EXPORT longlong
jeprof_disable(UDF_INIT *initid, UDF_ARGS *args, char *is_null, char *error)
{
    bool enable = false;
    if (mallctl("prof.active", NULL, NULL, &enable, sizeof(enable))) {
        *error = 1;
        return -1;
    }
    return 0;
}

// Dump the memory profile; returns 0 on success, NULL on error
PLUGIN_EXPORT my_bool
jeprof_dump_init(UDF_INIT *initid, UDF_ARGS *args, char *message)
{
    if (args->arg_count != 0) {
        strcpy(message, "Usage: prof_dump()");
        return 1;
    }
    return 0;
}

PLUGIN_EXPORT longlong
jeprof_dump(UDF_INIT *initid, UDF_ARGS *args, char *is_null, char *error)
{
    int ret = mallctl("prof.dump", NULL, NULL, NULL, 0);
    if(ret) {
        *error = 1;
        return ret;
    }
    return 0;
}
```

Compiling this code depends on jemalloc's header files, so install them first. With yum:

```bash
yum install jemalloc-devel
```

Then compile with the following command, and copy the resulting jemalloc_udf.so to the instance's [plugin_dir](https://dev.mysql.com/doc/refman/8.4/en/server-system-variables.html#sysvar_plugin_dir):

```bash
gcc -shared -fPIC -o jemalloc_udf.so jeprof_udf.c
```

> This command deliberately does not link jemalloc, so `mallctl` stays undefined in `jemalloc_udf.so` (`nm -D --undefined-only jemalloc_udf.so` shows `U mallctl`). The symbol is resolved at run time from the jemalloc that mysqld has already loaded through `LD_PRELOAD`. As a result, `CREATE FUNCTION` succeeds only on an instance that is running with jemalloc preloaded; on an instance without it, the load fails with `Can't open shared library ... undefined symbol: mallctl`. Do not statically link a second copy of jemalloc into this library.
{: .prompt-warning }

Before using these UDFs, load them with the following SQL:

```sql
CREATE FUNCTION jeprof_dump RETURNS INTEGER SONAME 'jemalloc_udf.so';
CREATE FUNCTION jeprof_enable RETURNS INTEGER SONAME 'jemalloc_udf.so';
CREATE FUNCTION jeprof_disable RETURNS INTEGER SONAME 'jemalloc_udf.so';
CREATE FUNCTION jeprof_prof_status RETURNS INTEGER SONAME 'jemalloc_udf.so';
CREATE FUNCTION jeprof_active_status RETURNS INTEGER SONAME 'jemalloc_udf.so';
```

You can then call them via `SELECT jeprof_xxx()`, as shown:

![](https://mmbiz.qpic.cn/sz_mmbiz_png/ibf4J5w9SFz7NRR7TKZ1xDE4uMGDm1rDvd4NonVULtibYficzsnxUNDUY9sN7GUQpFlHZ9xUQpovFucJw1p2n8yiaA/640?wx_fmt=png&from=appmsg)

- `jeprof_prof_status` returns the state of the `prof` option: 1 if on, 0 if off.
- `jeprof_active_status` returns the state of `prof.active`: 1 if on, 0 if off.
- `jeprof_enable` sets `prof.active` to `true`; returns 0 on success.
- `jeprof_disable` sets `prof.active` to `false`; returns 0 on success.
- `jeprof_dump` generates a memory snapshot file; returns 0 on success.

If the underlying `mallctl` call fails, each of these functions sets the UDF error flag, so the `SELECT` returns `NULL` rather than a numeric code.

## Generating a Profiling Call Graph

jemalloc includes a `jeprof` command-line tool that displays the information in a snapshot. A common use is generating a call graph. The snapshot data can be rendered as SVG, PDF, and other formats, as shown:

![](https://mmbiz.qpic.cn/sz_mmbiz_png/ibf4J5w9SFz7NRR7TKZ1xDE4uMGDm1rDvf7IPGcffqqyQ2zPyW47rMfsc0ibicZcRKM9Aut8zicRSrDDsG4H7bKMZw/640?wx_fmt=png&from=appmsg)

jemalloc names each snapshot file `<prefix>.<pid>.<seq>.<type><seq>.heap`, where the type character is `m` for a manual dump (`prof.dump`), `i` for an interval dump (`lg_prof_interval`), and `g` for a gdump (`prof_gdump`). With the prefix `/tmp/mysqld.jedump` used above, an actual file therefore looks like `/tmp/mysqld.jedump.4211.0.m0.heap`. Substitute the real filenames from your dump directory in the commands below.

The command to generate the graph:

```shell
jeprof ./sql/mysqld /tmp/mysqld.jedump.4211.1.m1.heap -svg > jedump.svg
```

And to diff two snapshots:

```shell
jeprof ./sql/mysqld --base /tmp/mysqld.jedump.4211.0.m0.heap /tmp/mysqld.jedump.4211.1.m1.heap -svg > jedump_diff.svg
```

## Conclusion

Jemalloc provides powerful memory-analysis capabilities. When it records allocation information, it also captures the call stack, and that information helps us locate memory problems quickly and precisely. Through MySQL's UDF mechanism, jemalloc's snapshot capability can be integrated into MySQL conveniently and enabled and used online. Because jemalloc collects the statistics by sampling, the overhead at the default sampling rate is low — negligible on Jemalloc 5.3, and only a few percent on 5.2 even under high concurrency — so in most cases it can be enabled with confidence.
