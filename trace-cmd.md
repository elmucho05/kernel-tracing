`ftrace` è un meccanismo di basso livello
- Può tornare utile utilizzare un front end come trace-cmd
- `man trace-cmd`
Nella pagina di manuale, il termine _plugin_ vuol dire tracer
- Approfondiamo poi le opzioni `-p`, `-e` e `-F` di [[#trace-cmd-record(1) — Linux manual page]] e guardiamo solo il primo esempio di uso in `man trace-cmd-report`
# TRACE-CMD(1) Manual Page

## NAME

```
trace-cmd - interacts with Ftrace Linux kernel internal tracer 
```

## SYNOPSIS

```
trace-cmd _COMMAND_ [_OPTIONS_]
```

## DESCRIPTION

The trace-cmd(1) command interacts with the Ftrace tracer that is built inside the Linux kernel. It interfaces with the Ftrace specific files found in the debugfs file system under the tracing directory. A _COMMAND_ must be specified to tell trace-cmd what to do.

## COMMANDS

```
record  - record a live trace and write a trace.dat file to the
          local disk or to the network.
```

```
set     - set a ftrace configuration parameter.
```

```
report  - reads a trace.dat file and converts the binary data to a
          ASCII text readable format.
```

```
stream  - Start tracing and read the output directly
```

```
profile - Start profiling and read the output directly
```

```
hist    - show a histogram of the events.
```

```
stat    - show tracing (ftrace) status of the running system
```

```
options - list the plugin options that are available to *report*
```

```
start   - start the tracing without recording to a trace.dat file.
```

```
stop    - stop tracing (only disables recording, overhead of tracer
          is still in effect)
```

```
restart - restart tracing from a previous stop (only effects recording)
```

```
extract - extract the data from the kernel buffer and create a trace.dat
          file.
```

```
show    - display the contents of one of the Ftrace Linux kernel tracing files
```

```
reset   - disables all tracing and gives back the system performance.
          (clears all data from the kernel buffers)
```

```
clear   - clear the content of the Ftrace ring buffers.
```

```
split   - splits a trace.dat file into smaller files.
```

```
list    - list the available plugins or events that can be recorded.
```

```
listen  - open up a port to listen for remote tracing connections.
```

```
agent   - listen on a vsocket for trace clients
```

```
setup-guest - create FIFOs for tracing guest VMs
```

```
restore - restore the data files of a crashed run of trace-cmd record
```

```
snapshot- take snapshot of running trace
```

```
stack   - run and display the stack tracer
```

```
check-events - parse format strings for all trace events and return
               whether all formats are parseable
```

```
convert   - convert trace files
```

```
attach   - attach a host trace.dat file to a guest trace.dat file
```

```
dump    - read out the meta data from a trace file
```

## OPTIONS

**-h**, --help

Display the help text.

Other options see the man page for the corresponding command.

## SEE ALSO

trace-cmd-record(1), trace-cmd-report(1), trace-cmd-hist(1), trace-cmd-start(1), trace-cmd-stop(1), trace-cmd-extract(1), trace-cmd-reset(1), trace-cmd-restore(1), trace-cmd-stack(1), trace-cmd-convert(1), trace-cmd-split(1), trace-cmd-list(1), trace-cmd-listen(1), trace-cmd.dat(5), trace-cmd-check-events(1), trace-cmd-stat(1), trace-cmd-attach(1)


# [trace-cmd-record(1) — Linux manual page](https://man7.org/linux/man-pages/man1/trace-cmd-record.1.html#top_of_page)


```
  trace-cmd record [_OPTIONS_] [_command_]
```

**Description**
The trace-cmd(1) record command will set up the Ftrace Linux kernel tracer to record the specified plugins or events that happen while the _command_ executes. If no command is given, then it will record until the user hits Ctrl-C.

The record command of trace-cmd will set up the Ftrace tracer to start tracing the various events or plugins that are given on the command line. It will then create a number of tracing processes (one per CPU) that will start recording from the kernel ring buffer straight into temporary files. When the command is complete
(or Ctrl-C is hit) all the files will be combined into a trace.dat file that can later be read (see trace-cmd-report(1)).

**Options**
- `-p` *tracer*
	- Specify a tracer. Tracers usually do more than just trace an event. Common tracers are: **function**, **function_graph**, **preemptirqsoff**, **irqsoff**, **preemptoff** and **wakeup**. A tracer must be supported by the running kernel. To see a list of available tracers, see trace-cmd-list(1).
- `-e`*event*
	- specify an event to trace. Various static trace points have been added to the Linux kernel. They are grouped by subsystem where you can enable all events of a given subsystem or specify specific events to be enabled. The _event_ is of the format "subsystem:event-name". You can also just specify the subsystem without the _:event-name_ or the event-name without the "subsystem:". Using "-e sched_switch" will enable the "sched_switch" event where as, "-e sched" will enable all events under the "sched" subsystem.
- `-F`
	- This will filter only the executable that is given on the command line. If no command is given, then it will filter itself (pretty pointless). Using -F will let you trace only events that are caused by the given command.
- `max-graph-depth depth`
	- Set the maximum depth the function_graph tracer will trace into a function. A value of one will only show where userspace enters the kernel but not any functions called in the kernel. The default is zero, which means no limit.
- `-g` *function-name*
	- This option is for the function_graph plugin. It will graph the given function. That is, it will only trace the function and all functions that it calls. You can have more than one **-g** on the command line.


