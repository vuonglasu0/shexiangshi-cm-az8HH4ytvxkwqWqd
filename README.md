具体调试代码参考github：[https://github.com/hggzhang/CppTest/tree/master](https://github.com)

# 概述

在程序设计中，我们希望关联程度低的对象之间的联系是“松耦合”的，也即减少直接依赖。一般的做法是使用消息机制进行信息的传递和响应，其中事件系统是其一种常规手段之一，下面我们尝试使用C++实现一个事件系统。

## 观察者模式

假如你想买一份报纸：

1. 不停的去邮局问今天的报纸到了嘛？到了嘛？到了嘛？... （轮询模式）
2. 去邮局登记要买今天的报纸，之后就回家；邮局报纸到了之后，邮局按照登记名单送报纸。（观察者模式）

对比这两种方式，相对你而言是不是方法2更加高效？你不必一直在邮局等着一直问报纸到了没，登记信息回家看电视等着就好了。所谓观察者模式基本就是这种思想，其核心要素有：

1. 订阅消息：对某个感兴趣的消息A进行登记。
2. 发布消息：“相关单位”发布消息A，通知所有登记者该消息内容。

当然，这只是主题思想，在程序上实现观察者模式还需要考虑一些其他因素，下面将会进行介绍。

# 运行测试

```
void event_test_main()
{
	auto lmd1 = [](const EventPosition& e) {
		LOG_VAR(e.X);
		LOG_VAR(e.Y);
	};

	auto register1 = std::make_shared();
	int id1 = register1->Sub(lmd1);
	EventBus::GetInst().Pub(EventPosition(10, 20));
	register1->UnSub(id1);
	EventBus::GetInst().Pub(EventPosition(20, 20));
}
```

![局部截取_20250908_090400](https://img2024.cnblogs.com/blog/2905902/202509/2905902-20250908090406586-2077639708.png)

# 程序总体设计

参考上述邮局的例子，我们来介绍下程序的核心“角色”（类）：

* Event 报纸
* EventBus 邮局
* EventListener 买报纸的人

另外还需要其他辅助

* EventRegister 类似秘书或助手，帮助管理订阅相关事务

总结下各个角色的相关职责划分：

* Event 事件信息载体，比如鼠标移动事件包含{x, y}的坐标
* EventBus 事件总线，管理事件的订阅/发布
* EventListener 事件观察者，包含其关心的事件的类型，事件的响应函数，自身的ID
  + ID：由于一个事件可能有多个观察者，用于比较确定我们自身是哪个
* EventRegister 管理EventListener，对事件相关行为进行封装，避免让每个使用者都实现一遍。

# 程序细节设计

这里结合C++语言特性和程序的特点说明下一些需要注意的细节：

## Event

[https://github.com/hggzhang/CppTest/blob/master/Program/Event.h](https://github.com)
使用多态来实现不同的事件。

```
class EventBase
{
public:
	virtual ~EventBase() = default;
};

class EventPosition : public EventBase
{
public:
	EventPosition(int x, int y) :X(x), Y(y) {}
	int X;
	int Y;
};
```

## EventBus

[https://github.com/hggzhang/CppTest/blob/master/Program/EventBus.h](https://github.com):[nuts坚果](https://zhongshanyuan.com)

```
class EventBus : public Singleton
{
	friend class Singleton;
	// ...
}
```

* Singleton 使用全局单例提高效率

```
std::unordered_map>> subers;
```

* 使用type\_index作为key来存储监听者列表，相对于字符串（需使用者定义）和typeid更加稳定
* 监听者列表使用list容器，添加和删除效率更好
* 使用弱指针weak\_ptr来缓存监听者避免循环依赖

```
#ifdef ENABLE_EVENTBUS_MULTI_THREAD
	mutable std::mutex mutex;
#endif
```

* 使用编译开关控制线程锁

```
template<typename EventT>
void Pub(const EventT& event)
{
	// prevent infinite recursion
	static thread_local int depth = 0;
	constexpr int MAX_DEPTH = 10;
	// ...
	std::vector> validListeners;
	// ...
	std::type_index typeIndex = typeid(EventT);
	// ...
}
```

* 使用模板方法来发布事件，typeid(EventT)计算标识符Key更加高效
* depth和validListeners副本列表避免循环，如发布事件的回调里油订阅了事件等

## EventListener

```
class EventListenerBase
{
public:
	int ID = 0;
	std::type_index Key = typeid(void);
};

template<typename EventT>
class EventListener : public EventListenerBase
{	
public:
	std::function<void(const EventT&)> callback;
	EventListener( std::function<void(const EventT&)> InCB)
		:callback(std::move(InCB))
	{
		Key = typeid(EventT);
	};
};
```

* 使用type\_index作为标识符相比string和typename更高效可靠
* 由于事件的监听者可能有多个，使用ID用于标识监听者，方便查找
* std::move 转移内存控制权，减少拷贝

## EventRegister

```
class EventRegister
{
private:
#ifdef ENABLE_EVENTBUS_MULTI_THREAD
	std::atomic<int> id = 0;
#else
	int id = 0;
#endif
public:
	std::vector> listeners;
	// ...
}
```

* 封装管理EventListener
* 启用多线程时，需要原子化变量

# 代码清单

参考github地址：[https://github.com/hggzhang/CppTest/tree/master](https://github.com)
部分代码在此贴出

## Event.h

```
class EventBase
{
public:
	virtual ~EventBase() = default;
};

class EventPosition : public EventBase
{
public:
	EventPosition(int x, int y) :X(x), Y(y) {}
	int X;
	int Y;
};

class EventKeyPress : public EventBase
{
public:
	EventKeyPress(char key) :Key(key) {}
	char Key;
};
```

## EventBus.h

```
#pragma once

#include 
#include 
#include 
#include 
#include 
#include 


#include 
#include 
#include "Event.h"
#include "TSingleton.h"

#define ENABLE_EVENTBUS_MULTI_THREAD

#ifdef ENABLE_EVENTBUS_MULTI_THREAD
#include 
#include 
#endif

class EventBus;

class EventListenerBase
{
public:
	int ID = 0;
	std::type_index Key = typeid(void);
};

template<typename EventT>
class EventListener : public EventListenerBase
{	
public:
	std::function<void(const EventT&)> callback;
	EventListener( std::function<void(const EventT&)> InCB)
		:callback(std::move(InCB))
	{
		Key = typeid(EventT);
	};
};

class EventBus : public Singleton
{
	friend class Singleton;
private:

#ifdef ENABLE_EVENTBUS_MULTI_THREAD
	mutable std::mutex mutex;
#endif

	// we use type_index as the key to store different event types, they more safe than using typeid(T) as string, 
	// and use weak_ptr to avoid circular reference
	std::unordered_map>> subers;
public:
	void Sub(std::shared_ptr lsner)
	{
#ifdef ENABLE_EVENTBUS_MULTI_THREAD
		std::lock_guard lock(mutex);
#endif

		std::type_index typeIndex = lsner->Key;
		if (subers.find(typeIndex) == subers.end())
		{
			subers[typeIndex] = std::list>();
		}

		subers[typeIndex].push_back(lsner);
	}

	void UnSub(std::shared_ptr lsner)
	{
#ifdef ENABLE_EVENTBUS_MULTI_THREAD
		std::lock_guard lock(mutex);
#endif
		std::type_index typeIndex = lsner->Key;
		auto it = subers.find(typeIndex);
		if (it != subers.end())
		{
			auto& vec = subers[typeIndex];
			// we need to compare the raw pointer, because the shared_ptr in vec is different from lsner
			vec.erase(std::remove_if(vec.begin(), vec.end(), 
				[&](const std::weak_ptr& wp) {
					auto sp = wp.lock();
					return !sp || sp.get() == lsner.get();
				}),
				vec.end());

			if (vec.empty()) {
				subers.erase(it);
			}
		}
	}
	
	template<typename EventT>
	void Pub(const EventT& event)
	{
		// prevent infinite recursion
		static thread_local int depth = 0;
		constexpr int MAX_DEPTH = 10;

		if (depth >= MAX_DEPTH) {
			// log error
			return;
		}

		depth++;
		std::vector> validListeners;

		{
			#ifdef ENABLE_EVENTBUS_MULTI_THREAD
			std::lock_guard lock(mutex);
			#endif

			// get valid ones and prevent callbacks from modifying the subers map during iteration
			// we cant't put validListeners outside, because Pub might be called recursively

			std::type_index typeIndex = typeid(EventT);
			if (subers.find(typeIndex) != subers.end())
			{
				auto& list = subers[typeIndex];
				for (auto it = list.begin(); it != list.end(); )
				{
					if (auto lsner = it->lock())
					{
						validListeners.push_back(lsner);

						++it;
					}
					else
					{
						it = list.erase(it);
					}
				}
			}
		}

		for (auto& lsner : validListeners) {
			try
			{
				auto derivedLsner = std::static_pointer_cast>(lsner);
				if (derivedLsner)
				{
					derivedLsner->callback(event);
				}
				else
				{
					// log error
					continue;
				}
			}
			catch (...)
			{
				// log error
				continue;
			}
		}

		depth--;
	}
};

class EventRegister
{
private:
#ifdef ENABLE_EVENTBUS_MULTI_THREAD
	std::atomic<int> id = 0;
#else
	int id = 0;
#endif
public:
	std::vector> listeners;

	template<typename EventT>
	int Sub(std::function<void(const EventT&)> callback)
	{
		auto& bus = EventBus::GetInst();
		auto listener = std::make_shared>(callback);
		listener->ID = ++id;
		bus.Sub(listener);
		listeners.push_back(std::move(listener));
		return id;
	}

	void UnSub(int ID)
	{
		auto& bus = EventBus::GetInst();
		auto it = std::remove_if(listeners.begin(), listeners.end(),
			[&](const std::shared_ptr& lsner) {
				if (lsner->ID == ID)
				{
					bus.UnSub(lsner);
					return true;
				}
				return false;
			});
		listeners.erase(it, listeners.end());
	}

	void Clear()
	{
		auto& bus = EventBus::GetInst();
		for (auto& lsner : listeners)
		{
			bus.UnSub(lsner);
		}
		listeners.clear();
	}
};
```
