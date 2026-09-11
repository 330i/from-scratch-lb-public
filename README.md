# Overview
The following repository only contains the overview of the load balancer architecture and flow. The code itself is not available publicly due to copyright (Texas A&M Code of Honor) reasons. Code is available upon request for individuals who are not students. As of current, this will be granted for recruitment reference purposes only.

### Architecture
The load balancer will be separated into two distinct components: the load balancer process and the web server cluster. The load balancer process will be responsible for handling incoming requests, distributing them to the web server cluster, and receiving responses from the web server cluster and sending them back to the clients. The web server cluster will be responsible for harboring and managing available web servers. The architecture will resemble the following diagram.

![Load Balancer Architecture Diagram](/docs/resources/architecture.png)

There will be 5 main steps in general.

1. Request handling will read incoming requests. The network switch will direct the request to a web server (in this case, Streaming Server for 'S' requests, Processing Server for 'P' and other requests) if it passes all checks given by the firewall.
2. Request allocation will push the directed request into the request queue. It will also notify the firewall to increment the request count map used for rate limiting.
3. Web server allocation will happen at the same time as request allocation. This will involve the process of monitoring the size of the request queue and allocating or deallocating web servers accordingly.
4. Request processing will pair the first request and available web server in a first-in-first-out basis. This request will be sent to the selected web server for processing.
5. Response handling will receive the response from the web server and send it back to the client. It will also notify the firewall to decrement the request count map used for rate limiting.

### Flow Chart
As request processing will always precede response handling, this will be a single subprocess within the process flow. This means the load balancer process will separate into three different subprocess loops after initialization. This is reflected in the following.

![Load Balancer Flow Chart](/docs/resources/flowchart.png)

As shown by the three outwards arrows from "Web Server Allocator allocates default number of servers." step, this will require the three subprocesses to run in parallel.

- Like what was mentioned in the Architecture section, request allocation will read incoming requests and push them to the request queue. If there are no incoming requests, the request allocator will wait until there are incoming requests.
- The web server allocation subprocess will monitor the size of the request queue. As the goal is to maintain a queue size of 50 to 80 requests, the allocator will allocate one more web server if there are more than 80 requests and deallocate (notify termination then destruct) if there are less than 50 requests. The conditional check will be done every 20 cycles (subject to change).
- Request processing subprocess will pair the first request and available web server in a first-in-first-out basis from the request queue and the web server queue respectively. Although this is technically two different subprocess loops (request serving and request processing), it is represented as one in the flow chart to show the process with respect to a certain request selected for processing.

<br>
<br>

# Deliberate Design Decisions Deviating From Directions
Do note that this isn't to say I completely ignored the "clock cycle" directions. The normal timing mode of the program uses actual time units for the sake of realism. There is a clock cycle mode you can change to for "clock cycle" time units mentioned in the later parts of this section (or in the [Quick Start Guide](@ref quickstart)).

### Usage of Milliseconds for Scaling Check (Instruction 10, "wait n clock cycles and check again.")
The scaling check interval uses milliseconds instead of clock cycles due to each clock cycle taking an infinitesimal amount of time. Due to the highly frequent checking intervals, the rescaling process starts to flood the operating system's process queue, which in turn delays the execution of the web servers.
In order to adjust the scaling check interval to prevent the log files from becoming mostly rescaling messages, a very high number of clock cycles had to be assigned. Which at that point, I decided to use milliseconds with the sleep function.

### Usage of Seconds for Run Time (Deliverable 2, "10000 clock cycles")
As the process splits the client, processing load balancer request handling, processing web server cluster scaling handling, streaming load balancer request handling, streaming web server scaling handling, and each web server into different threads, each thread is having to fight for at most 4 CPU cores to complete as much as they could within 3.57 microseconds (Intel Core i7-1165G7, 2.80 GHz, 10000 cycles). As context switches could take thousands of clock cycles, there is a possibility that the program terminates before the server gets to do anything.

### Usage of Microsecond Since Epoch for Time (Overview, "time (in clock cycles)")
Measuring clock cycles can be done through hardware specific instructions. x86-based processors can use [`__rdtsc()`](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#ig_expand=5274&text=rdt) accessed through compiler intrinsics to read clock cycles. Arm-based processors (Snapdragon X Elite, it's the only non-Apple Arm-based chip for laptops as far as I know) can use the [`pmccntr_el0`](https://docs.qualcomm.com/doc/80-78185-2/topic/performance_optimization.html) register to read clock cycles. However, as it is hardware-specific, doing this within the project's timeline is not feasible. Particularly if someone runs this in MacOS, as evidenced by [here](https://lemire.me/blog/2021/03/24/counting-cycles-and-instructions-on-the-apple-m1-processor/) and [here](https://gist.github.com/ibireme/173517c208c7dc333ba962c1f0d67d12).

### Problems With Using Clock Cycles as a Unit of Time
The usage of clock cycles as a unit of time has particularly been a problem for the early load balancer design as setting "clock cycles" (for loop) for time intervals resulted in a flood of requests and rescaling processes. Even if the interval was set to 10000 cycles. This meant many requests would be sent towards the web server, but fewer requests would actually be processed as they would be overshadowed by the sheer volume of incoming request and scaling check instructions.

In addition, stopping the program at a certain clock cycle cannot be done accurately. Compiler optimizations, CPU architecture, hardware instructions, and context switching make clock cycle measurement unfeasible for this case. And if you were to use a sleep function tied to the CPU's clock cycle, it would actually run longer than what it is instructed to due to operating system scheduling.
This isn't to say it isn't impossible. Although I don't think there's a way to stop a function at a certain clock cycle, operating systems do provide a way for real-time sleep functions in nanosecond precision. For example, in Linux, [`clock_nanosleep()`](https://man7.org/linux/man-pages/man2/clock_nanosleep.2.html) provides a thread sleep in nanosecond precision. In Windows, the [`RtwqSetDeadline`](https://learn.microsoft.com/en-us/windows/win32/api/rtworkq/nf-rtworkq-rtwqsetdeadline) function within the Real-Time Work Queue API within the Multimedia Class Scheduler Service allows you to set nanosecond precision deadlines to a certain process. If you have a fixed CPU clock cycle, it is theoretically possible.

### But Since it is Required
If you want to run this for 10000 clock cycles, you could. Setting the forth command argument to `true` (e.g. `./loadbalancertest 10000 0 normal true`) will change all time units to "clock cycles" (measured through a variable that is being incremented, ~~timed and stopped through loop incrementing volatile variable~~). Setting the fifth command argument to a positive integer will set the starting clock cycle. ~~When looking at the log files, you will notice that a few requests out of requests sent by the client after filling the queue will be pushed to the queue and little to no request will be processed. You will also notice that the interim and final status report will be printed before the initial report. This means 10000 cycles passes so quickly that the server start process is still running even as the server end process is beginning.~~

~~I found 10000000 cycles to be a good number of cycles that balances between doing little work and printing way too much. Although, I did find it too short to get to any requests sent after the request queue was filled. Make sure to change the scaling intervals (5000 was fine) to prevent the logging file from being mostly scaling messages. And when going back to actual time, make sure to change the scaling intervals back to a reasonable value. Else, the process will hang in the end as it waits for the scaling interval sleep to end.~~

As the "clock cycle" using an incremented variable (wait function now directly accesses the clock under clock cycle mode) used in the process takes much longer than the "clock cycle" using an incremented volatile variable, 10000 "clock cycles" can be run in a reasonable way. However, it is still too short to give the web servers an opportunity to process requests sent after the request queue is filled and demonstrate dynamic change in server count (it only has time to scale up to match request queue size). Also, I still would not expect this to stop at exactly 10000 "clock cycles" due to the processes being run concurrently.

<br>
<br>

# Quick Start Guide

### Build
A `makefile` is provided with the ability to run the following commands.
- `make build`: Cleans all logs, object files, and executable and builds executable.
- `make`: Builds executable only. Run this after removing object files.
- `make clean`: Cleans all logs, object files, and executable.
- `make clean-logs`: Cleans all logs.

To simply build the project before running, use `make build` and run `./loadbalancertest` with the required arguments. To rerun, use `make clean-logs` and run `./loadbalancertest` with the required arguments again.

### Configuration (Server)
The configuration file for the server is located in the `/config/server.ini` file. As it is not processed like a typical Windows `.ini` file, it is recommended that you keep the formatting and only change the variable values. Specifically, a section line is only to contain the name encosed by square brackets. A key-value pair is only to include no space unless it is within quotation marks (e.g. `Key=Value` or `Key="Value Data"`). The server will have the following configuration values.
- Firewall threshold (`Threshold` in `Firewall` section): Number of requests that one client IP could make until it is rate limited. Once the threshold is reached, a request deficit is counted. If that deficit is 10 times the threshold, the IP will get banned.
- Blacklist directory (`BlacklistDirectory` in `Firewall` section): Directory to the blacklist in a `.txt` file. Make sure to have one IP per line.
- Verbose mode (`Verbose` in `Server` section): Printing of full logging values within the console. Can only be `true` or `false`. If `false`, only information pertaining to configuration reporting, server status reporting, scaling, client status reporting, and error will be printed.
- Processing server initial web server count (`InitialWebServerCount` in `Server.Processing` section): Initial web server count for the processing server.
- Processing server scaling check interval (`CheckIntervalMilliseconds` in `Server.Processing` section): Rescaling check interval for the processing server. In milliseconds.
- Processing server workload delay (`WorkloadDelay` in `Server.Processing` section): Workload delay for the processing server. This is to simulate the delay caused by computational tasks. In microseconds.
- Streaming server initial web server count (`InitialWebServerCount` in `Server.Streaming` section): Initial web server count for the streaming server.
- Streaming server scaling check interval (`CheckIntervalMilliseconds` in `Server.Streaming` section): Rescaling check interval for the streaming server. In milliseconds.
- Streaming server workload delay (`WorkloadDelay` in `Server.Streaming` section): Workload delay for the streaming server. This is to simulate the delay caused by computational tasks. In microseconds.

If a variable in the configuration file cannot be read, the server will use the default value as listed below.
- Firewall threshold (`Threshold` in `Firewall` section): 100 requests.
- Verbose mode (`Verbose` in `Server` section): `true`.
- Blacklist directory (`BlacklistDirectory` in `Firewall` section): No blacklist.
- Processing server initial web server count (`InitialWebServerCount` in `Server.Processing` section): 20 web servers.
- Processing server scaling check interval (`CheckIntervalMilliseconds` in `Server.Processing` section): 10ms.
- Processing server workload delay (`WorkloadDelay` in `Server.Processing` section): 20us.
- Streaming server initial web server count (`InitialWebServerCount` in `Server.Streaming` section): 20 web servers.
- Streaming server scaling check interval (`CheckIntervalMilliseconds` in `Server.Streaming` section): 10ms.
- Streaming server workload delay (`WorkloadDelay` in `Server.Streaming` section): 20us.

### Configuration (Testing)
Configuration for the `Client` class used in testing will be changed in the command arguments. The executable will have the usage `./loadbalancertest <runTime> <requestMaxDelay> <testMode> <isClockCycleMode> <startTime>` with the following arguments.
- `runtime`: Run time in seconds.
- `requestMaxDelay`: Maximum random time between requests in microseconds.
- `testMode`: Test mode. Set to `normal` for testing normal requests. Set to `dos` for testing denial-of-service attacks.
- `isClockCycleMode`: Clock cycle mode. Set to `true` for activation. Optional argument that is not required to be entered. Only used for special circumstances. When enabling or disabling this mode between tests, make sure to set the check interval to a reasonable value (e.g. if you set `CheckIntervalMilliseconds` to 5000 and change it back to non-clock cycle mode, change it to a value like 50 or the server will keep running for a total of 50 seconds waiting for the check interval to end even after conclusion).
- `startTime`: Clock cycle mode. Positive integer indicating start clock cycle. Optional argument that is not required to be entered. Only used for special circumstances.

### Logging
Logging files will be stored in the `log/` directory. These logs are split between three different categories.
- `general-log.txt`: Information not specific to the processing or streaming server.
- `processing-log.txt`: Information pertaining to the load balancer or web server cluster of the processing server.
- `streaming-log.txt`: Information pertaining to the load balancer or web server cluster of the streaming server.

Do note that these logging files will not show the final status report as the final statement. This is due to the incorporation of a gradual shutdown system to prevent orphan processes and memory faults. I also assume this happens if the OS scheduler runs the client shutdown process and the web server request processing process before the server shutdown process.

Web server responses will be stored in the `output/` directory. The file names are in the format `[client IP].txt`.
