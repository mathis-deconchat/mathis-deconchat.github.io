---
title: "Stresstest"
date: 2023-08-21T10:59:09+02:00
draft: false
icon: "code_blocks"
weight: 100

---
# Goal 
The goal of this page is to be able to test your server's performance, and see how it reacts to a high load. Also, have a report about it. 
I won't go into monitoring tools since they have a whole module about them.

## Stress
Stress is a simple tool to stress your server. It's available on most Linux distributions.

```bash
sudo apt install stress
```

As a example, we can first look at the load average of our server with `uptime`

```bash
❯ uptime
 11:05:42 up 4 days, 15:58,  1 user,  load average: 5,23, 3,79, 2,08
```

Then launch a simple stress test with `stress`

```bash
stress --cpu 16 --io 4 --vm 4 --vm-bytes 1024M --timeout 10s
```

You can determine the number of cpu of your machine using `lscpu`

- `io` : number of workers for writing and reading on disk
- `vm` : number of workers for allocating memory
- `vm-bytes` : amount of memory to allocate

After the test, we can see that the load average has increased a lot

```bash 
❯ uptime
 11:08:38 up 4 days, 16:01,  1 user,  load average: 8,50, 5,33, 2,94
```
