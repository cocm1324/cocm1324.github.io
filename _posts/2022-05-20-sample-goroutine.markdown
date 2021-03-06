---
layout: post
title:  "Get to Know How Goroutine Works"
date:   2022-05-20 14:00:00 +0900
categories: project
---
> TL; DR
>
> - utilized goroutine to implement worker-pool simulation
> - use context to control terminations of goroutine   
>
> ![normal](/assets/images/normal-case.png)
> 
  

I'm planning to implement [system-view](https://github.com/cocm1324/system-view) with Go. Main point of this project is to simulate various system component such as server, cache, loadbalancer, etc. Since I decided to use Go here, it seems using goroutine for each component would do the work.

Before jump right into project, I have to be get used to goroutine. So I started this [sample project](https://github.com/cocm1324/sample-goroutine). This document is brief explanation of this project.

## Goroutine? 

```go
go func() {
    log.Printf("other goroutine\n")
}()

log.Printf("main goroutine\n")
```
Goroutine is kind of concurrent feature similar to threads (but not the same thing). In the middle of the Go code, use keyword 'go' to start goroutine.  

```go
// blocking channel example. 
// main goroutine waits for channel c to be input from nested goroutine.
c := make(chan bool)
go func() {
    time.Sleep(time.Second)
    c <- true
}()
result := <- c // wait here
close(c) // closing channel
```

For the communication between goroutine, channel can be used.
Channel has input side and output side, input side will input data to channel, output side waits 

```go
// non-blocking channel example
c := make(chan bool)
go func() {
    time.Sleep(time.Second)
    c <- true
}

L:
for {
    select { // 'select' instead of 'switch'
    case result := <- c:
        // do something using result
        break L
    default: // if c doesn't output, default will be executed
        // do something here while waiting
    }
}
// do something after channel receives

close(c)
```

Channel usually blocks output side, which is blocking channel. However, there is a technique called non-blocking channle. By using 'select' in the loop, it will execute codes in default until channel inputs.

Using goroutine and channels, I will implement some kind of worker simulation.


## Worker Simulation  

I will implement a simple simulation pool and workers. There will be multiple workers and a pool. Pool will manage worker's lifetime such as starting, kill during execution, and receving done from workers. Workers will do task that takes some time (maybe 1 or 2 seconds). After work is done, worker will produce done to pool.

|![simple](/assets/images/normal-case.png)|![kill](/assets/images/kill-case.png)|
|:---:|:---:|
|Normal Case|Kill Case|


### Starting Worker and Wait for Done


### Send Kill using Context



## What I've learned