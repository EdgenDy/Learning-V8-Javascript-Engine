# Understanding How V8 Javascript Engine Really Works
## Introduction

V8 is a javascript engine that powered both chrome browser and nodejs, V8 gets the spotlight and standout among others javascript engine like spidermonkey of mozilla firefox and chakra of Microsoft Edge. In this lesson we will find out how V8 javascript engine really works thru examining some lines from the v8 source codes. We will used a sample code provided by the v8 the `hello-world.cc` from the `samples` directory of v8.

**Note:** v8-7.9.2 is the version of v8 that we will referenced throughout this lesson, you can find it [here](here). 

## Hello World 

`hello-world.cc` is a sample code provided by the v8 team that demonstrate how to execute a very short javascript statement `'hello' + 'world'`, a concatenation of two strings 'hello' and 'world', but before that we need to create and initialize the following objects.
1. Platform
2. Isolate
3. Context
4. String

### 1. Platform 

Platform is an interface member of the namespace `v8`, it is part of the public API of V8. It is used by V8 to get memory page allocation, system clock and threads info. 

```c++
// @location  : include/v8-platform.h

283 | class Platform {
284 |  public:
285 |   virtual ~Platform() = default;
```

#### NewDefaultPlatform
```
hello-world.cc STACK:
  NewDefaultPlatform < main
```
 
```c++
// @location  : samples/hello-world.cc

16 | std::unique_ptr<v8::Platform> platform = v8::platform::NewDefaultPlatform();
17 | v8::V8::InitializePlatform(platform.get());
```
As you've seen on line 16 of the hello-world.cc we have used `v8::platform::NewDefaultPlatform()` function. It returns an instance or implementation of `v8::Platform` wraps inside `std::unique_ptr` object. 

```c++
// @location  : include/libplatform/libplatform.h
// @namespace : v8::platform
// @invoker   : hello-world.cc > main

37 | V8_PLATFORM_EXPORT std::unique_ptr<v8::Platform> NewDefaultPlatform(
38 |     int thread_pool_size = 0,
39 |     IdleTaskSupport idle_task_support = IdleTaskSupport::kDisabled,
40 |     InProcessStackDumping in_process_stack_dumping =
41 |         InProcessStackDumping::kDisabled,
42 |     std::unique_ptr<v8::TracingController> tracing_controller = {});
```

This function takes 4 parameters and each parameter has its default value. Because we invoke this function without supplying any arguments on it, it will used all the default values of these parameters.

#### DefaultPlatform
```
hello-world.cc STACK:
  DefaultPlatform < NewDefaultPlatform < main
```


Inside of this function there is a instantation of the class `v8::platform::DefaultPlatform` using `new` operator that allocates its instance on heap. 

```c++
// @location  : src/libplatform/default-platform.cc
// @namespace : v8::platform
// @invoker   : v8::platform::NewDefaultPlatform

41 | std::unique_ptr<DefaultPlatform> platform(
42 |     new DefaultPlatform(idle_task_support, std::move(tracing_controller)));
```
This class is the default implementation of `v8::Platform` and the returning value of `v8::platform::NewDefaultPlatform()` function, as we learn earlier `v8::Platform` is an interface therefore it needs an implementation of a child class to instantiate it. 

```c++
// @location  : src/libplatform/default-platform.h
// @namespace : v8::platform

32 | class V8_PLATFORM_EXPORT DefaultPlatform : public NON_EXPORTED_BASE(Platform) {
33 |  public:
34 |   explicit DefaultPlatform(
35 |       IdleTaskSupport idle_task_support = IdleTaskSupport::kDisabled,
36 |       std::unique_ptr<v8::TracingController> tracing_controller = {});
37 | 
38 |   ~DefaultPlatform() override;
```

```c++
// @location  : src/libplatform/default-platform.cc
// @namespace : v8::platform

69 | DefaultPlatform::DefaultPlatform(
70 |     IdleTaskSupport idle_task_support,
71 |     std::unique_ptr<v8::TracingController> tracing_controller)
72 |     : thread_pool_size_(0),
73 |       idle_task_support_(idle_task_support),
74 |       tracing_controller_(std::move(tracing_controller)),
75 |       page_allocator_(new v8::base::PageAllocator()),
76 |       time_function_for_testing_(nullptr) {
```
Inside its constructor at line 75 it creates a new instance of `v8::base::PageAllocator()` and passed its pointer value on its variable `page_allocator_`. 


#### PageAllocator
```
hello-world.cc STACK:
  PageAllocator < DefaultPlatform < NewDefaultPlatform < main
```

```c++
// @location  : src/base/page-allocator.h
// @namespace : v8::base

15 | class V8_BASE_EXPORT PageAllocator
16 |     : public NON_EXPORTED_BASE(::v8::PageAllocator) {
17 |  public:
18 |   PageAllocator();
19 |   ~PageAllocator() override = default;
```

```c++
// @location  : src/base/page-allocator.cc
// @namespace : v8::base

27 | PageAllocator::PageAllocator()
28 |     : allocate_page_size_(base::OS::AllocatePageSize()),
29 |       commit_page_size_(base::OS::CommitPageSize()) {}
```

Inside `PageAllocator` constructor it retrieves our memory page size by invoking `v8::base::OS::AllocatePageSize()` function then stored the returned value to its `allocate_page_size_` variable and then finally retrieves our memory commit size by invoking `v8::base::OS::CommitPageSize()` then stored the returned value on its `commit_page_size_` variable.

#### OS

This class has static methods for different platform specific functions. 

```c++
// @location  : src/base/platform/platform.h
// @namespace : v8::base

107 | class V8_BASE_EXPORT OS {

261 |   static size_t AllocatePageSize();

263 |   static size_t CommitPageSize();
```
V8 has many implementations of this class on each different operating systems but for our example we will used the Windows operating system. 

#### AllocatePageSize

```
hello-world.cc STACK:
  AllocatePageSize < PageAllocator < DefaultPlatform < NewDefaultPlatform < main
```

```c++
// @location  : src/base/platform/platform-win32.cc
// @namespace : v8::base::OS

696 | size_t OS::AllocatePageSize() {
697 |   static size_t allocate_alignment = 0;
698 |   if (allocate_alignment == 0) {
699 |     SYSTEM_INFO info;
700 |     GetSystemInfo(&info);
701 |     allocate_alignment = info.dwAllocationGranularity;
702 |   }
703 |   return allocate_alignment;
704 | }
```

This function returns the allocation alignment on our physical memory. It creates a static variable `allocate_alignment` and stored on it the returned value of `dwAllocationGranularity`. It used the windows GetSysInfo function to get the allocation granularity of our physical memory. 

```
hello-world.cc STACK:
  CommitPageSize < PageAllocator < DefaultPlatform < NewDefaultPlatform < main
```

```c++
// @location  : src/base/platform/platform-win32.cc
// @namespace : v8::base::OS

707 | size_t OS::CommitPageSize() {
708 |   static size_t page_size = 0;
709 |   if (page_size == 0) {
710 |     SYSTEM_INFO info;
711 |     GetSystemInfo(&info);
712 |     page_size = info.dwPageSize;
713 |     DCHECK_EQ(4096, page_size);
714 |   }
715 |   return page_size;
716 | }
```

This function retrieves the value of page size of our physical memory using `GetSysInfo` function again and then stores the value of `dwPageSize` variable of SYSTEM_INFO struct on a static variable `page_size`. After that we will return to `PageAllocator` constructor which will return us to `DefaultPlatform` constructor.

```
hello-world.cc STACK:
  DefaultPlatform < NewDefaultPlatform < main
```

```c++
// @location  : src/libplatform/default-platform.cc
// @namespace : v8::platform

77 |   if (!tracing_controller_) {
78 |     tracing::TracingController* controller = new tracing::TracingController();
79 |     controller->Initialize(nullptr);
80 |     tracing_controller_.reset(controller);
81 |   }
```
We can see at line 77 there is a checking on our `tracing_controller_` variable, if it is a nullptr we will create a new `v8::platform::tracing::TracingController` instance and allocates its pointer value on heap like we have see at line 78. 

#### TracingController

```c++ 
// @location  : include/v8-platform.h
// @namespace : v8

137 | class TracingController {
138 |  public:
139 |   virtual ~TracingController() = default;
```

`v8::TracingController` is an abstract class that trace all events in V8, it can be implemented by the embedder and passed its value on `NewDefaultPlatform` function. You can see that at line 78 we created a `TracingController` instance of namespace `tracing::` instead of the namespace `v8::` thats because V8 has its own implementation of this class. 

```
hello-world.cc STACK:
  TracingController < DefaultPlatform < NewDefaultPlatform < main
```

```c++
// @location  : include/v8-tracing.h
// @namespace : v8::platform::tracing

229 | class V8_PLATFORM_EXPORT TracingController
230 |     : public V8_PLATFORM_NON_EXPORTED_BASE(v8::TracingController) {
```

```c++
// @location  : src/libplatform/tracing
// @namespace : v8::platform::tracing

64 | TracingController::TracingController() = default;

81 | void TracingController::Initialize(TraceBuffer* trace_buffer) {
82 |   trace_buffer_.reset(trace_buffer);
83 |   mutex_.reset(new base::Mutex());
84 | }
```

```
hello-world.cc STACK:
  Initialize < TracingController < DefaultPlatform < NewDefaultPlatform < main
```

After we have created its instance we will invoke its `Initialize` function and passing `nullptr` as its argument and passed the `nullptr` value on `reset` function of `trace_buffer_` variable. 

#### TraceBuffer

```c++
// @location  : include/libplatform/v8-tracing.h
// @namespace : v8::platform::tracing

156 | class V8_PLATFORM_EXPORT TraceBuffer {
157 |  public:
158 |   TraceBuffer() = default;
159 |   virtual ~TraceBuffer() = default;

295 |   std::unique_ptr<TraceBuffer> trace_buffer_;
```
#### Mutex

At line 83 we will invoke `reset` function of the variable `mutex_` passing new instance of `base::Mutex` as its argument. 

```
hello-world.cc STACK:
  Mutex < Initialize < TracingController < DefaultPlatform < NewDefaultPlatform < main
```

```c++
// @location  : src/base/platform/mutex.h
// @namespace : v8::base

37 | class V8_BASE_EXPORT Mutex final {
38 |  public:
39 |   Mutex();
40 |   ~Mutex();

60 |   using NativeHandle = SRWLOCK;

72 |  private:
73 |   NativeHandle native_handle_;
```
```c++
// @location  : src/base/platform/mutex.cc
// @namespace : v8::base

80 | Mutex::Mutex() {
81 |   InitializeNativeHandle(&native_handle_);
```

Inside `Mutex` constructor we will invoke `InitializeNativeHandle` function passing its `native_handle_` variable as argument. 

```
hello-world.cc STACK:
  InitializeNativeHandle < Mutex < Initialize < TracingController < DefaultPlatform < NewDefaultPlatform < main
```

```c++
// @location  : src/base/platform/mutex.cc
// @namespace : v8::base

14 | static V8_INLINE void InitializeNativeHandle(pthread_mutex_t* mutex) {
15 |   int result;

28 |   result = pthread_mutex_init(mutex, nullptr);
```

Inside `InitializeNativeHandle` function there is a call for `pthread_mutex_init` passing `mutex` and `nullptr` as arguments then after that we will returning to `DefaultPlatform` function. After we initialized our `controller` we will passed it value on `tracing_controller_` variable. 

```
hello-world.cc STACK:
  DefaultPlatform < NewDefaultPlatform < main
```

```c++
// @location  : src/libplatform/default-platform.cc
// @namespace : v8::platform

80 |     tracing_controller_.reset(controller);
81 |   }
```

After we fully created our `DefaultPlatform` instance, we will set some settings for it. 

```
hello-world.cc STACK:
  NewDefaultPlatform < main
```

```c++
43 |   platform->SetThreadPoolSize(thread_pool_size);
44 |   platform->EnsureBackgroundTaskRunnerInitialized();
45 |   return std::move(platform);
```














