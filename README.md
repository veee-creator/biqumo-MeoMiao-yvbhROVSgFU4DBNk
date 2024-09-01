
今天复习前几年在项目过程中积累的各类技术案例，有一个小的 coredump 案例，当时小组里几位较资深的同事都没看出来，后面是我周末查了两三个小时解决掉的，今天再做一次系统的总结，给出一个复现的案例代码，案例代码比较简单，便于学习理解。


# 1\. 简介


原则：临时对象不应该被 lambda 引用捕获，因为临时对象在它所在的语句结束就会被析构掉，只能采用值捕获。
当临时对象比较隐蔽时，我们就可能犯这个低级错误。本文介绍一类case：以基类智能指针对象的 const 引用为函数形参，并在函数内对该参数做引用捕获，然后进行跨线程异步使用。当函数调用者使用派生类智能指针作为实参时，此时派生类智能指针对象会向上转换为基类智能指针对象，这个转换是隐式的，产生的对象是临时对象，然后被 lambda 引用捕获，后续跨线程使用引发“野引用” core。


# 2\. 案例


下面写一个简单的 demo 代码来模拟这个案例。案例涉及的代码流程，如下图所示：
![](https://img2024.cnblogs.com/blog/104985/202408/104985-20240831195239230-527210276.png)
其中，基类 BaseTask，派生类 DerivedTask，main 函数将 lambda 闭包抛到工作线程中异步执行。
详细示例代码如下：



```
/**
 * @brief 关键字：lambda、多线程、std::shared_ptr 隐式向上转换
 * g++ main.cc -std=c++17 -O3 -lpthread
 */

#include 
#include 
#include 
#include 
#include 
#include 
#include 
#include 
#include 

using namespace std::chrono_literals;

/// 简易线程池
template <typename Func>
class ThreadPool {
 public:
  ~ThreadPool() {
    stop_ = true;
    for (auto& item : workers_) {
      item.join();
    }
  }

  void Run() {
    static constexpr uint32_t kThreadNum = 2;

    uint32_t idx = 0;
    for (uint32_t idx = 0; idx != kThreadNum; ++idx) {
      workers_.emplace_back(std::thread([this, idx] { ThreadFunc(idx); }));
      mutexs_.emplace_back(std::make_shared());
    }

    job_queues_.resize(kThreadNum);
  }

  void Stop() { stop_ = true; }

  bool Post(Func&& f) {
    if (!stop_) {
      uint32_t index = ++job_cnt_ % job_queues_.size();
      auto& queue = job_queues_[index];
      std::lock_guard locker(*mutexs_[index]);
      queue.push(std::move(f));
      return true;
    }

    return false;
  }

  void ThreadFunc(uint32_t idx) {
    auto& queue = job_queues_[idx];
    auto& mutex = *mutexs_[idx];
    // 退出前清空任务队列
    while (true) {
      if (!queue.empty()) {
        std::lock_guard locker(mutex);
        const auto& job_func = queue.front();
        job_func();
        queue.pop();
      } else if (!stop_) {
        std::this_thread::sleep_for(10ms);
      } else {
        break;
      }
    }
  }

 private:
  /// 工作线程池
  std::vector workers_;
  /// 任务队列，每个工作线程一个队列
  std::vector> job_queues_;
  /// 任务队列的读写保护锁，每个工作线程一个锁
  std::vector> mutexs_;
  /// 是否停止工作
  bool stop_ = false;
  /// 任务计数，用于将任务均衡分配给多线程队列
  std::atomic<uint32_t> job_cnt_ = 0;
};
using MyThreadPool = ThreadPoolvoid()>>;

/// 基类task
class BaseTask {
 public:
  virtual ~BaseTask() = default;
  virtual void DoSomething() = 0;
};
using BaseTaskPtr = std::shared_ptr;

/// 派生task
class DeriveTask : public BaseTask {
 public:
  void DoSomething() override {
    std::cout << "derive task do someting" << std::endl;
  }
};
using DeriveTaskPtr = std::shared_ptr;

/// 示例用户
class User {
 public:
  User() { thread_pool_.Run(); }
  ~User() { thread_pool_.Stop(); }

  void DoJobAsync(const BaseTaskPtr& task) {
    // task 是 user->DoJob 调用产生的临时对象，捕获它的引用会变成也指针
    thread_pool_.Post([&task] { task->DoSomething(); });
  }

 private:
  MyThreadPool thread_pool_;
};
using UserPtr = std::shared_ptr;

/// 测试运行出 core
int main() {
  auto user = std::make_shared();
  DeriveTaskPtr derive_task1 = std::make_shared();
  // derive_task 会隐式转换为 BaseTask 智能指针对象，
  // 该对象是临时对象，在 DoJob 执行完之后生命周期结束。
  user->DoJobAsync(derive_task1);

  DeriveTaskPtr derive_task3 = std::make_shared();
  user->DoJobAsync(derive_task3);

  std::this_thread::sleep_for(std::chrono::seconds(3));
  return 0;
}

```

上面这个例子代码，会出现 coredump，或者是没有执行派生类的 DoSomething，总之是不符合预期。不符合预期的原因如下：这份代码往一个线程里 post lambda 函数，lambda 函数引用捕获智能指针对象，这是一个临时对象，其离开使用域之后会被析构掉，导致 lambda 函数在异步线程执行时，访问到一个"野引用"出错。而之所以捕获的智能指针是临时对象，是因为调用 User.DoJobAsync 时发生了类型的向上转换。


上述的例子还比较容易看出来问题点，但当我们的项目代码层次较深时，这类错误就非常难看出来，也因此之前团队里的资深同事也都无法发现问题所在。


这类问题有多种解决办法：
（1）方法1：避免出现隐式转换，消除临时对象；
（2）方法2：函数和 lambda 捕获都修改为裸指针，消除临时对象；引用本质上是指针，需要关注生命周期，既然采用引用参数就表示调用者需要保障对象的生命周期，智能指针的引用在用法上跟指针无异，那么这里不如用裸指针，让调用者更清楚自己需要保障对象的生命周期；
（3）方法3：异步执行时采用值捕获/值传递，不采用引用捕获，但值捕获可能导致性能浪费，具体到本文的例子，这里的性能开销是一个智能指针对象的构造，性能损耗不大，是可接受的。


# 3\. 其他


临时对象的生命周期可以参考这篇文档：[https://en.cppreference.com/w/cpp/language/reference\_initialization\#Lifetime\_of\_a\_temporary](https://github.com)


 本博客参考[西部世界官网](https://tianchuang88.com)。转载请注明出处！
