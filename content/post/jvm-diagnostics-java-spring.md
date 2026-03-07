---
title: "How I Prepared Our Java Spring App Servers to Capture JVM Diagnostics During a CPU Spike"
date: 2026-03-07T13:03:06Z
draft: false
tags: [JVM, Java, Spring, Tomcat, DevOps, Diagnostics, Performance, Heap Dump, Thread Dump]
categories: [DevOps, Java, Performance]
thumbnail: "images/jvm-diagnostics.png"
summary: "A practical DevOps walkthrough on how to safely capture JVM heap dumps and thread dumps during a CPU spike on a Java Spring/Tomcat production server — using a shell script built around jmap, jstack, and jcmd."
---

## The Problem

When our care web app started experiencing CPU spikes on the production Tomcat servers, the software engineering team needed answers fast. What was the JVM doing at that exact moment? Was memory the problem? Were threads getting stuck? I needed a repeatable, safe way to capture a snapshot of the JVM mid-spike - without making things worse.

---

## A Bit of Background - The JVM

Our care web app is a Java Spring application running on Apache Tomcat. When Tomcat starts, it runs inside the **JVM - the Java Virtual Machine**. Think of the JVM as the engine keeping the application alive. It manages memory, runs threads to handle incoming requests, and periodically cleans up unused objects through **Garbage Collection (GC)**.

---

## The Heap - Where Memory Lives

Inside the JVM, all Java objects are stored in a region of memory called the **heap**. Two numbers define it:

- **Heap capacity** - the maximum memory the JVM is allowed to use, set via the `-Xmx` flag. On our servers this was fixed at **22 GB**, reserved from physical RAM at startup.
- **Heap in use** - how much is actually occupied by live objects right now.

```
Heap capacity:  22 GB   (the ceiling - fixed at startup)
Heap in use:    15 GB   (what is actually used right now)
Usage:          68%
```

When heap in use climbs too close to the capacity ceiling, the Garbage Collector works harder and harder to free space. That GC pressure is one of the most common causes of CPU spikes in Java applications.

---

## Threads - the Workers Inside the JVM

The JVM also manages **threads** - Tomcat spins up one per incoming HTTP request. Under pressure, threads can end up stuck:

- **RUNNABLE** - actively executing
- **WAITING** - idle, waiting to be woken up
- **BLOCKED** - waiting for another thread to release a lock

A CPU spike often means a large number of threads in RUNNABLE or BLOCKED state - the JVM thrashing rather than making progress.

---

## The Two Diagnostic Captures

To understand what the JVM is doing during a spike, there are two standard captures:

**Heap dump** - a complete snapshot of everything in heap memory at a single point in time. Every object, its size, and what is holding a reference to it.

**Thread dump** - a snapshot of every thread and what it is doing at that exact moment.

Together they give the engineering team the full picture of JVM state during an incident.

---

## Estimating the Heap Dump File Size

Before capturing anything, it is worth knowing how large the file will be:

> **Heap dump size ≈ heap currently in use at time of capture**

The dump only captures occupied memory - not the full 22 GB capacity. I checked the live heap on each server before doing anything:

```bash
sudo /usr/lib/jvm/jdk-19.0.1/bin/jcmd <PID> GC.heap_info
```

Which returned:

```
garbage-first heap   total 23068672K, used 15966281K
```

That is 22 GB total, 15.2 GB in use - so the dump file would be approximately 15 GB. Knowing this upfront meant I could confirm there was enough disk space before proceeding.

---

## Where These Tools Come From

The tools used to capture dumps - `jmap`, `jstack`, and `jcmd` - are not third-party software. They are bundled inside the **JDK (Java Development Kit)** installed on the server:

```
/usr/lib/jvm/jdk-19.0.1/bin/jmap     <- takes heap dumps
/usr/lib/jvm/jdk-19.0.1/bin/jstack   <- takes thread dumps
/usr/lib/jvm/jdk-19.0.1/bin/jcmd     <- checks heap info, triggers GC
```

One thing worth knowing: the JRE (Java Runtime Environment) is enough to *run* a Java app, but it does not include these diagnostic tools. Only the full JDK does. Confirming the JDK is present is the first check before any diagnostic work.

---

## The Diagnostic Script

Rather than running commands manually under pressure during a live incident, I built a shell script to handle the full capture process safely and consistently. The script is available on my GitHub - [jvm_diagnostics.sh](https://github.com/Mik3asg/devops-sre-scripts/blob/98ed63846a47739242a2eb1a0a6ed6febcaeec27/jvm_diagnostics.sh).

At a high level it does the following:

**Step 1 - Verify Tomcat is running.** Detects the process, grabs the PID, and locates `jmap`, `jstack`, and `jcmd` automatically. Aborts if anything is missing.

**Step 2 - Check disk space.** Verifies the filesystem has enough room before writing anything. Aborts if usage is above the safety threshold.

**Step 3 - Check heap usage.** Queries the live JVM, calculates heap usage percentage, and estimates the dump file size. If the heap is critically full it skips the heap dump to protect the JVM - but still takes the thread dump.

**Step 4 - Human checkpoint.** Pauses and shows a summary before doing anything irreversible. Only proceeds on an explicit `yes`.

**Step 5 - Heap dump.** Writes the full memory snapshot to disk as a timestamped `.hprof` file.

**Step 6 - Thread dumps.** Captures three snapshots thirty seconds apart for a view of thread behaviour over time.

**Step 7 - Blocked thread scan.** Automatically scans the thread dump for BLOCKED threads and surfaces them on screen immediately.

**Step 8 - Summary.** Prints file locations, sizes, and the exact `scp` command to copy files off the server.

---

## Testing on Non-Production First

Before touching production, I ran the script on our dev server to validate the full flow - tool detection, dump output, safety checks, and file sizes. It also gave me a feel for the brief JVM pause that occurs during a heap dump, which is important to be aware of on a live server.

---

## Deploying to Production

Once validated, I deployed the script to all four production Tomcat servers. For now the script sits under the home directory:

```
/home/<user>/jvm_diagnostics.sh
```

This can be moved to a more permanent location such as `/opt/tomcat/bin/` once the team agrees on a standard. Before deploying I checked each server individually - disk space, current heap usage, and JDK tool availability - to confirm everything was ready.

The servers were now prepared. The next time a CPU spike occurred, the team had a single script to run to safely capture the full JVM state without guesswork.

---

## Getting the Files Off the Server

Once captured, the dump files need to be transferred off the VM for the software engineers to analyse. The heap dump is a **binary file** - raw memory written directly to disk, not human-readable. It requires a specialist tool such as **Eclipse MAT (Memory Analyzer Tool)** to open and interpret. The thread dump is plain text and can be read or shared directly.

The script outputs the exact `scp` command to copy files to a local machine:

```bash
scp user@tomcat-01:/tmp/heapdump-tomcat-01-20260306.hprof .
scp user@tomcat-01:/tmp/threaddump-tomcat-01-20260306.txt .
```

---

## What Comes Next

My role as a DevOps Engineer stops here - getting the captures clean, complete, and safely off the servers. The deeper analysis belongs to the software engineering team, who need to look into the dumps from an application level perspective to understand what the code was doing and where the root cause lives.

When the spike hits, the last thing you want is to be figuring out tooling under pressure. That groundwork is done.

---