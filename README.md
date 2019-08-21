# celery日常命令汇总

## 1. 启动

    // proj是目录，proj目录下有celery.py文件
    $ celery -A proj worker -l info

    // 或者是proj目录下的app.py文件
    $ celery -A proj.app worker -l info

## 2. 触发任务

任务调用：

- task.delay(arg1, arg2, kwarg1='x', kwarg2='y')
- task.apply_async(args=[arg1, arg2], kwargs={kwarg1:'x', kwarg2:'y'})

>task.delay(1, 2)  
>task.apply_async((1,2), compression='zlib')

    In  : from proj.tasks import add
    In  : r = add.delay(1, 2)
    In  : r
    In  : r.result
    In  : r.status
    In  : r.successful()
    In  : r.backend
    In  : task_id = "6ea61ed9-79a4-4c1e-8fd3-4e3cd0591748"
    In  : add.AsyncResult(task_id).get()  # 获取结果方式一
    In  : from celery.result import AsyncResult
    In  : from celery.result import AsyncResult
    In  : AsyncResult(task_id).get()  # 获取结果方式二

## 3. 任务调度

使用celery的beat进程自动生成任务。创建一个projb目录，对projb/celeryconfig.py添加如下配置：

```python
CELERYBEAT_SCHEDULE = {
    'add': {
        'task': 'projb.tasks.add',
        'schedule': timedelta(seconds=10),
        'args': (16, 16)
    }
}
```

CELERYBEAT_SCHEDULE中指定了tasks.add这个任务每10秒跑一次，执行的时候参数是16和16

启动beat程序：

    $ celery beat -A projb

然后启动worker进程：

    $ celery -A projb worker -l info

之后可以看到每10秒都会自动执行一次tasks.add。

>beat和worker进程可以一并启动：  
>`$ celery -A -A projb worker -l info`

## 4. worker管理

使用`multi`子命令管理worker，如用daemon方式启动worker进程：

    $ celery multi start web -A proj -l info --pidfile=/tmp/celery_%n.pid --logfile=/tmp/celery_%n.log

web是对项目启动的标识，之后都是用这个标识来管理，%n是格式化用法（表示只包含主机名，还有其它格式化）

    $ celery multi show web  # 查看web启动时的命令
    $ celery multi names web  # 获取web的节点名字
    $ celery multi stop web  # 停止web进程
    $ celery multi restart web
    $ celery multi kill web  # 杀掉web进程

## 5. 监控和管理celery

1. `shell`: python交互环境，内置了celery应用实例和全部已注册的任务  
    支持默认的Python解释器、IPython、BPython

        $ celery shell -A proj
        In  : celery
        In  : add.delay(1, 2)

2. `result`: 通过task_id在命令行获得执行结果

        $ celery -A proj result 6ea61ed9-79a4-4c1e-8fd3-4e3cd0591748

3. `inspect active`: 列出当前正在执行的任务

        $ celery -A proj inspect active

4. `inspect stats`: 列出worker的统计数据，用来查看配置是否正确及系统的使用情况

        $ celery -A proj inspect stats

## 6. 子任务

可以把任务通过签名的方法传给其它任务，成为一个子任务：

    In  : from celery import signature
    In  : task = signature('tasks.add', args(2,2), countdown=10)
    In  : task
    In  : task.apply_async()

还可以通过如下方式创建子任务：

    In  : from proj.tasks import add
    In  : task = add.subtask((2,2), countdown=10)
    In  : task.apply_async()

add.subtask同样可以使用快捷方式`add.s(2, 2, countdown=10)`

子任务实现偏函数（Partial）的方式非常有用，这种方式可以让任务在传递过程中才传入参数

    In  : partial = add.s(2)
    In  : partial.apply_async((4,))

子任务支持如下5种原语来实现工作流。原语表示由若干条指令组成的，用于完成一定功能的过程。

1. `chain`: 调用链。前面的执行结果作为参数传给后面的任务，直到全部完成。

        In  : from celery import chain
        In  : res = chain(add.s(2,2), add.s(4), add.s(8))()
        In  : res.get()
        // chain也可以使用管道(|):
        In  : (add.s(2,2) | add.s(4) | add.s(8))().get()

2. `group`: 一次创建多个（一组）任务。

        In  : from celery import group
        In  : res = group(add.s(i, i) for i in range(10))()
        In  : res.get()

3. `chord`: 等待任务全部完成时添加一个回调任务。

        In  : res = chord((add.s(i,i) for i in range(10)), add.s(['a]))()
        In  : res.get()

4. `map/starmap`: 每个参数都作为任务的参数执行一遍，`map`的参数只有一个，`starmap`支持的参数有多个。

        In  : ~add.starmap(zip(range(10), range(10)))
        
    // 相当于
    ```python
    @app.task
    def temp():
    return [add(i,i) for i in range(10)]
    ```

5. `chunks`: 将任务分块。

        In  : res = add.chunks(zip(range(50), range(50)), 10)()
        In  : res.get()
