[TOC]

## 一.epoll惊群效应产生的原因

很多朋友都在Linux下使用epoll编写过socket的服务端程序，在多线程环境下可能会遇到epoll的惊群效应。那么什么是惊群效应呢。其产生的原因是什么呢？
在多线程或者多进程环境下，为了提高程序的稳定性，往往会让多个线程或者多个进程同时在epoll_wait监听socket描述符。当一个新的链接请求进来时，操作系统不知道选派那个线程或者进程处理此事件，则干脆将其中几个线程或者进程给唤醒，而实际上只有其中一个进程或者线程能够成功处理accept事件，其他线程都将失败，且errno错误码为EAGAIN。这种现象称为惊群效应，结果是肯定的，惊群效应肯定会带来资源的消耗和性能的影响。
那么如何解决这个问题。

## 二.惊群问题的解决方法

### 2.1多线程环境下解决惊群解决方法

这种情况，不建议让多个线程同时在epoll_wait监听的socket，而是让其中一个线程epoll_wait监听的socket,当有新的链接请求进来之后，由epoll_wait的线程调用accept，建立新的连接，然后交给其他工作线程处理后续的数据读写请求，这样就可以避免了由于多线程环境下的epoll_wait惊群效应问题。

### 2.2多进程下的解决方法

目前很多开源软件，如lighttpd,nginx等都采用master/workers的模式提高软件的吞吐能力及并发能力，在nginx中甚至还采用了负载均衡的技术，在某个子进程的处理能力达到一定负载之后，由其他负载较轻的子进程负责epoll_wait的调用，那么nginx和Lighttpd是如何避免epoll_wait的惊群效用的。
lighttpd的解决思路是无视惊群效应，仍然采用master/workers模式，每个子进程仍然管自己在监听的socket上调用epoll_wait，当有新的链接请求发生时，操作系统仍然只是唤醒其中部分的子进程来处理该事件，仍然只有一个子进程能够成功处理此事件，那么其他被惊醒的子进程捕获EAGAIN错误，并无视。
nginx的解决思路：在同一时刻，永远都只有一个子进程在监听的socket上epoll_wait，其做法是，创建一个全局的pthread_mutex_t，在子进程进行epoll_wait前，则先获取锁。代码如下：

```c++
ngx_int_t  ngx_trylock_accept_mutex(ngx_cycle_t *cycle)  
{
    if (ngx_shmtx_trylock(&ngx_accept_mutex))
    {
        if (ngx_enable_accept_events(cycle) == NGX_ERROR)
        {
            ngx_shmtx_unlock(&ngx_accept_mutex);
            return NGX_ERROR;
        }
        ngx_accept_mutex_held = 1;
        return NGX_OK;
    }
    if (ngx_accept_mutex_held)
    {
        if (ngx_disable_accept_events(cycle) == NGX_ERROR)
            return NGX_ERROR;

        ngx_accept_mutex_held = 0;
    }
    return NGX_OK;
}

//且只有在ngx_accept_disabled < 0 时，才会去获取全局锁，及只有在子进程的负载能力在一定的范围下才会尝试去获取锁，并进入epoll_wait监听的socket。
void  ngx_process_events_and_timers(ngx_cycle_t *cycle)  
{
    if (ngx_accept_disabled > 0)
	{  
		ngx_accept_disabled--;    
	}
	else
	{  
		if (ngx_trylock_accept_mutex(cycle) == NGX_ERROR)
			return;  
     }
}

ngx_accept_disabled = ngx_cycle->connection_n / 8  - ngx_cycle->free_connection_n;
```

表示当子进程的连接数达到连接总数的7/8时，是不会尝试去获取全局锁，只会专注于自己的连接事件请求。
