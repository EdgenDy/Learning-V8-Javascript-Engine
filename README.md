## A Detailed View inside of V8 Javascript Engine

In this tutorial we will find out how exactly V8 executes the js code `'hello' + 'world'`, by analyzing and examining every line in `hello-world.cc` file in `v8/samples` directory. 

Note : The version of v8 that we will used here is v8-7.9.2. 

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

41 |        		InProcessStackDumping::kDisabled,42 | /* 4 */    std::unique_ptr<v8::TracingController> tracing_controller = {});

```

The first parameters is `thread_pool_size` which is an integer with an initial value of zero. We will discussed it later. 

The second is `idle_task_support` which is a key of enum class `IdleTaskSupport` with an initial value `IdleTaskSupportkDisabled::kDisabled` and its declaration and definition is found on `src/libplatform/libplatform.h` at line 16.

```c++

16 | enum class IdleTaskSupport { kDisabled, kEnabled };

```

The third is `in_process_stack_dumping` which is a key of enum class `InProcessStackDumpingInProcessStackDumping::kDisabled` with an initial value `kDisabled` and its declaration and definition is found on `src/libplatform/libplatform.h` at line 17.

```c++

17 | enum class InProcessStackDumping { kDisabled, kEnabled };

```

And the last is `v8::TracingController` that wraps in `std::unique_ptr` object and construct it with the default constructor as its initial value. I ts declaration is found at line 137 in `include/v8-platform.h`

```c++

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

**Declaration**

```c++

// src/libplatform/default-platform.h

32 | class V8_PLATFORM_EXPORT DefaultPlatform : public NON_EXPORTED_BASE(Platform) {

      ...

90 | } 

```

**Definition**

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
