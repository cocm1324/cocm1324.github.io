---
layout: post
title:  "Get to Know How Goroutine Works"
date:   2022-05-20 14:00:00 +0900
categories: project
---

I'm planning to implement [system-view](https://github.com/cocm1324/system-view) with Go. Main point of this project is to simulate variouse system component such as server, cache, loadbalancer, etc. Since I decided to use Go here, it seems using goroutine for each component would do.

Before jump right into project, I have to be get used to goroutine. So I started a [sample project](https://github.com/cocm1324/sample-goroutine).

## Goroutine? (skip if you know what goroutine is)
```
go func() {
    log.Printf("other goroutine\n")
}()

log.Printf("main goroutine\n")
```
Goroutine is kind of concurrent feature similar to threads (but not the same thing). In the middle of the Go code, use keyword 'go' to start goroutine.  

```
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

```
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




## What to Make  

Af first, I thought of these
- Task struct represents latency task (i.e it takes some time to complete, maybe 1~2 seconds) 
    - Start() to start goroutine of this task
    - Kill() to stop this goroutine in the middle of execution
    - If task is completed, it signals to Pool struct of its completion.
- Pool struct represents 
    - same as task, Start() and Kill()
    - 
