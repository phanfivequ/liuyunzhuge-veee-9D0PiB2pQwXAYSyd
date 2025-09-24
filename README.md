Python 异步编程是一种基于**非阻塞 IO 模型**的并发编程范式，核心目标是在处理 IO 密集型任务（如网络请求、文件读写、数据库交互）时，通过高效的任务调度减少等待时间，最大化 CPU 利用率。

异步编程通过**事件循环**实现任务调度：当一个任务因 IO 操作需要等待时，事件循环会暂停该任务，切换到其他就绪任务；当 IO 操作完成（如响应到达），事件循环再恢复原任务的执行。

**核心优势**：

* 单线程内实现并发，避免多线程的上下文切换开销。
* 针对 IO 密集型任务，性能提升显著（通常是同步方式的数倍到数十倍）。

### 1、核心组件

#### 1.1 事件循环

事件循环是异步编程的 “心脏”，负责**任务调度、IO 事件监听、状态管理**。

```
# 伪代码
任务列表 = [ 任务1,任务2,任务3,...]

while True:
    可执行的任务列表,已完成的任务列表 = 去任务列表中检查所有的任务,将'可执行'和'已完成'的任务返回
    
	for 就绪任务 in 已准备就绪的任务列表:
		执行已就绪的任务
       
	for 已完成的任务 in 已完成的任务列表:
		在任务列表中移除 已完成的任务
       
	如果 任务列表 中的任务都已完成,则终止循环
```

**关键细节**：

* 事件循环在**单线程**中运行，所有任务都在这个线程内切换执行。
* 仅当任务遇到 `await`（表示需要等待 IO）时才会切换，纯计算任务不会触发切换。

#### 1.2 协程

协程是异步任务的基本单元，是一种**用户态的上下文切换技术**，其实就是通过**一个线程实现代码块相互切换**执行，本质是**可暂停 / 恢复的函数**，通过 `async def` 定义。与普通函数的区别在于：

* 调用协程不会立即执行，而是返回一个**协程对象**。
* 必须通过**事件循环**调度（如 `await`、`create_task`）才能执行。

```
# 协程的定义与状态
import asyncio

async def my_coroutine():
    print("协程开始")
    await asyncio.sleep(1)  # 暂停点：释放CPU，允许切换
    print("协程结束")
    return "结果"

# 协程对象（未执行）
coro = my_coroutine()
print(type(coro))  # 

# 必须通过事件循环执行
async def main():
    result = await coro  # 调度执行，等待结果
    print(result)  # 输出：结果

asyncio.run(main())
```

**协程的生命周期**：

* **创建**：`coro = my_coroutine()` → 未执行状态。
* **运行**：`await coro` 或 `create_task(coro)` → 进入事件循环。
* **暂停**：执行到 `await` 语句 → 等待 IO 时挂起。
* **恢复**：IO 完成 → 从暂停点继续执行。
* **完成**：执行到函数末尾 → 返回结果或抛出异常。

#### 1.3 任务

任务是**协程的包装器**，由事件循环直接调度，用于实现**并发**。任务会将协程注册到事件循环，并跟踪其状态（运行中 / 已完成 / 已取消）。

```
async def task_func(name, delay):
    print(f"任务 name={name}, delay={delay} === 111")
    await asyncio.sleep(delay)
    print(f"任务 name={name}, delay={delay} === 222")
    return f"任务 {name} 完成"

async def main():
    # 创建任务（立即加入事件循环，开始调度）
    task1 = asyncio.create_task(task_func("A", 1))
    task2 = asyncio.create_task(task_func("B", 2))
    
    print("任务状态:", task1.done())  # False（未完成）
    
    # 等待任务完成并获取结果
    result1 = await task1
    result2 = await task2
    
    print("结果:", result1, result2)  # 任务 A 完成 任务 B 完成
    print("任务状态:", task1.done())  # True（已完成）

asyncio.run(main())
```

**任务的核心方法**：

* `task.done()`：判断任务是否完成。
* `task.result()`：获取任务返回值（任务未完成时调用会报错）。
* `task.cancel()`：取消任务（触发 `CancelledError`）。
* `task.add_done_callback(func)`：注册回调函数（任务完成后执行）。

**更常用写法：**

```
async def task_func(name, delay):
    print(f"任务 name={name}, delay={delay} === 111")
    await asyncio.sleep(delay)
    print(f"任务 name={name}, delay={delay} === 222")
    return f"任务 {name} 完成"


async def main():
    # 创建任务（立即加入事件循环，开始调度）
    task_list = [
        asyncio.create_task(task_func("A", 1), name="task_A"),
        asyncio.create_task(task_func("B", 2), name="task_B"),
    ]

    done, pending = await asyncio.wait(task_list, timeout=None)

    # 等待任务完成并获取结果
    print(done)


asyncio.run(main())
```

#### 1.4 Future 对象

Future 是**异步操作结果的容器**，表示 “未来可能完成的操作”。任务（Task）是 Future 的子类，因此具备 Future 的所有特性，task对象内部await结果的处理是基于future的：

* 存储异步操作的状态（`PENDING`/`FINISHED`/`CANCELLED`）。
* 提供结果设置（`set_result()`）和异常设置（`set_exception()`）方法。
* 支持通过 `await` 获取结果，或注册回调函数。

```
async def main():
    # 创建一个空的Future对象
    future = asyncio.Future()
    
    # 定义一个设置Future结果的协程
    async def set_future_result():
        await asyncio.sleep(1)
        future.set_result("Future 结果")  # 设置结果，标记为完成
    
    # 并发执行：设置结果的协程 + 等待结果的操作
    asyncio.create_task(set_future_result())
    result = await future  # 等待Future完成
    print(result)  # 输出：Future 结果

asyncio.run(main())
```

**Task 与 Future 的关系**：

* Task 继承自 Future，是 “可执行的 Future”（绑定了协程）。
* Future 更底层，通常用于手动管理异步操作结果（如包装回调式 API）。

### 2、基础语法和核心API

#### 2.1 `async/await` 语法

`async/await` 是 Python 3.5+ 引入的异步语法糖，用于定义协程和暂停执行：

* `async def`：定义协程函数（返回**协程对象**）。
* `await`：暂停协程，**等待**另一个**协程 / Future/Task** 完成，只能在协程内部使用。

```
async def nested():
    return 42

async def main():
    # 直接调用协程不会执行，必须用await
    result = await nested()  # 等待nested完成，获取结果
    print(result)  # 42

asyncio.run(main())
```

**注意**：

* 普通函数中不能使用 `await`（会报 `SyntaxError`）。
* `await` 后面必须是 “**可等待对象**”（协程、Task、Future）

#### 2.2 事件循环的启动与管理

Python 3.7+ 推荐用 `asyncio.run()` **启动事件循环**（自动创建、运行、关闭循环），低版本需手动管理：

```
# Python 3.7+ 推荐方式
async def main():
    await asyncio.sleep(1)
    print("完成")

asyncio.run(main())  # 自动处理事件循环的生命周期

# 低版本手动管理方式（3.6及以下）
loop = asyncio.get_event_loop()  # 获取事件循环
try:
    loop.run_until_complete(main())  # 运行直到协程完成
finally:
    loop.close()  # 关闭循环
```

#### 2.3 并发任务管理

##### 2.3.1 `asyncio.gather()`

**批量并发与结果聚合**

`gather()` 用于同时运行多个可等待对象，按输入顺序返回结果，适合需要统一收集结果的场景。

```
async def task(i):
    await asyncio.sleep(i)
    return i

async def main():
    # 并发执行3个任务
    results = await asyncio.gather(
        task(1),
        task(2),
        task(0.5)
    )
    print(results)  # [1, 2, 0.5]（按输入顺序，而非完成顺序）

asyncio.run(main())
```

**高级参数**：

* `return_exceptions=True`：将异常作为结果返回，不中断其他任务。

  ```
  async def faulty_task():
      raise ValueError("出错了")

  async def main():
      results = await asyncio.gather(
          faulty_task(),
          task(1),
          return_exceptions=True  # 异常会被包装到结果中
      )
      print(results)  # [ValueError('出错了'), 1]
  ```

##### 2.3.2 `asyncio.wait()`

**灵活控制任务完成条件**

`wait()` 比 `gather()` 更灵活，支持按 “第一个完成”“所有完成” 等条件返回，返回值是已完成和未完成的任务集合。

```
async def main():
    tasks = [task(1), task(2), task(0.5)]
    # 等待所有任务完成（默认）
    done, pending = await asyncio.wait(tasks)
    print("已完成任务数:", len(done))  # 3
    print("未完成任务数:", len(pending))  # 0

    # 等待第一个任务完成
    done, pending = await asyncio.wait(tasks, return_when=asyncio.FIRST_COMPLETED)
    print("第一个完成的任务结果:", [t.result() for t in done])  # [0.5]
```

`return_when` 可选值：

* `FIRST_COMPLETED`：第一个任务完成时返回。
* `FIRST_EXCEPTION`：第一个任务抛出异常时返回（无异常则等所有完成）。
* `ALL_COMPLETED`：所有任务完成时返回（默认）。

### 3、同步代码的异步化：兼容旧库

实际开发中常需在异步程序中调用同步阻塞库（如 `requests`、`pymysql`），直接调用会阻塞事件循环，需通过**线程池**异步执行。

#### 3.1 核心方法：`loop.run_in_executor()`

该方法将同步函数提交到线程池执行，返回 Future 对象，可通过 `await` 获取结果。

```
import asyncio
import requests  # 同步阻塞库

# 同步函数（阻塞）
def sync_get(url):
    return requests.get(url).status_code

async def async_get(url):
    # 获取事件循环
    loop = asyncio.get_event_loop()
    # 提交到线程池执行（None 表示使用默认线程池）
    future = loop.run_in_executor(
        None,  # 线程池执行器（可选自定义）
        sync_get,  # 同步函数
        url  # 函数参数
    )
    return await future  # 等待线程池结果

async def main():
    urls = ["https://www.baidu.com", "https://www.github.com"]
    # 并发执行同步函数的异步包装
    results = await asyncio.gather(*[async_get(url) for url in urls])
    print("结果:", results)  # [200, 200]

asyncio.run(main())
```

#### 3.2 自定义线程池

默认线程池大小有限（通常为 CPU 核心数 \* 5），高并发场景可自定义线程池：

```
from concurrent.futures import ThreadPoolExecutor

async def main():
    # 自定义线程池（最大10个线程）
    executor = ThreadPoolExecutor(max_workers=10)
    loop = asyncio.get_event_loop()
    
    # 使用自定义线程池
    future = loop.run_in_executor(executor, sync_get, "https://www.baidu.com")
    print(await future)  # 200
```

### 4、异步 IO 实战

#### 4.1 异步网络请求（`aiohttp`）

`aiohttp` 是异步 HTTP 客户端 / 服务器库，支持异步请求、连接池、超时控制等，是替代同步 `requests` 的最佳选择。

**并发爬取网页（带超时与重试）**

```
import asyncio
import aiohttp
from aiohttp import ClientTimeout

async def fetch(session, url, retry=3):
    """带重试机制的异步请求"""
    timeout = ClientTimeout(total=10)  # 超时控制（10秒）
    try:
        async with session.get(url, timeout=timeout) as response:
            return {
                "url": url,
                "status": response.status,
                "length": len(await response.text())
            }
    except Exception as e:
        if retry > 0:
            print(f"请求 {url} 失败，重试 {retry-1} 次: {e}")
            await asyncio.sleep(1)  # 重试前等待1秒
            return await fetch(session, url, retry-1)
        return {"url": url, "error": str(e)}

async def main():
    urls = [
        "https://www.baidu.com",
        "https://www.github.com",
        "https://www.python.org",
        "https://invalid.url"
    ]
    
    # 创建会话（复用连接，提高效率）
    async with aiohttp.ClientSession() as session:
        # 生成任务列表
        tasks = [fetch(session, url) for url in urls]
        # 并发执行
        results = await asyncio.gather(*tasks)
    
    # 输出结果
    for res in results:
        if "error" in res:
            print(f"{res['url']}: {res['error']}")
        else:
            print(f"{res['url']} | 状态: {res['status']} | 长度: {res['length']}")

asyncio.run(main())
```

#### 4.2 异步文件操作（`aiofiles`）

传统 `open()` 是同步阻塞的，`aiofiles` 提供异步文件读写，支持 `async with` 和 `await` 语法。

**异步读写多文件**

```
import asyncio
import aiofiles

async def write_file(filename, content):
    """异步写入文件"""
    async with aiofiles.open(filename, 'w', encoding='utf-8') as f:
        await f.write(content)  # 异步写入
    print(f"已写入: {filename}")

async def read_file(filename):
    """异步读取文件"""
    async with aiofiles.open(filename, 'r', encoding='utf-8') as f:
        content = await f.read()  # 异步读取
    return filename, content

async def main():
    # 并发写入3个文件
    await asyncio.gather(
        write_file("file1.txt", "异步文件1"),
        write_file("file2.txt", "异步文件2"),
        write_file("file3.txt", "异步文件3")
    )
    
    # 并发读取文件
    files = ["file1.txt", "file2.txt", "file3.txt"]
    results = await asyncio.gather(*[read_file(f) for f in files])
    
    # 打印内容
    for name, content in results:
        print(f"{name} 内容: {content}")

asyncio.run(main())
```

#### 4.3 异步数据库操作（`aiomysql`）

`aiomysql` 是 MySQL 的异步驱动，支持异步连接、查询、事务，避免同步 `pymysql` 的阻塞问题。

**异步查询 MySQL**

```
import asyncio
import aiomysql

async def query_db():
    # 建立异步连接
    connection = await aiomysql.connect(
        host='localhost',
        port=3306,
        user='root',
        password='password',
        db='test',
        autocommit=True
    )
    
    try:
        # 创建游标
        async with connection.cursor(aiomysql.DictCursor) as cursor:
            # 异步执行查询
            await cursor.execute("SELECT * FROM users LIMIT 3")
            # 异步获取结果
            results = await cursor.fetchall()
            print("查询结果:", results)
    finally:
        # 关闭连接
        connection.close()

asyncio.run(query_db())
```

### 5、总结

Python 异步编程通过事件循环驱动的任务切换，实现了 IO 密集型任务的高效并发。核心组件包括协程（任务单元）、事件循环（调度中心）、任务（并发单元）和 Future（结果容器）。

本博客参考[wgetCloud](https://wgetcloud6.org)。转载请注明出处！
