### 接口
```
int event_base_dispatch(struct event_base *);
```

#### 外围
1. 判断event_base是否已经处于循环中；
2. 缓存的时间清零（作用）；
3. 如果有信号事件添加到event_base，将base部分信息copy一份（为什么）；
4. 写入执行loop的线程id；
5. 将退出标志位置零；


#### 循环
1. 检测是否直接退出（）或break（）；
2. 进行时间修正，更新check time，如果出现时间倒退，要更新超时事件队列/堆里的超时事件(减去某个offset）；
3. 如果没有已经处于激活的事件，对最近的超时事件进行判断：如果超时事件已经小于当前时间，表明事件已经超时(==epoll_wait的超时时间会设为0==)，否则输出一条超时剩余时间的调试信息(==epoll_wait的超时时间会设为改值==)；
4. 如果没有注册事件，loop退出；
5. 更新更新check time为当前，同时缓存的事件清零；

##### 进入实际的dispatch（以epoll为例）
eventop, IO模型统一的结构体封装
- 将监听事件的更改应用到event_base；
- 调用epoll_wait等待事件发生，==设定超时时间==；
- 根据返回的有事件发生的fd数激活事件:
```
for (i = 0; i < res; i++) {
	int what = events[i].events;
	short ev = 0;

	if (what & (EPOLLHUP|EPOLLERR)) {
		ev = EV_READ | EV_WRITE;
	} else {
		if (what & EPOLLIN)
			ev |= EV_READ;
		if (what & EPOLLOUT)
			ev |= EV_WRITE;
	}

	if (!ev)
		continue;

	evmap_io_active(base, events[i].data.fd, ev | EV_ET);
}
    
```
- evmap_io_active(struct event_base *base, evutil_socket_t fd, short events);
- [ ] 从base的fd-event的映射map中取出fd对应的events；
- [ ] 对events中的每一个event，加入event_base的激活事件队列（这里如果事件flag中包含EVLIST_ACTIVE，就不会添加到激活队列中，只是将事件类型保存起来）；
- 分配2倍多的epoll_event，如果这次epoll_wait已经将原来分配的events空间填满；

##### dispatch之后
- 如event_base没有置EVENT_BASE_FLAG_NO_CACHE_TIME，更新event_base缓存的时间；
- 激活已经超时的超时事件；
- [ ] 循环从event_base的最小堆中取根节点，判断是否已经超过当前事件，是的话就激活，不是的话直接退出；
---

- invoke已经激活的事件callback，event_process_active
- [ ] 按照优先级由高到低（数值小优先级高）分别处理激活事件列表，==循环1==；
- 调用event_process_active_single_queue处理每个优先级的单个列表；
- [ ] 根据每个event的ev_closure标志经不同的方式执行回调函数；
- [ ] 如果是EV_CLOSURE_SIGNAL，event_signal_closure内部调用回调，并执行完一次回调后判断event_base的event_break标志位，如果非0直接return；
- [ ] 如果是EV_CLOSURE_PERSIST，event_persist_closure内部会重新将event注册到event_base之后，调用回调；
- [ ] 默认为EV_CLOSURE_NONE， 直接调用回调；

---

- 注意这里，如果有其他线程等待在该event（event_base的当前事件条件变量）上，会向这些线程broadcast；
- 判断标志位，如果event_break，直接return -1（这里并没有检测event_gotterm），如果event_continue，break这个优先级的事件列表循环；
- 正常情况下event_process_active_single_queue返回的是已经执行回调的外部事件数（内部事件如signal），如果该值大于0，则break==循环1==，如果该值小于0，表明立即结束，直接return -1，如果等于0进入下一次循环；
- event_process_deferred_callbacks，处理延后的回调，最多执行
```
#define MAX_DEFERRED 16
```
同样执行完一次回调后会检测event_break,若果置位，直接return -1；

##### 退出event loop的条件
- dispatch执行之前
- [ ] 循环开始检查event_gotterm，event_break，如果被置break循环，将retval保持为0；
- [ ] 如果没有事件注册到event_base且没有处于激活状态的事件，将retval值为1；
- dispatch执行之后
- [ ] dispatch返回-1，将retval值为-1；
- [ ] 执行event_process_active之后，如果event_base_loop调用flag（通过event_base_dispatch调用永远是0）包含EVLOOP_ONCE，且event_base没有处于激活状态的事件,且event_process_active返回值不为0；
- [ ] 如果event_base_loop调用flag（通过event_base_dispatch调用永远是0）包含EVLOOP_NONBLOCK;
 

#### 循环后
- 清理，值标志位，释放锁， 返回retval
```
done:
	clear_time_cache(base);
	base->running_loop = 0;
	
	EVBASE_RELEASE_LOCK(base, th_base_lock);
	return (retval);
```
