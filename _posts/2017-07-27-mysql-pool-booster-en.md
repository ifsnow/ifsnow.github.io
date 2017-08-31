---
layout: default
title:  "Performance booster for the pool of mysql (node.js)"
is_english: true
og_title : "mysql-pool-booster"
og_description: "Performance booster for the pool of mysql (node.js)"
og_image: "ifsnow_ver2.png"
og_image_width: 200
og_image_height: 200
fb_like: true
fb_comments: true
date:   2017-07-27 00:00:00
categories: nodejs
meta_description: "It easily converts the pool of mysql to more faster and improved version. By applying one line, you can use everything. Please try it once if you're already using the pool of mysql."
---

This is for those who use the connection pool in Node.js + mysql module environment.

1. Performance booster for the pool of mysql (node.js)
2. [Try the improved PoolCluster](/nodejs/2017/08/30/advanced-poolcluster-of-mysql-pool-booster-en.html)

# # mysql-pool-booster

By adding just a few lines, you can use the connection pool with better performance and useful options than orginal.

# # The reason I made

I sometimes contribute to the mysql module project whenever I was bored. I got the feeling that maintainers were prudent about applying a little change request([pull request # 1779](https://github.com/mysqljs/mysql/pull/1779)). I know it's because of a big project with a long history and a lot of users. I fully understand and respect the devotion and passion of those who maintain open source projects.

So, I have created a new module that does not affect existing modules, but can use the proposed features. I hope I have someone who needs this module besides me.

# # Features

## 1. Performance improvements

Applying without additional work, your service can be improved 80% or more efficiently.

This is the result of a [performance measurement code](https://gist.github.com/ifsnow/5cc2a628574c2708eb91231c1abe92cd) that handles 30,000 requests.

|  | Original | Booster  |
| --- | --- | --- |
| Processing per second | 5,877  | 10,650 (181% ↑) |

You can experience a better effect if your service connects to remote mysql server and handles heavy concurrent requests.

## 2. Useful options

The default option for the connection pool in the mysql module is so simple. `connectionLimit` is that the maximum number of connections that can be created, and  `queueLimit` is that the maximum size of queues to wait for the connection if there are no available connections.

There is no problem with normal use, but it was so hard to remove unnecessary connections from the temporarily growing pool. Now, `mysql-pool-booster` gives you the flexibility to run the pool with many useful options similar to Java's DBCP.

| Option  | Default | Description |
| --- | --- | --- |
| queueTimeout | 0 (off) | The maximum number of milliseconds that the queued request will wait for a connection when there are no available connections. If set to `0`, wait indefinitely. |
| testOnBorrow | true | Indicates whether the connection is validated before borrowed from the pool. If the connection fails to validate, it is dropped from the pool. |
| testOnBorrowInterval | 20000 | The number of milliseconds that indicates how often to validate if the connection is working since it was last used. If set too low, performance may decrease on heavy loaded systems. If set to `0`, It is checked every time. |
| initialSize | 0 (off) | The initial number of connections that are created when the pool is started. If set to `0`, this feature is disabled. |
| maxIdle | 10 | The maximum number of connections that can remain idle in the pool. If set to `0`, there is no limit. |
| minIdle | 0 (off) | The minimum number of connections that can remain idle in the pool. |
| maxReuseCount | 0 (off) | The maximum connection reuse count allows connections to be gracefully closed and removed from the connection pool after a connection has been borrowed a specific number of times. If set to `0`, this feature is disabled. |
| timeBetweenEvictionRunsMillis | 0 (off) | The number of milliseconds to sleep between runs of examining idle connections. The eviction timer will remove existent idle conntions by `minEvictableIdleTimeMillis` or create new idle connections by `minIdle`. If set to `0`, this feature is disabled. |
| minEvictableIdleTimeMillis | 1800000 | The minimum amount of time the connection may sit idle in the pool before it is eligible for eviction due to idle time. If set to `0`, no connection will be dropped. |
| numTestsPerEvictionRun | 3 | The number of connections to examine during each run of the eviction timer (if any). |

# # How to use

- Install `mysql-pool-booster`

{% highlight bash %}
npm install mysql-pool-booster
{% endhighlight %}

- Convert `mysql`

{% highlight javascript %}
var mysql = require('mysql');

// Converting an existing mysql object
var MysqlPoolBooster = require('mysql-pool-booster');
mysql = MysqlPoolBooster(mysql);

// You can use it just as you used it
mysql.createPool({ ... });
{% endhighlight %}

- Use the `mysql` just as it used to be

# # Restrictions on use
I think there's no problem with normal use becuase it passes all tests provided by the mysql module and provides 100% compatibility. However, there may be some problem if you are using it incorrectly such as accessing private underscore-prefix properties. For example.

{% highlight javascript %}
// It's a misuse. you must not access private variables
if (pool._allConnections.length > 0) {
  ...
}

// You should use the following method instead
if (pool.getStatus().all > 0) {
  ...
}
{% endhighlight %}

# # If you have any problems
Please leave an issue in [github](https://github.com/ifsnow/mysql-pool-booster) or send [me](mailto:ifsnow@gmail.com) an e-mail. I'll do my best to solve your problem as soon as possible. I have tried to make it work without problems, but since it's a new project that has just started, please apply it to your service after enough verification steps.

[View on github](https://github.com/ifsnow/mysql-pool-booster)

I hope this helps. thanks.
