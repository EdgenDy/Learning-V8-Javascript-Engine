## A Detailed View inside V8 Javascript Engine

In this tutorial we will find out how exactly V8 executes the js code `'hello' + 'world'`, by analyzing and examining every line in [`hello-world.cc`](https://github.com/v8/v8/blob/7.9.2/samples/hello-world.cc) file in `v8/samples` directory. 

Note : The version of v8 that we will used here is [v8-7.9.2](https://github.com/v8/v8/tree/7.9.2). 

What are we waiting for, lets get started!



### Initializing The Platform

If we look at the line 16 of the `v8/samples/hello-world.cc` we will see the following code:
``` c++ 
16 | std::unique_ptr<v8::Platform> platform = v8::platform::NewDefaultPlatform();
```
You will notice that we create a `v8::Platform` object that wraps in a `std::unique_ptr` object by using `NewDefaultPlatform()` method of the namespace `v8::platform`, the definition of this method is in the line 34 of `v8/src/libplatform/default-platform.cc`, while the declaration is in the `v8/include/libplatform/libplatform.h` at line 37.

```c++
// src/libplatform/libplatform.h 

37 | V8_PLATFORM_EXPORT std::unique_ptr<v8::Platform> NewDefaultPlatform(
38 | /* 1 */    int thread_pool_size = 0,
39 | /* 2 */    IdleTaskSupport idle_task_support = IdleTaskSupport::kDisabled,
40 | /* 3 */    InProcessStackDumping in_process_stack_dumping =
41 |        		InProcessStackDumping::kDisabled,
42 | /* 4 */    std::unique_ptr<v8::TracingController> tracing_controller = {});
```
The first parameters is `thread_pool_size` which is an integer with an initial value of zero. We will discussed it later. 

The second is `idle_task_support` which is a key of enum class `IdleTaskSupport` with an initial value `IdleTaskSupportkDisabled::kDisabled`.

```c++
// src/libplatform/libplatform.h

16 | enum class IdleTaskSupport { kDisabled, kEnabled };
```

The third is `in_process_stack_dumping` which is a key of enum class `InProcessStackDumpingInProcessStackDumping::kDisabled` with an initial value `kDisabled`.
```c++
// src/libplatform/libplatform.h

17 | enum class InProcessStackDumping { kDisabled, kEnabled };
```

And the last is `v8::TracingController` that wraps in an empty `std::unique_ptr` object.

```c++
// include/v8-platform.h

137 | class TracingController {
138 |  ...
199 | }
```

Inside this method there is a checking on our parameter. 

```c++
// src/libplatform/default-platform.cc

38 | if (in_process_stack_dumping == InProcessStackDumping::kEnabled) {
39 |     v8::base::debug::EnableInProcessStackDumping();
40 | }
```

In the code above it checked if the `in_process_stack_dumping` argument is `kEnabled` so we can call `EnableInProcessStackDumping` method, but in our case our `in_process_stack_dumping` is `kDisabled` so we will proceed on creation of platform. 

```c++
// src/libplatform/default-platform.cc

41 | std::unique_ptr<DefaultPlatform> platform(
42 |      new DefaultPlatform(idle_task_support, std::move(tracing_controller)));
```

In this line of code we allocate `DefaultPlatform` object instance on the heap by using `new` operator and wraps it in `std::unique_ptr` object. `DefaultPlatform` is a child class of `v8::Platform` and we used it as our default platform. 


**Declaration:**
```c++
// src/libplatform/default-platform.h

32 | class V8_PLATFORM_EXPORT DefaultPlatform : public NON_EXPORTED_BASE(Platform) {
      ...
90 | } 
```


**Implementation:**
```c++
// src/libplatform/default-platform.cc

69 | DefaultPlatform::DefaultPlatform(
70 |     IdleTaskSupport idle_task_support,
71 |     std::unique_ptr<v8::TracingController> tracing_controller)
72 |     : thread_pool_size_(0),
73 |       idle_task_support_(idle_task_support),
74 |       tracing_controller_(std::move(tracing_controller)),
75 |       page_allocator_(new v8::base::PageAllocator()),
76 |       time_function_for_testing_(nullptr) {
77 |   if (!tracing_controller_) {
78 |     tracing::TracingController* controller = new tracing::TracingController();
79 |     controller->Initialize(nullptr);
80 |    tracing_controller_.reset(controller);
81 |  }
82 | }
```
Inside this constructor it only check if `tracing_controller_` is empty and then it will create a new `v8::platform::tracing::TracingController` object instance and it will Initialize as you've seen on the code above. `TracingController` extends the public base class `v8::TracingController`.

**Declaration**
```c++
// include/libplatform/v8-tracing.h

229 | class V8_PLATFORM_EXPORT TracingController
230 |     : public V8_PLATFORM_NON_EXPORTED_BASE(v8::TracingController) {
231 |  public:
    |   ...
249 |   void Initialize(TraceBuffer* trace_buffer);
    |   ... 
310 | }; 
```

**Implementation**
```c++
// src/libplatform/tracing/tracing-controller.cc

64 | TracingController::TracingController() = default;
   | ... 
81 | void TracingController::Initialize(TraceBuffer* trace_buffer) {
82 |   trace_buffer_.reset(trace_buffer);
83 |   mutex_.reset(new base::Mutex());
84 | }
```























