

### event_base结构

event_base是libevent中的总的管理结构，包含了所有的IO多路调度、事件管理、事件回调功能，具体包含以下数据对象：
1. IO多路模式select、epoll等的封装；
2. 循环检测事件的运行控制位；
3. 已经处于active的事件列表；
4. 超时事件列表/堆；
5. map映射，fd-event, signal-event;
6. 推迟执行的回调列表；
7. 所有事件列表；
8. event_base线程控制相关信息；


### 创建event_base
```
struct event_base* event_base_new(void);
```

(1) 创建 event_config 对象
```
struct event_config *cfg = event_config_new();
```
(2) 创建 event_base 对象
```
base = event_base_new_with_config(cfg);
```

```
struct event_base* event_base_new_with_config(const struct event_config* cfg);
```
1. 初始化一些变量的默认值；
2. 选择底层IO多路模型，libevent源代码 configure阶段会从脚本中生成event-config.h，其中定义了系统支持的模型，注意这里会调用模型的封装结构的init方法对event_base进行初始化（IO模型统一的封装方式）；
3. 这里会初始化event_base结构体中的一对socketpair，th_notify_fd，并将事件th_notify绑定到fd[0]读上，添加到event_base调度，通过向fd[1]写用于激发事件；
