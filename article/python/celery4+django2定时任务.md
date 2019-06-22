网上有很多celery + django实现定时任务的教程，不过它们大多数是基于djcelery + celery3的；
或者是使用django_celery_beat配置较为繁琐的。

显然简洁而高效才是我们最终的追求，而celery4已经不需要额外插件即可与django结合实现定时任务了，原生的celery beat就可以很好的实现定时任务功能。

当然使用原生方案的同时有几点插件所带来的好处被我们放弃了：

- 插件提供的定时任务管理将不在可用，当我们只需要任务定期执行而不需要人为调度的时候这点忽略不计。
- 无法高效的管理或追踪定时任务，定时任务的跟踪其实交给日志更合理，但是对任务的修改就没有那么方便了，不过如果不需要经常变更/增减任务的话这点也在可接受范围内。

## Celery定时任务配置

在进行配置前先来看看项目结构：

```text
.
├── linux_news
│   ├── celery.py
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── manage.py
├── news
│   ├── admin.py
│   ├── apps.py
│   ├── __init__.py
│   ├── migrations
│   ├── models
│   ├── tasks.py
│   ├── tests.py
│   └── views
└── start-celery.sh
```

其中news是我们的app，用于从一些rss订阅源获取新闻信息，linux_news则是我们的project。我们需要关心的主要是<span style="color:red">celery.py</span>，<span style="color:red">settings.py</span>，<span style="color:red">tasks.py</span>和<span style="color:red">start-celery.sh</span>。

首先是celery.py，想让celery执行任务就必须实例化一个celery app，并把settings.py里的配置传入app：

```python
import os
from celery import Celery

# set the default Django settings module for the 'celery' program.
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'linux_news.settings')

app = Celery('linux_news')

# 'django.conf:settings'表示django,conf.settings也就是django项目的配置，celery会根据前面设置的环境变量自动查找并导入
# - namespace表示在settings.py中celery配置项的名字的统一前缀，这里是'CELERY_'，配置项的名字也需要大写
app.config_from_object('django.conf:settings', namespace='CELERY')

# Load task modules from all registered Django app configs.
app.autodiscover_tasks()
```

配置就是这么简单，为了能在django里使用这个app，我们需要在__init__.py中导入它：

```python
from .celery import app as celery_app
```

然后我们来看tasks.py，它应该位于你的app目录中，前面我们配置了自动发现，所以celery会自动找到这些tasks，我们的tasks将写在这一模块中，代码涉及了一些orm的使用，为了契合主题我做了些精简：

```python
from linux_news.celery import celery_app as app
from .models import *
import time
import feedparser
import pytz
import html


@app.task(ignore_result=True)
def fetch_news(origin_name):
    """
    fetch all news from origin_name
    """
    origin = get_feeds_origin(origin_name)
    feeds = feedparser.parse(origin.feed_link)
    for item in feeds['entries']:
        entry = NewsEntry()
        entry.title = item.title
        entry.origin = origin
        entry.author = item.author
        entry.link = item.link
        # add timezone
        entry.publish_time = item.time.replace(tzinfo=pytz.utc)
        entry.summary = html.escape(item.summary)

        entry.save()


@app.task(ignore_result=True)
def fetch_all_news():
    """
    这是我们的定时任务
    fetch all origins' news to db
    """
    origins = NewsOrigin.objects.all()
    for origin in origins:
        fetch_news.delay(origin.origin_name)
```

tasks里是一些耗时操作，比如网络IO或者数据库读写，因为我们不关心任务的返回值，所以使用`@app.task(ignore_result=True)`将其屏蔽了。

任务配置完成后我们就要配置celery了，我们选择redis作为任务队列，我强烈建议在生产环境中使用rabbitmq或者redis作为任务队列或结果缓存后端，而不应该使用关系型数据库：

```python
# redis
REDIS_PORT = 6379
REDIS_DB = 0
# 从环境变量中取得redis服务器地址
REDIS_HOST = os.environ.get('REDIS_ADDR', 'redis')

# celery settings
# 这两项必须设置，否则不能正常启动celery beat
CELERY_ENABLE_UTC = True
CELERY_TIMEZONE = TIME_ZONE
# 任务队列配置
CELERY_BROKER_URL = f'redis://{REDIS_HOST}:{REDIS_PORT}/{REDIS_DB}'
CELERY_ACCEPT_CONTENT = ['application/json', ]
CELERY_RESULT_BACKEND = f'redis://{REDIS_HOST}:{REDIS_PORT}/{REDIS_DB}'
CELERY_TASK_SERIALIZER = 'json'
```

然后是我们的定时任务设置：

```python
from celery.schedules import crontab
CELERY_BEAT_SCHEDULE={
        'fetch_news_every-1-hour': {
            'task': 'news.tasks.fetch_all_news',
            'schedule': crontab(minute=0, hour='*/1'),
        }
}
```

定时任务配置对象是一个dict，由任务名和配置项组成，主要配置想如下：

- task：任务函数所在的模块，模块路径得写全，否则找不到将无法运行该任务
- schedule：定时策略，一般使用`celery.schedules.crontab`，上面例子为每小时的0分执行一次任务，具体写法与linux的crontab类似可以参考文档说明
- args：是个元组，给出任务需要的参数，如果不需要参数也可以不写进配置，就像例子中的一样
- 其余配置项较少用，可以参考文档
至此，配置celery beat的部分就结束了。

## 启动celery beat

配置完成后只需要启动celery了。

启动之前配置一下环境。不要用root运行celery！不要用root运行celery！不要用root运行celery！重要的事情说三遍。

start-celery.sh：

```bash
export REDIS_ADDR=127.0.0.1

celery -A linux_news worker -l info -B -f /path/to/log
```

-A 表示app所在的目录，-B表示启动celery beat运行定时任务。
celery正常启动后就可以通过日志来查看任务是否正常运行了：

```text
[2018-12-21 13:00:00,022: INFO/MainProcess] Received task: news.tasks.fetch_all_news[e4566ede-2cfa-4c19-b2f3-0c7d6c38690d]  
[2018-12-21 13:00:00,046: INFO/MainProcess] Received task: news.tasks.fetch_news[583e96dc-f508-49fa-a24a-331e0c07a86b]  
[2018-12-21 13:00:00,051: INFO/ForkPoolWorker-2] Task news.tasks.fetch_all_news[e4566ede-2cfa-4c19-b2f3-0c7d6c38690d] succeeded in 0.02503809699555859s: None
[2018-12-21 13:00:00,052: INFO/MainProcess] Received task: news.tasks.fetch_news[c61a3e55-dd3c-4d49-8d6d-ca9b1757db25]  
[2018-12-21 13:00:00,449: INFO/ForkPoolWorker-5] Task news.tasks.fetch_news[c61a3e55-dd3c-4d49-8d6d-ca9b1757db25] succeeded in 0.39487219898728654s: None
[2018-12-21 13:00:00,606: INFO/ForkPoolWorker-3] Task news.tasks.fetch_news[583e96dc-f508-49fa-a24a-331e0c07a86b] succeeded in 0.5523456179944333s: None
```

以上就是celery4运行定时任务的内容，如有错误和疏漏，欢迎指正。
