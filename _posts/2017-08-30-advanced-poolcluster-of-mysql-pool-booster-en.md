---
layout: default
title:  "Try the improved PoolCluster"
is_english: true
og_title : "Try the improved PoolCluster"
og_description: "This article describes how to use the improved PoolCluster in the Node.js + mysql module environment."
og_image: "ifsnow_ver2.png"
og_image_width: 200
og_image_height: 200
fb_like: true
fb_comments: true
date:   2017-08-30 00:00:00
categories: nodejs
meta_description: "This article describes how to use the improved PoolCluster in the Node.js + mysql module environment."
---

This article is useful for those who use the connection pool in the Nodes.js + mysql module environment. This is the second part of introducing the `mysql-pool-booster` module, which allows you to use the connection pool with better performance and functionality than original module.

1. [Performance booster for the pool of mysql (node.js)](/nodejs/2017/07/27/mysql-pool-booster-en.html)
2. Try the improved PoolCluster

# # PoolCluster?
Have you ever heard about `PoolCluster`? This feature was added to `mysql` module in 2013, but many people who have used `mysql` module for a long time will not know that. (It was my first contribution for open source project.)

`PoolCluster` provides the functionality to easily handle multiple connection pools. Please see this [official document](https://github.com/mysqljs/mysql/blob/master/Readme.md#poolcluster) if you want to know about basic usage and options more.

Suddenly, I thought I had to improve this feature. Just like my fate.

# # The beginning of improvement
For example, you have the following server groups.

| Server group | Description |
| --- | --- |
| core | Core information is stored (Master, Slave) |
| data-* | Sharded data is stored |

1. Execute an INSERT query from a master server in the core group.
2. Get the information from a slave server in the core group.
3. Execute a SELECT query from one of servers in the data group.

The implementation using `PoolCluster` is as follows.

{% highlight javascript %}
const poolCluster = mysql.createPoolCluster({...});

poolCluster.add('core-master', {
  host : '...',
  user : '...',
  password : '...'
});

poolCluster.add('core-slave', {
  host : '...',
  ...
});

poolCluster.add('data-math', {
  host : '...',
  ...
});

poolCluster.add('data-english', {
  host : '...',
  ...
});

poolCluster.getConnection('core-master', (err, connection) => {
  connection.query('INSERT log ...');
  connection.release();
});

poolCluster.getConnection('core-slave', (err, connection) => {
  connection.query('SELECT category FROM data_info WHERE seq=?', [seq], (err, results) => {
    connection.release();

    const clusterId = `data-${results[0].category}`;
    poolCluster.getConnection(clusterId, (err, connection) => {
      connection.query('SELECT ...', (err, rows) => {
        // ...
      });
    });  
  });
});
{% endhighlight %}

I thought there were a lot of things to improve.

# # The direction of improvement

Three functions have been added.

## 1. Create with options

I usually define the configuration as JSON, so adding a node using the `add` function is inconvenient. That's because I need to make additional codes (logic to read and initialize the configuration). Now, you can create it with options(`nodes` > `clusterId`) instead of using `add` function.

{% highlight javascript %}
// original version using the function
const cluster = mysql.createPoolCluster({...});

cluster.add('master', {
  host : '...',
  ...
});

cluster.add('slave', {
  host : '...',
  ...
});

// booster version using the options
const cluster = mysql.createPoolCluster({
  ....,
  nodes : [{
    clusterId : 'master',
    host : '...',
    ...
  }, {
    clusterId : 'slave',
    host : '...',
    ...
  }]
});
{% endhighlight %}

## 2. Writer & Reader
In many cases, many people will use the connection pool consisting of a master and a slave server. In this case, I think it's easier to use by accessing the concept of Writer and Reader rather than the ID base defined. You can set it up with options(`nodes` > `clusterType`) or functions(`addWriter`, `addReader`, `add`).

{% highlight javascript %}
/**
 * 4 nodes are created.
 * # Writer : No ID
 * # Reader : main, sub, sub2
 */
const cluster = mysql.createPoolCluster({
  ....,
  nodes : [{
    clusterType : 'writer', // without clusterId
    host : '...',
    ...
  }, {
    clusterType : 'reader', // with clusterId
    clusterId : 'main',
    host : '...',
    ...
  }
});

cluster.addReader('sub', {
  host : '...',
  ...
});

cluster.add({  
  clusterType : mysql.CLUSTER_TYPE.READER, // mysql.CLUSTER_TYPE.WRITER
  clusterId : 'sub2',
  host : '...',
  ...
});

// You can get a connection of the Writer group's nodes right away.
// the same as cluster.getWriter().getConnection(...)
cluster.getWriterConnection((err, connection) => {
  ...
});

// You can get a connection from all of the Reader group's nodes (`main`,` sub`, `sub2`)
cluster.getReaderConnection((err, connection) => {
  ...
});

// You can get a connection from specific Reader group's nodes (`sub`, `sub2`)
// the same as cluster.getReader('sub*').getConnection(function(err, connection)
cluster.getReaderConnection('sub*', (err, connection) => {
  ...
});
{% endhighlight %}

## 3. Sharding
If you want to use the application-level [sharding](https://goo.gl/9DdfAK), all you need to do is set it up with options(`shardings`) or `addSharding` function. In complex cases, I think you'd better use another professional sharding platform.

{% highlight javascript %}
const cluster = mysql.createPoolCluster({
  ....,
  nodes : [{
    clusterId : 'old',
    host : '...',
    ...
  }, {
    clusterId : 'new',
    host : '...',
    ...
  }],
  shardings : {
    byUserSeq : (user) => {
      return user.seq > 10000000 ? 'new' : 'old';
    }
  }
});

cluster.addSharding('byTwoParams', (a, b) => {
  return a + b > 10 ? 'new' : 'old';
});
{% endhighlight %}

You can use the `getSharding` or `getShardingConnection`(`getShardingReaderConnection`, `getShardingWriterConnection`) function with argument to get a connection.

{% highlight javascript %}
const user = {
  seq : 50000
};

// The connection is based on the user parameter. (In this case, 'old')
// the same as cluster.getSharding('byUserSeq', user).getConnection(...)
cluster.getShardingConnection('byUserSeq', user, (err, connection) => {
  ...
});

// The argument must be an array type if there is more than one.
cluster.getShardingConnection('byTwoParams', [valueA, valueB], (err, connection) => {
  ...
});

// You must use getReaderConnection() or getWriterConnection() if the nodes are defined as Writer&Reader.
// the same as cluster.getShardingReaderConnection('byUserNumber', user, ...)
cluster.getSharding('byUserNumber', user).getReaderConnection((err, connection) => {
  ...
});
{% endhighlight %}

# # The result of improvement

Here's an example of what you've seen in the previous version. Do you think it's a little simpler?

{% highlight javascript %}
const poolCluster = mysql.createPoolCluster({
  ...
  nodes : [{
    clusterType : mysql.CLUSTER_TYPE.WRITER,
    clusterId : 'core',
    host : '...',
    ...
  }, {
    clusterType : mysql.CLUSTER_TYPE.READER,
    clusterId : 'core',
    host : '...',
    ...
  }, {
    clusterId : 'data-math',
    host : '...',
    ...
  }, {
    clusterId : 'data-english',
    host : '...',
    ...
  }],
  shardings : {
    dataByCategory : (dataInfo) => `data-${dataInfo.category}`
  }
});

poolCluster.getWriterConnection((err, connection) => {
  connection.query('INSERT log ...');
  connection.release();
});

poolCluster.getReaderConnection((err, connection) => {
  connection.query('SELECT category FROM data_info WHERE seq=?', [seq], (err, results) => {
    connection.release();

    poolCluster.getShardingConnection('dataByCategory', results, (err, connection) => {
      connection.query('SELECT ...', (err, rows) => {
        // ...
      });
    });  
  });
});
{% endhighlight %}

# # Using the improved version

Install the `mysql-pool-booster` module with the` mysql` module installed,

{% highlight bash %}
npm install mysql-pool-booster
{% endhighlight %}

If you convert an existing mysql object, you can use the improved features.

{% highlight javascript %}
let mysql = require('mysql');

// Converting an existing mysql object
const MysqlPoolBooster = require('mysql-pool-booster');
mysql = MysqlPoolBooster(mysql);

// Use it just as you used it
mysql.createPool({ ... });
mysql.createPoolCluster({ ... });
{% endhighlight %}

# # If you have any problems
Please leave an issue in [github](https://github.com/ifsnow/mysql-pool-booster) or send [me](mailto:ifsnow@gmail.com) an e-mail. I'll do my best to solve your problem as soon as possible. I have tried to make it work without problems, but since it's a new project that has just started, please apply it to your service after enough verification steps.

[View on github](https://github.com/ifsnow/mysql-pool-booster)

I hope this helps. thanks.
