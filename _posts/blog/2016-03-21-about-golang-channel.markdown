---
layout: post
title: 关于golang的一些使用注意事项
description: 
category: blog
---

golang中的channel经常和goroutine结合使用，最常见的方式是生产者/消费者模式。
一般情况下，必须是生产者调用close，关闭channel，消费者判断channel是否关闭和退出。
对于已经关闭的channel，我们可以继续从channel取出数据，但不能继续往关闭的channel中写数据了；
如果写了，会panic。如果对已经关闭的channel再次调用close，同样也会panic。
如：
<pre>
<code>
func producer(c chan int64, max int) {
    defer
        close(c)

        for i:= 0; i < max; i ++ {
            c <- time.Now().Unix()
        }
}

func consumer(c chan int64) {
    var v int64
        ok := true

        for ok {
            if v, ok = <-c; ok {
                fmt.Println(v)
            }
            // ok == false 时 channel已经被关闭
        }
}
</code>
</pre>

但有时候是需要消费者关闭channel的场景，那我们怎么做呢？我们可以借用sync中的信号量来实现Channel的安全关闭.

<pre>
<code>
type SafeChan struct {
    c chan int 64
    wg sync.WaitGroup
}
func producer(sc *SafeChan, max int) {
    defer
        for i:= 0; i < max; i ++ {
            sc.wg.Add(1)
            sc.c <- time.Now().Unix()
            sc.wg.Done()
        }
}

func consumer(sc *SafeChan, max int) {
    var v int64
        ok := true
        count := 0
        for ok {
            if v, ok = <-sc.c; ok {
                fmt.Println(v)
            }
            if count >= max {
                sc.wg.Wait()
                close(sc.c)
                break
            }
        }
}
</code>
</pre>

下面是完整的可运行代码：
<pre>
<code>
package main

import (
	"fmt"
	"sync"
	"time"
)

func producer(c chan int64, max int) {
	defer close(c)

	for i := 0; i < max; i++ {
		c <- time.Now().Unix()
	}

	fmt.Print("producer quit\n")
}

func consumer(c chan int64) {
	var v int64
	ok := true

	for ok {
		if v, ok = <-c; ok {
			fmt.Println(v)
		}
	}
	fmt.Print("consumer quit\n")
}

type SafeChan struct {
	c  chan int64
	wg sync.WaitGroup
}

func producerEx(sc *SafeChan, max int) {
	for i := 0; i < max; i++ {
		sc.wg.Add(1)
		sc.c <- time.Now().Unix()
		sc.wg.Done()
	}
	fmt.Print("producerEx quit\n")
}

func consumerEx(sc *SafeChan, max int) {
	var v int64
	ok := true
	count := 0
	for ok {
		if v, ok = <-sc.c; ok {
			fmt.Println(v)
		}
		if count >= max {
			sc.wg.Wait()
			close(sc.c)
			break
		}
	}
	fmt.Print("consumerEx quit\n")
}

func main() {
	c := make(chan int64)
	sc := &SafeChan{
		c: make(chan int64),
	}
	max := 100
	go consumer(c)
	go producer(c, max)
	go consumerEx(sc, max)
	go producerEx(sc, max)
	time.Sleep(30 * time.Second)
}

</code>
</pre>

有时候我们希望对一个channel进行循环读取：
<pre>
<code>
for i := range ch { // ch关闭时，for循环会自动结束 
        println(i) 
} 
</code>
</pre>

channel有缓冲和非缓冲之分。channel的读都是阻塞的，channel中有数据时读才会返回；
非缓冲achannel写也是阻塞的，必须等channel读完成之后，才能继续写。
有缓冲的channel写是非阻塞的，但是缓冲满了后则也是阻塞的。
我们有时候希望channel读和写不要一直阻塞，则我们可以加入定时器，用法如下：
<pre><code>
//读的超时设置
select { 
    case <- time.After(time.Second*2): 
        println("read channel timeout") 
    case i := <- ch: 
            println(i) 
} 
//写的超时设置
elect { 
        case <- time.After(time.Second *2): 
                    println("write channel timeout") 
                        case ch <- "hello": 
                            println("write ok") 
}
</code></pre>


