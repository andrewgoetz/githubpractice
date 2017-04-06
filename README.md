#### Instructions:
An enterprise software company sells a program that prints out 100,000 random
integers. We have placed 2 copies of this program in your home directory. With
version 1 of the software (RandomGenerator1), the time it takes to print those
100,000 integers is pretty fast. Unfortunately inadequate performance testing
has caused the company to ship RandomGenerator2 that runs much more slowly.

#### First, run the 2 programs and verify they indeed print 100,000 integers. How can you be sure?

I ran each program and redirected the output to a file. I then used `wc -l` on each output file to check the number of lines. Each file had 100,000 lines, which means that they do indeed print exactly 100,000 integers. Later into my investigation, once I found the source code of the files (in `ubuntu/`), I was also able to read the `for` loops of each file to determine they should both produce the expected result.

These commands were:

```
interview@ip-10-0-0-88:~$ java RandomGenerator1 > rand1output.txt
interview@ip-10-0-0-88:~$ wc -l rand1output.txt
100000 rand1output.txt
interview@ip-10-0-0-88:~$ java RandomGenerator2 > rand2output.txt
interview@ip-10-0-0-88:~$ wc -l rand2output.txt
100000 rand2output.txt
```

#### Next, compare the running time of the two versions of the software. Is the second one actually slower? If so, how much slower? Is it slow all the time or just some of the time?

I used the verbose `time` module to record execution time and details of each program. I ran each program multiple times, and `RandomGenerator1` was always very fast, often executing in sub-second speeds.

`RandomGenerator2` on the other hand is much, much slower. This program often took 5-6 minutes to complete, with the fastest time of 3 minutes. The variability of the execution time was interesting.

At this point I also noticed that `RandomGenerator2` seemed to hang before printing anything to the console, which indicated to me that whatever is causing the slowdown is called before the `for` loop is executed.

Two examples of the timing I performed are below:

```
/usr/bin/time -v java RandomGenerator1
Command being timed: "java RandomGenerator1"
	User time (seconds): 0.31
	System time (seconds): 0.24
	Percent of CPU this job got: 78%
	Elapsed (wall clock) time (h:mm:ss or m:ss): 0:00.71
	Average shared text size (kbytes): 0
	Average unshared data size (kbytes): 0
	Average stack size (kbytes): 0
	Average total size (kbytes): 0
	Maximum resident set size (kbytes): 42780
	Average resident set size (kbytes): 0
	Major (requiring I/O) page faults: 0
	Minor (reclaiming a frame) page faults: 4912
	Voluntary context switches: 142
	Involuntary context switches: 162230
	Swaps: 0
	File system inputs: 0
	File system outputs: 64
	Socket messages sent: 0
	Socket messages received: 0
	Signals delivered: 0
	Page size (bytes): 4096
	Exit status: 0
```

```
/usr/bin/time -v java RandomGenerator2
Command being timed: "java RandomGenerator2"
	User time (seconds): 0.59
	System time (seconds): 0.32
	Percent of CPU this job got: 0%
	Elapsed (wall clock) time (h:mm:ss or m:ss): 6:28.55
	Average shared text size (kbytes): 0
	Average unshared data size (kbytes): 0
	Average stack size (kbytes): 0
	Average total size (kbytes): 0
	Maximum resident set size (kbytes): 54228
	Average resident set size (kbytes): 0
	Major (requiring I/O) page faults: 0
	Minor (reclaiming a frame) page faults: 5787
	Voluntary context switches: 8272
	Involuntary context switches: 182238
	Swaps: 0
	File system inputs: 0
	File system outputs: 280
	Socket messages sent: 0
	Socket messages received: 0
	Signals delivered: 0
	Page size (bytes): 4096
	Exit status: 0
```


#### Finally, try to find the root cause as to why the second program runs more slowly. Is it the software itself? The system? Something else? All of the above?

Coming from a stronger software background, I decided to first try to take a look at the code. I opened each each `.class` file with `javap -v`, and it was pretty apparent that there were changes between the files (even by looking at bytecode). However, this didn’t seem like a simple, one-line change or bug.

It was obviously difficult to read bytecode so I found the following instructions to help translate it https://en.wikipedia.org/wiki/Java_bytecode_instruction_listings.

The file above was not much help, so I started researching how to use a decompiler to make these `.class` files readable. After a few minutes of research, I felt that installing and using a decompiler seemed a little too involved, and instead of trying to install a program I am not familiar with on a system that I may not own, I started poking around in the system instead.

This is when I found the location of `RandomGenerator1.java` and `RandomGenerator2.java` files. I was relieved to be able to see some readable code.

After comparing both program’s codebases, it does appear that there is a simple addition to the `RandomGenerator2` program, the `SecureRandom` library.

I started researching the `SecureRandom` library for known issues, especially pertaining to Java 1.7, but was unable to find anything specific, until I read that `SecureRandom` uses `/dev/random` or `/dev/urandom` to create random values based on the system entropy data. 

On some systems, this might not be an issues as keyboard and mouse input can create a lot of system entropy data. However, on remote servers with no inputs, it may be hard to create entropy. The other caveat of `SecureRandom` is that it might utilize `/dev/random` which will block until there is enough entropy to create the values. This means it is possible that using the `SecureRandom` library may block for an unknown amount of time. 

This is starting to line up with my earlier hunch, something is blocking the `for` loop from executing.

Based on my research, a few hundred to a few thousand bytes would be a “normal” amount of entropy. I used the following command to get a measurement from the system and most times I measured it was under 50, which is much lower than the ideal use-case for `SecureRandom`.

```
cat /proc/sys/kernel/random/entropy_avail
```

The lack of entropy for `SecureRandom` to utilize is what I believe to be the root cause of the issue. This also means that the cause of the issue is likely a combination of both the software and the system.

After researching further, it turns out that there may be a [bug with how `/dev/urandom` is set](http://bugs.java.com/view_bug.do?bug_id=6202721) in the `java.security` file, and simply running the `RandomGenerator2` program with the following command will override the setting and allow the program to run at the expected speeds.

```
java -Djava.security.egd=file:/dev/./urandom RandomGenerator2
```

This change overrides the existing values and ensures that the program actually uses the expected `/dev/urandom` file instead of the blocking `/dev/random`.

At this point, I am able to run `RandomGenerator2` in under a second. This seems sufficient enough to present to the customer.


## Response to the Customer


Hello Customer, 

Thank you for taking the time to provide the details to our team so that we could investigate the change in behavior between `RandomGenerator1` and `RandomGenerator2`. We understand how critical the performance of the `RandomGenerator2` program is to your company, which is why we took a deep dive into what could be causing this issue. 

After thorough investigation into possible causes of `RandomGenerator2` executing much slower than `RandonGenerator1`, we believe we can attribute this to be the addition of the `SecureRandom` library in `RandomGenerator2`. While this library can help ensure the random numbers are generated in a more secure way, the method in which the `SecureRandom` library creates its random values may be detrimental to performance and execution time in certain cases.

This is due to how the library generates values based on random data collected in the system’s `/dev/urandom` and `/dev/random` files. How this might create a performance impact is that calls to `/dev/random` will **block** when there is not enough entropy data available. The time spent blocking is longer when there is a lack of entropy, as the system will wait until there is enough to proceed. This issue compounds itself on dedicated Linux servers or hosts as Linux uses keyboard and mouse input to generate entropy.

We believe you may be encountering a bug related to this to the report linked below:

http://bugs.java.com/view_bug.do?bug_id=6202721

---

The good news is that this bug has a solution. This issue is related to `/dev/random` always being used even when `/dev/urandom` is selected in the `java.security` configuration file.

To see if that is the case, you can run the `RandomGenerator2` program with the following java system property to verify if the execution time improves:

```
java -Djava.security.egd=file:/dev/./urandom RandomGenerator2
```

Alternatively, you can change the `securerandom.source` value to `/dev/./urandom` in the `java.security` configuration file to get a similar result.

We know there was likely an important reason that the change was made to use the `SecureRandom` library, so we suggest that you verify the validity of the reported issue as well as the output of the program with this change, however based off of our testing thus far, the results do appear promising.

We will leave it up to you and your team on how to best proceed and which design choices or trade-offs you would like to consider, but we can offer some suggestions. 

- Address the potential bug in the `java.security` file where setting the `securerandom.source` to `/dev/urandom` is not actually working as expected and instead should be `/dev/./urandom`.
- Reevaluate your use-case and decided if you have a need for extremely secure random numbers. If not, it may be best to use `Random` instead of `SecureRandom`.
- Generate entropy in a different way with a tool like `rng-tools` or simply generating I/O with large find operations. 

We hope this explanation of our investigation helps you correct the performance issue. As always, please do follow up with us with any additional questions or concerns. If you do make the suggested changes, we would also like to hear how it goes.

Thank you and we look forward to your response,

Andrew Goetz
