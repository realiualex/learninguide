# AsyncIO 基础

## 基础概念

### 同步(synchronous)与异步(asynchronous)

如果任务是同步模式，即使使用了多线程，当任务从线程A切换到线程B，由于是同步阻塞，线程A空闲，线程B开始工作，因此并不能提高效率。

如果是异步的模式，那么任务则可以同时执行

### 并发(concurrency)和并行(parallelism)

并发指的是，可以接受多个任务，表面上看，所有的任务都在同时执行，但是细看，在某一个特定的时间点，只有一个任务在执行。也就是说微观上看是交替运行。

并行指的多个任务同时发生，微观上也是同时做

### 线程(Thread)与协程(Coroutine)

python的多线程，其实本质上是单线程。因为Python中有一个 GIL 全局解释器锁，所以多线程是串行执行的。资源占用率比较高，且开发者不能限制什么时候开始切换线程。

协程又叫做轻线程，本身是在一个线程里执行，资源占用率低，开发者可以自己指定切换的时机。同时由于只有1个线程工作，可以充分利用IO的等待时间。因此使用协程适合IO密集型工作，而不适合CPU密集型工作。

Python的协程，多数时候使用 asyncio实现(python 3.7+)，除了这个之外，还有 TRIO 也可以实现，生成器也可以实现协程。



## AsyncIO 注意点

* aysncio.run 是 python 3.7 才支持的，在python3.5-3.7之间，要用 loop = asyncio.get_event_loop()然后 loop.run_until_complete(task)



## 简单Demo

先使用python创建一个 server，由于 asyncio更多的是在 IO上提高效率，因此不能在异步请求中，直接使用time.sleep

```python
from fastapi import FastAPI
import uvicorn
import time

app = FastAPI()

@app.get("/")
def read_root():
    return {"Hello": "World"}

@app.get("/timeit/{seconds}")
def timeit(seconds):
    seconds = int(seconds)
    time.sleep(seconds)
    return f"sleep seconds: {seconds}"

if __name__ == "__main__":
    uvicorn.run(app)
    
```

再创建 client，注意不能用print，否则会报错，因为要求这个函数里全部都是 corountine 

```python
import asyncio
import httpx
import time

async def get_time():
    start_time = time.time()
    async with httpx.AsyncClient() as client:
        await client.get("http://127.0.0.1:8000/timeit/2")
    end_time = time.time()
    print("whole time is: ", end_time - start_time)

asyncio.run(get_time())
```

如果是多个请求，那么可以先创建为普通的对象，然后使用 asyncio.gather将task给传进去

```python
import asyncio
import time
import httpx
import requests


async def get_timeit():
    start_time = time.time()
    async with httpx.AsyncClient() as client:
        response = client.get("http://127.0.0.1:8000/timeit/1")
        # print(response.content)
        response2 = client.get("http://127.0.0.1:8000/timeit/2")
        # print(response2.content)
        await asyncio.gather(response, response2)
    end_time = time.time()
    print("total time is: ", end_time - start_time)

asyncio.run(get_timeit())

```



## AsyncIO 详解

### asyncIO 的 task

如果有多个任务，可以通过 asyncio.create_task() 来将任务添加进去，其实也可以创建event loop，然后loop.create_task()来添加

```python
import asyncio
import time

async def real_async(n):
    print("this is real async func")
    s_time = time.time()
    await asyncio.sleep(n)
    e_time = time.time()
    d_time = e_time - s_time
    print(f"end sleep {d_time}")

async def do_it():
    print("invoke real async")
    t1 = asyncio.create_task(real_async(1))
    t2 = asyncio.create_task(real_async(2))
    await t1
    await t2

asyncio.run(do_it())
```

不过这种做法用的比较少，用的比较多的是下面的。将任务放在一个列表里，然后用 await asyncio.wait()，将列表中的任务添加进来，在 asyncio.wait()函数里，还支持 timeout，指定最多等待时长(默认是 None)。这个函数有两个返回值，分别是 done 和 pending

```python
import asyncio
import time

async def real_async(n):
    print("this is real async func")
    s_time = time.time()
    await asyncio.sleep(n)
    e_time = time.time()
    d_time = e_time - s_time
    print(f"end sleep {d_time}")

async def do_it():
    print("invoke real async")
    task_list = [
        asyncio.create_task(real_async(1)), 
        asyncio.create_task(real_async(2))
        ]
    done, pending = await asyncio.wait(task_list, timeout=10)
    print(done)

asyncio.run(do_it())
```

上述 asyncio.create_task 的本质，是在 asyncio.run()的时候，已经创建了一个 event loop，会将task添加到 event loop里。如果我们想直接将任务给 asyncio.run，不创建额外的函数，那么应该按下面的写法

```python
import asyncio
import time

async def real_async(n):
    print("this is real async func")
    s_time = time.time()
    await asyncio.sleep(n)
    e_time = time.time()
    d_time = e_time - s_time
    print(f"end sleep {d_time}")

task_list = [
    real_async(1), 
    real_async(2)
    ]

done, pending = asyncio.run(asyncio.wait(task_list))
```

### asyncio 的 future

asyncio 的 task 在执行的时候，之所以不会一直被 hang 住，是因为在 asyncio.get_running_loop()的loop中，创建了一个 future 对象，这个future 对象如果什么都不做，是会被hang住的，但一旦有 future.set_result() 赋值的时候，就会返回。而task在执行成功之后，就会自动给这个future对象赋值，因此不会hang住。一般我们不会手动的创建future对象，而是直接用task



### concurrent的future

concurrent 里的future对象和 asyncio的future没有任何关系，在使用线程池，或进程池实现异步操作的时候，会用到的。但由于使用协程的时候，await后面跟的对象必须是一个协程对象(或协程task或协程future)，但有些第三方包可能未必支持协程，那这时候就需要将 asyncio 和多 线程/进程  的模块结合起来做异步。

#### 使用 concurrent 模块创建一个基本的多线程

```python
import time
from concurrent.futures import Future
# IO密集的任务,用ThreadPoolExecutor;CPU密集任务,用ProcessPoolExcutor
from concurrent.futures import ThreadPoolExecutor
from concurrent.futures import ProcessPoolExecutor

def func(value):
    time.sleep(1)
    print(value)
    return 123

# 创建线程池
pool1 = ThreadPoolExecutor(max_workers= 5)
# 创建进程池
#pool1 = ProcessPoolExecutor(max_workers= 5)

for i in range(10):
    fut = pool1.submit(func, i)
    print(fut)
```

#### 将asyncio 和 concurrent 多线程结合使用

一个最简单的示例，可以在 asyncio 中，通过 loop.run_in_executor 将 进程池或线程池的future，转换为 asyncio 协程的 future

```python
import asyncio
import requests
from concurrent.futures import ThreadPoolExecutor


def get_content(url):
    with requests.session() as session:
        content = session.get(url)
        return content.content


async def main():
    loop = asyncio.get_running_loop()
    with ThreadPoolExecutor() as pool:
        result1 = await loop.run_in_executor(pool, get_content, "https://www.google.com")
        #result2 = await loop.run_in_executor(pool, get_content, "https://www.apple.com")
        print(result1)
        # print(result2)

if __name__ == "__main__":
    asyncio.run(main())
    
```

在上述代码中，如果有多个任务，我们可以在 main()函数里，运行多个await loop.run_in_executor，如上注释所示。但是这样组织比较复杂，在改的时候也容易该乱。所以我们可以将这个函数拆分成两个，一个只是实现将普通的任务，返回一个 concurrent 的 future对象，另外在 main 函数里开始真正执行。而main 函数里，我们也可以选择 as_complete 做异步返回，也可以用 gather 来收集多个task，让所有task都拿到结果再一次性返回。这样以后改这个返回逻辑的时候，也比较简单，不会动到 async_get_content的内容和逻辑

```python
import asyncio
import requests
from concurrent.futures import ThreadPoolExecutor


def get_content(url):
    with requests.session() as session:
        content = session.get(url)
        return content.content


async def async_get_content(loop, url):
    with ThreadPoolExecutor() as pool:
        result = await loop.run_in_executor(pool, get_content, url)
        return result


async def main():
    loop = asyncio.get_running_loop()
    gather_google = async_get_content(loop, "https://google.com")
    gather_apple = async_get_content(loop, "https://apple.com")
    tasks = [gather_apple, gather_google]
    for future in asyncio.as_completed(tasks):
        result = await future
        print(result)


if __name__ == "__main__":
    asyncio.run(main())

```



### 有关await 和 asyncio.run

await后面只能跟3类对象：

1. 协程对象 (coroutine object)，这里指的是使用 async 定义的函数，返回的对象就是协程对象。他们可以直接用 await 关键字出发执行
1. 可等待的对象 (awaitable object)，可以被await语句使用的对象，其中包括了协程函数，任务和其他抽象基类实现了 \_\_await\_\_()方法的对象。我们可以使用 asyncio.iscoroutine() 或者 asyncio.iscoroutinefunction() 函数来检查对象是否可等待
1. Future对象，asyncio.future 对象表示一个尚未完成的计算，await可以等待 future对象完成。

* 协程对象示例：

  下面的cc()和 dd()就是协程对象，可以直接通过 run_until_complete() 给运行起来

```python
import asyncio
async def cc():
    print("cc")

async def dd():
    print("dd")

loop = asyncio.get_event_loop()
loop.run_until_complete(cc())
loop.run_until_complete(dd())

```

在python3.7，可以不用 get_event_loop 和 run_until_complete，直接使用 asyncio.run（），则上面代码可改成

```python
async def cc():
    print("cc")
async def dd():
    print("dd")

asyncio.run(cc())
asyncio.run(dd())
```

如果任务比较多，我们可以使用 gather 来收集多个任务

```python
async def cc():
    print("cc")

async def dd():
    print("dd")

main_task = asyncio.gather(cc(),dd())
loop = asyncio.get_event_loop()
loop.run_until_complete(main_task)
```

有时候为了使得代码更容易阅读，我们会把 gather对象放到list里，然后再解包，比如

```python
task_list = [cc(), dd()]
main_task = asyncio.gather(*task_list)
```

如果使用 asyncio.run，由于默认gather返回的是一个 gather future对象，因此要用await，为了使用 await，我们要再定义一个task函数

```python
import asyncio
async def cc():
    print("cc")

async def dd():
    print("dd")
    
task_list = [cc(), dd()]
async def task():
    await asyncio.gather( *task_list )

asyncio.run(task())
```

除了使用gather方法之外，我们还可以直接使用wait方法，这样不用定义额外的函数，会更简单一些

```python
async def cc():
    print("cc")
    await asyncio.sleep(1)

async def dd():
    print("dd")

task_list = [cc(), dd()]
wait_task = asyncio.wait(task_list)
asyncio.run(wait_task)
```



2. 对于awaitable对象，也比较好理解，就是在 async 函数里，加上 await。示例

```python
async def dd():
    print("dd")
    await asyncio.sleep(2)
 asyncio.run(dd())
```

当然如果跟1一样，如果有多个任务的话，我们可以用 asyncio.gather()来收集起来

```python
async def cc():
    print("cc")
    await asyncio.sleep(1)

async def dd():
    print("dd")
    await asyncio.sleep(2)
    
task_list = [cc(), dd()]

async def task():
    await asyncio.gather(*task_list)

asyncio.run(task())
```

3. 如果是Gatherfuture对象，那么asyncio.run()是运行不起来的，不过loop.run_until_complete是可以的.

## python 3.9+ 的 to_thread将任意方法转为异步

在 python 3.9开始，我们可以用 asyncio.to_thread()方法，将task放到另外一个thread里来实现异步，在 thread内部本质还是并行的，但是多个 thread之间是异步的。所以相当于是用 cpu 的资源来解决IO的问题

```python
import asyncio
from asyncio import Task
import requests
from requests import Response


async def fetch_status(url: str) -> dict:
    print(f"fetching status for {url}")
    response: Response = await asyncio.to_thread(requests.get, url)
    print("done")
    return {"status": response.status_code, 'url': url}


async def main() -> None:
    apple_task: Task[dict] = asyncio.create_task(fetch_status('https://apple.com'))
    google_task: Task[dict] = asyncio.create_task(fetch_status('https://google.com'))

    apple_status: dict = await apple_task
    google_status: dict = await google_task

    print(apple_status, google_status)

if __name__ == '__main__':
    asyncio.run(main=main())

```

### python 3.9 之后版本的asyncio 使用最佳实践

这里我们通过使用 aiohttp 的标准的 asyncio 方法来执行，在拿到 asyncio 的执行结果的时候，有三种方法:
第一种是直接返回，适合任务比较少的

第二种是用 asyncio.gather() 收集多个任务，然后等所有的结果都拿到后，再一次性返回

第三种是用 asyncio.as_complete()，将有结果的任务立即返回

```python
import asyncio
from asyncio import Task
import aiohttp


async def fetch_status(url: str) -> dict:
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            status = response.status
            return {"status": status, "url": url}


async def main() -> None:
    apple_task: Task = asyncio.create_task(fetch_status("https://www.apple.com"))
    google_task: Task = asyncio.create_task(fetch_status("https://www.google.com"))

    # # 方法一: 直接获得每一个task结果，但如果任务比较多的时候，不好管理
    # apple_result = await apple_task
    # google_result = await google_task
    # print(apple_result, google_result)

    # 方法二和三需要放到list里
    tasks = [apple_task, google_task]
    # # 方法二
    # # 如果用 gather，那么会在所有任务都完成之后，再返回。
    # results = await asyncio.gather(*tasks)
    # #这里的第一个结果在 results[0]，第二个在  results[1]
    # for result in results:
    #     print(result)

    # 方法三
    # 如果用 as_completed，那么一旦有任何一个返回，就会立即返回 as_completed 返回的结果是一个迭代器
    for future in asyncio.as_completed(tasks):
        result = await future
        print(result)


if __name__ == "__main__":
    asyncio.run(main())


```





## 多线程与多进程

python的GIL主要影响 CPU 密集型任务，在任何时候，都只能有一个线程运行，并不能真正处理多个任务。但是如果是 IO 密集型任务，一个线程在等待IO，另外一个线程可以运行，所以依然能提高效率。GIL并不影响多进程的任务。如果没有GIL，那么一个程序的多个线程，理论上可以同时跑在多个物理cpu上，因为线程是操作系统调度的最小单位。

python实现多进程的底层库是 multiprocessing，实现多线程的底层库是 threading。多线程multiprocessing 提供了3种进程间通信的方法，分别是Pipe, Queue, Pool。其中Queue是通过 /dev/shm (一个基于内存的文件系统)来实现的。但由于所有的进程都能访问这个路径，因此也有一定的安全风险。Queue的实现是线程安全，不存在锁的问题。由于 /dev/shm 是内存的映射，甚至可以使用 dd 命令对这个路径进行写入来做内存的压测(注意不能超过可用内存大小，否则OOM)

```
dd if=/dev/zero of=/dev/shm/test bs=1M count=1024
```

在python 3.2引入了 concurrent.future 的ProcessPoolExecutor 和 ThreadPoolExecutor 来简化多进程和多线程的操作，concurrent.future 也是靠 multiprocessing (pool，没有提供 pipe和 queue) 和 threading 实现的

### 多线程

一个多线程的示例

```python
from concurrent.futures import ThreadPoolExecutor
import time

def a(n):
    time.sleep(n)
    print(f"sleep_{n}")
    return n


if __name__ == "__main__":
    # max_workers 指线程池最多能运行的线程数目
    executor = ThreadPoolExecutor(max_workers=10)
    # 方法一，直接提交任务，submit不是阻塞的，而是立即完成，要使用 done 方法判断是否真正的执行完成
    z = executor.submit(a, 1)
    print(z.done())
    time.sleep(2)
    print(z.done())
    # 通过result拿到真正的结果，result的方法是阻塞的
    print(z.result())
    # 方法二：如果任务比较多，我们可以用 map 方法
    for data in executor.map(a, [3,2,4]):
        # map方法是按顺序返回的
        print(data)
    # 方法三: 在方法一上的优化，通过 as_complete方法一次性取出所有任务的结果，而不是不停的遍历每一个线程的结果
    sp = [3, 2, 4]
    jobs = [executor.submit(a, s) for s in sp]
    for future in as_completed(jobs):
      data=future.result()
      print(data)


```

在上述示例种，如果使用 top -H -p PID 来观察每个线程，会发现是同时关闭的，这是因为 map 或 as_completed 维护了线程池，没有任务的线程也会等待接受新任务。

### 多进程

多进程的用法，跟多线程类似，区别在于：

1. 一开始就创建了 max_workers 个进程，这个与多线程不一样，多线程是根据 task决定创建多少个线程，如果task多余max_worker，则剩余task会处于等待状态。
2. 多进程的 executor.map() 的返回值是每个任务结果的可迭代对象，由于这个示例return得是int类型的值，因此直接打印这个int值就可以了。而多线程要用 .result() 方法获得结果

```python
from concurrent.futures import ProcessPoolExecutor
import time


def a(n):
    time.sleep(n)
    print(f"sleep {n}")
    return n


if __name__ == "__main__":
    executor = ProcessPoolExecutor(max_workers=10)

    tasks = [60,30,40]

    for result in executor.map(a, tasks):
        print(result)
```

### multiprocessing 里的 fork vs spawn
* fork 
在 python 3.14 之前的版本，POSIX (Linux) 标准的系统，使用的是 fork 创建子进程，fork 在创建子进程的时候，会 copy 父进程的环境变量和状态信息（copy on write复制父进程的内存所有信息，在fork的时候还用同一块内存，当有写操作的时候才真正复制），也许子进程用不到，但是 fork 出子进程的时候会完全拷贝，这样性能不好，且容易出现安全问题。
* spawn
在 Windows 和 MacOS 系统上，python 用的是 spawn 创建出来子进程， spawn 在创建子进程的时候，启动的是一个全新的 python 解释器环境，这个环境在启动的时候，只会复制 run() 方法启动所需要的数据。spawn 启动比较慢
* forksever
从 3.14 开始，python 在 Linux系统上，弃用了 fork() 方法创建子进程，使用 forksever() 来创建，forkserver的原理是: 在 multiprocessing 第一次创建子进程的时候，先用 spawn() 创建一个 forkserver 进程，这个进程比较干净，然后再用 forkserver 进程通过 fork 方法创建子进程
	* 从这里来看，如果 linux下的 python 程序要创建比较多的子进程，那么 forkserver 比较合适，能兼备 spawn 比较干净 和 fork 比较快的优势
	*  但如果python 程序只创建1个子进程，那么其实使用 spawn 会更快( 少了 forkserver 的创建过程)
