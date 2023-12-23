---
title: 定时任务实践（Python向）
date: 2016-02-28 16:25:03
tags:
    - Python
categories: 工具
---

之所以会有这篇博客其主要原因是因为最近需要写一些脚本用于定时运行，而我个人对于Shell的crontab又是处于看了就忘，忘了就不想看的阶段，Python在我目前看来是最适合替换掉Shell脚本的手段，其中自然也有大量定时运行的技术手段，这篇对于三种可使用不同方法执行定时任务的库做个介绍，**仅仅只是简单介绍，如果需要详细了解恐怕还是看其官方文档来得更加适合**。

<!--more-->
## Plan

[官方文档​](http://plan.readthedocs.org/)

Plan其实是借用Python创建一条crontab的命令，然后依据crontab来运行
### 安装

``` Python
pip install plan
```
### 基本用法
1. 创建一个Plan实例(`cron = Plan()`)
2. 向其实例添加一个实时运行的命令，脚本或模块
   1. `cron.command('command', every='1.day')`
   2. `cron.script('script.py', path='/web/yourproject/scripts', every='1.month')`
   3. `cron.module('calendar', every='feburary', at='day.3')`
3. 运行(`cron.run()`)
### 关键方法

无论是command，script或是module其实在Plan中都可以称之为Job，我们甚至可以创建自己的Job，先让我们来了解一下Plan中的Job。
#### Job

其实Job就相当于crontab中的一条命令，它带有`task`, `every`, `at`, `path`, `environment` 与 `output`，这些参数含义大概从名字来看也可猜出一二。

**task**指定运行任务名

其中**every参数**是指定其运行周期性，其：

``` Python
[1-60].minute
[1-24].hour
[1-31].day
[1-12].month
jan feb mar apr may jun jul aug sep oct nov dec
and all of those full month names(case insensitive)
sunday, monday, tuesday, wednesday, thursday, friday, saturday
weekday, weekend (case insensitive)
[1].year
```

当然你也可以用原始的crontab的时间表示方法来表示，如下：

`job = Job('demo', every='1,2 5,6 * * 3,4')`

其中还可以指定一些特殊的值，如果指定这些值则at参数将会被忽略:

``` Python
"yearly"    # Run once a year at midnight on the morning of January 1
"monthly"   # Run once a month at midnight on the morning of the first day
            # of the month
"weekly"    # Run once a week at midnight on Sunday morning
"daily"     # Run once a day at midnight
"hourly"    # Run once an hour at the beginning of the hour
"reboot"    # Run at startup
```

**at参数**则是指定于合适的时间运行，具体实例

``` Python
job = Job('onejob', every='1.day', at='hour.12 minute.15 minute.45')
# or even better
job = Job('onejob', every='1.day', at='12:15 12:45')
```

**path**指定任务(文件)所在路径

**environment**则预设系统环境变量

**output**指定标准输出文件名，这里介绍其中最直接的一种写法

``` Python
job = Job('job', every='1.day', output=dict(stdout='/tmp/stdout.log',stderr='/tmp/stderr.log'))`
```
#### 自定义Job

总体来说command，script， module命令都是Plan库预设的一些Job，这些Job已经符合我们大多数需求了，如果我们由于某些特殊需求需要设定一些自己的Job，可以参照Plan库设置的这三个Job来进行设置，其中Script源码如下：

``` Python

class ScriptJob(Job):
    """The script job.
    """

    def task_template(self):
        """Template::
            'cd {path} && {environment} %s {task} {output}' % sys.executable
        """
        return 'cd {path} && {environment} %s {task} {output}' % sys.executable
```

可见，定义一个新的Job实在是有些简单，只需要将Job类中的`task_template`方法重写即可。
## Sched

其实这个库非常简单，就几个方法，其中我们需要用到的方法只有两个,`enter`, `run`

实例如下：

``` Python
>>> import sched, time
>>> s = sched.scheduler(time.time, time.sleep)
>>> def print_time(): print "From print_time", time.time()
...
>>> def print_some_times():
...     print time.time()
...     s.enter(5, 1, print_time, ())
...     s.enter(10, 1, print_time, ())
...     s.run()
...     print time.time()
...
>>> print_some_times()
930343690.257
From print_time 930343695.274
From print_time 930343700.273
930343700.276
```

这里我们可以看出这玩意其实只是指定某个时间运行某种方法，但是如果我们变一下同样可以周期运行：

``` Python
import sched, time

s = sched.scheduler(time.time, time.sleep)
def print_time(): 
    s.enter(5, 1, print_time, ())
    print "From print_time", time.time()

def print_some_times():
    print time.time()
    s.enter(5, 1, print_time, ())
    s.run()
    print time.time()
```

这样就形成了一个周期循环的函数，其实这种方法与我们常用的死循环相同

``` Python
import time

def loop():
    _t = 1
    while True:
        time.sleep(1)
        _t += 1
        if _t >= 5:
            print "time out"
            break
    test()

def test():
    print "get"
    loop()

test()
```

而其实sched其实也与这段代码有些类似，下面是它的部分源码:

``` Python
class scheduler:
    def __init__(self, timefunc, delayfunc):
        """Initialize a new instance, passing the time and delay
        functions"""
        self._queue = [] # 需要运行的事件
        self.timefunc = timefunc
        self.delayfunc = delayfunc

    def enterabs(self, time, priority, action, argument):
        """Enter a new event in the queue at an absolute time.

        Returns an ID for the event which can be used to remove it,
        if necessary.

        """
        event = Event(time, priority, action, argument)
        heapq.heappush(self._queue, event)
        return event # The ID

    def enter(self, delay, priority, action, argument):
        """A variant that specifies the time as a relative time.

        This is actually the more commonly used interface.

        """
        time = self.timefunc() + delay
        return self.enterabs(time, priority, action, argument)

    def run(self):
        # localize variable access to minimize overhead
        # and to improve thread safety
        q = self._queue
        delayfunc = self.delayfunc
        timefunc = self.timefunc
        pop = heapq.heappop
        while q:
            time, priority, action, argument = checked_event = q[0]
            now = timefunc()
            if now < time:
                delayfunc(time - now) # 相当于死循环中的sleep
            else:
                event = pop(q)
                # Verify that the event was not removed or altered
                # by another thread after we last looked at q[0].
                if event is checked_event:
                    action(*argument)
                    delayfunc(0)   # Let other threads run
                else:
                    heapq.heappush(q, event)
```

不过我们也从源码中看出，这个库针对单线程的还好，针对多线程的库来说程序是极有可能出错的(原因之一它是通过是否含有queue来确定是否继续运行)。

如果是多线程的话推荐使用threading.Timer库来做定时任务。这里贴个实例就不详细介绍了:

``` Python
>>> import time
>>> from threading import Timer
>>> def print_time():
...     print "From print_time", time.time()
...
>>> def print_some_times():
...     print time.time()
...     Timer(5, print_time, ()).start()
...     Timer(10, print_time, ()).start()
...     time.sleep(11)  # sleep while time-delay events execute
...     print time.time()
...
>>> print_some_times()
930343690.257
From print_time 930343695.274
From print_time 930343700.273
930343701.301
```
## Advanced Python Scheduler

[官方文档​](https://apscheduler.readthedocs.org/en/latest/)

这个库说实话真是满足了各种需求，在这三个用法中我最后实践选择的就是这个，符合各种需求，多种方式运行，还可以多种方式来持久化运行记录，如果真要了解它的详细用法估计不得不读一下他那比较多的官方文档，这里只能是介绍常用用法。
### 安装

```
pip install apscheduler

```

``` Python
#!/usr/bin/env python
# coding=utf-8

import os

from apscheduler.schedulers.blocking import BlockingScheduler

from etc import config
from scripts import auto_script

BASE_PATH = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
LOOP_TIME = 3 # 3.day
MYSQL_URL = "mysql://{mysql_user}:{mysql_password}@{mysql_host}/{mysql_dbname}".format(
    mysql_user=config.mysql_user,
    mysql_password=config.mysql_password,
    mysql_host=config.mysql_host,
    mysql_dbname=config.job_store_db_name
)

if __name__ == "__main__":
    sched = BlockingScheduler()
    sched.add_jobstore("sqlalchemy", url=MYSQL_URL)
    sched.add_job(func=auto_script.run, trigger="interval", days=LOOP_TIME)
    sched.start()

```

简单来介绍一下:
- 这里先是创建了一个阻塞类型的Scheduler
- 并使用Mysql作为其持久化数据库用来保存期运行记录，其中也非使用的是完全的Mysql官方提供的库，而是使用了Python中最常用的ORM库SqlAlchemy
- 添加了一个job，其指定运行auto_script脚本中的run方法，运行方式(`trigger`)是间隔时间，间隔时间为LOOP_TIME
### Scheduler类型

``` Python
BlockingScheduler # 在前端运行
BackgroundScheduler # 作为后台进程运行
AsyncIOScheduler # 如果程序中使用了asyncio模块，可使用这个来运行
GeventScheduler # 如果程序中使用了gevent模块，可使用这个来运行
TornadoScheduler # 如果是构建一个Tornado应用，可使用这个来运行
TwistedScheduler # 如果是构建一个Twisted应用，可使用这个来运行
QtScheduler # 如果是构建一个Qt应用，可使用这个来运行
```
### triggers

**apscheduler.triggers.cron**: 与Cron指定执行方式相同，同时也可指定开始与结束时间

``` Python
class apscheduler.triggers.cron.CronTrigger(year=None, month=None, day=None, week=None, day_of_week=None, hour=None, minute=None, second=None, start_date=None, end_date=None, timezone=None)
```

**apscheduler.triggers.interval**:  间隔时间周期执行，可指定开始与结束时间   

``` Python
class apscheduler.triggers.interval.IntervalTrigger(weeks=0, days=0, hours=0, minutes=0, seconds=0, start_date=None, end_date=None, timezone=None)
```

**apscheduler.triggers.date**:  特定时间执行一次

``` Python
class apscheduler.triggers.date.DateTrigger(run_date=None, timezone=None)
```

通过 `add_job`方法可指定其运行周期方式，而其中参数也由add_job来指定，从其源码来看该方法开头:

``` Python
def add_job(self, func, trigger=None, args=None, kwargs=None, id=None, name=None,
                misfire_grace_time=undefined, coalesce=undefined, max_instances=undefined,
                next_run_time=undefined, jobstore='default', executor='default',
                replace_existing=False, **trigger_args):
       job_kwargs = {
            'trigger': self._create_trigger(trigger, trigger_args), # 在此处指定对应的trigger参数
            'executor': executor,
            'func': func,
            'args': tuple(args) if args is not None else (),
            'kwargs': dict(kwargs) if kwargs is not None else {},
            'id': id,
            'name': name,
            'misfire_grace_time': misfire_grace_time,
            'coalesce': coalesce,
            'max_instances': max_instances,
            'next_run_time': next_run_time
        }
        job_kwargs = dict((key, value) for key, value in six.iteritems(job_kwargs) if
                          value is not undefined)
        job = Job(self, **job_kwargs)

        # Don't really add jobs to job stores before the scheduler is up and running
        with self._jobstores_lock:
            if not self.running:
                self._pending_jobs.append((job, jobstore, replace_existing))
                self._logger.info('Adding job tentatively -- it will be properly scheduled when '
                                  'the scheduler starts')
            else:
                self._real_add_job(job, jobstore, replace_existing, True)

        return job

```
### Job Store

通过`add_jobstore(jobstore, alias='default', **jobstore_opts)`指定对应的持久化数据库

可添加的有Mongo， Sqlalchemy类(几乎所有常用关系型数据库)， Redis

同时还可以通过在创建`Scheduler`实例时候来指定Store

``` Python
...
jobstores = {
    'mongo': MongoDBJobStore(),
    'default': SQLAlchemyJobStore(url='sqlite:///jobs.sqlite')
}
...
scheduler = BackgroundScheduler(jobstores=jobstores, executors=executors, job_defaults=job_defaults, timezone=utc)
...
scheduler = BackgroundScheduler({
    'apscheduler.jobstores.mongo': {
         'type': 'mongodb'
    },
    'apscheduler.jobstores.default': {
        'type': 'sqlalchemy',
        'url': 'sqlite:///jobs.sqlite'
    },
    'apscheduler.executors.default': {
        'class': 'apscheduler.executors.pool:ThreadPoolExecutor',
        'max_workers': '20'
    },
    'apscheduler.executors.processpool': {
        'type': 'processpool',
        'max_workers': '5'
    },
    'apscheduler.job_defaults.coalesce': 'false',
    'apscheduler.job_defaults.max_instances': '3',
    'apscheduler.timezone': 'UTC',
})
...
jobstores = {
    'mongo': {'type': 'mongodb'},
    'default': SQLAlchemyJobStore(url='sqlite:///jobs.sqlite')
}
scheduler = BackgroundScheduler()
scheduler.configure(jobstores=jobstores, executors=executors, job_defaults=job_defaults, timezone=utc)
```

在这里我们也可以看到，还可以指定`executors`已指定线程运行方式，具体可以参考官方文档，这里就不详细展开了。
### 运行

`scheduler.start()`
## 总结

如果是在工作中使用，在只是需要一个定时任务且不复杂的情况下可以使用`sched`模块，如果是任务复制则可以使用`Advanced Python Scheduler`，如果是一些可以详细拆分的文件定时运行，或者是说定时运行的不是Python脚本则可以使用`Plan`

最后，水平有限，如果有错误希望能指正并发送邮件与我联系并交流，谢谢。
