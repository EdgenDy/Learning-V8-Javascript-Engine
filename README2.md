## Understanding How V8 Javascript Engine Really Works
---
V8 is a javascript engine that powered both chrome browser and nodejs and standout among others javascript engine like spidermonkey of mozilla firefox and chakra of Microsoft Edge. In this lesson we will find out how V8 javascript engine really works thru examining many lines of the v8 source codes. We will used a sample code from the v8 itself the `hello-world.cc` from the `samples` directory of v8 project for our examination.

**Note:** v8-7.9.2 is the version of v8 that we will referenced throughout this lesson, you can find it [here](here). 

```c++
```

### Initializing Platform

In our example code we will execute a js code `'hello' + 'world'`, but before that we need to initialize a platform and then we can initialized V8. This platform will be used as a driver for Memory allocation and other platform specific tasks like system clock time and threads. V8 is not only runs on one platform but on different operating systems and different architectures. 

```c++
// samples/hello-world.cc

16 | std::unique_ptr<v8::Platform> platform = v8::platform::NewDefaultPlatform();
17 | v8::V8::InitializePlatform(platform.get());
```

To initialized a platform we need to create a platform object. V8 team encourage embedders to create their own implementation of the `v8::Platform` interface but in our example code we will use `v8::platform::DefaultPlatform` class, the default implementation of `v8::Platform`. To create a new instance of it we will used `v8::platform::NewDefaultPlatform()` method.

```c++
// include/libplatform/libplatform.h

37 | V8_PLATFORM_EXPORT std::unique_ptr<v8::Platform> NewDefaultPlatform(
38 |     int thread_pool_size = 0,
39 |     IdleTaskSupport idle_task_support = IdleTaskSupport::kDisabled,
40 |     InProcessStackDumping in_process_stack_dumping =
41 |         InProcessStackDumping::kDisabled,
42 |     std::unique_ptr<v8::TracingController> tracing_controller = {});
```

This method takes 4 parameters and each parameter has its default value. Because we invoke this method without supplying any arguments it will used all the default value of these parameters. 