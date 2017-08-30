---
layout: default
title:  "개선된 PoolCluster 사용해보기"
og_title : "개선된 PoolCluster 사용해보기"
og_description: "Node.js + mysql 모듈 환경에서 개선된 PoolCluster를 사용하는 방법에 관한 글입니다."
og_image: "ifsnow_ver2.png"
og_image_width: 200
og_image_height: 200
fb_like: true
fb_comments: true
date:   2017-08-29 00:00:00
categories: nodejs
meta_description: "Node.js + mysql 모듈 환경에서 개선된 PoolCluster를 사용하는 방법에 관한 글입니다."
---

이 글은 Node.js + mysql 모듈 환경에서 커넥션 풀을 사용하는 분들에게 도움이 될만한 내용입니다. `mysql` 모듈에서 기본으로 제공되는 것보다 좋은 성능과 유용한 기능의 커넥션 풀을 사용할 수 있게 해주는 `mysql-pool-booster` 모듈의 두 번째 이야기입니다.

1. [Performance booster for the pool of mysql (node.js)](/nodejs/2017/07/26/mysql-pool-booster.html)
2. 개선된 PoolCluster 사용하기

# # PoolCluster?
무료하게 반복되는 개발자의 삶을 탈피해보고자 난생처음으로 오픈소스 프로젝트에 Pull Request를 보내봤는데 운영자가 기분 좋은 일이 있었는지 바로 수락해줬던 게 PoolCluster라는 이름의 기능이었습니다. 벌써 4년 전, 2013년의 일이네요. (아직 무료하고 반복되는 개발자의 삶을 살고 있습니다.)

PoolCluster는 복수의 커넥션 풀을 쉽게 운영할 수 있는 기능을 제공합니다. `mysql` 모듈을 오래 사용하셨던 분도 이런 기능이 있었는지는 몰랐을 겁니다. [공식 문서](https://github.com/mysqljs/mysql/blob/master/Readme.md#poolcluster)에서 기본 설명과 옵션을 보시면 어떤 역할의 기능인지 이해하실 수 있습니다.

버린 자식을 이제라도 보살피려는 아비의 마음으로 4년 만에 개선 작업을 해봤습니다.

# # 개선의 시작
아래 같은 서버 그룹이 있다고 가정해보겠습니다.

| 서버 그룹 | 기능 |
| --- | --- |
| core | 핵심 정보 저장 (마스터, 슬레이브) |
| data-* | 샤딩된 데이터 저장 |

(조금 억지스럽지만) core 그룹의 마스터 서버에 INSERT 쿼리를 요청하고, 슬레이브 서버에서 정보를 구한 다음 데이터 서버 그룹 중 해당하는 서버에서 SELECT 쿼리를 수행하는 로직을 `PoolCluster`로 구현하면 아래와 같습니다.

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

오랜만에 관심을 가지고 보니 제안했던 접근에 불편한 점과 개선할 수 있는 것들이 보였습니다.

# # 개선의 방향

소스를 정리하는 작업 이외에 추가된 기능은 크게 3가지입니다.

## 1. 옵션으로 생성하기
주로 설정을 JSON으로 정의해서 사용하는데 `add` 함수를 이용해서 노드를 추가하는 방식은 별도의 코드(설정을 읽어 초기화하는 로직)를 작성해야 해서 불편합니다. 이제 생성 함수의 옵션으로 노드를 바로 생성할 수 있습니다. `createPoolCluster` 함수의 옵션 중 `nodes`에 배열 형식으로 정의하면 됩니다.

{% highlight javascript %}
// 함수를 이용한 기존 방식
const cluster = mysql.createPoolCluster({...});

cluster.add('master', {
  host : '...',
  ...
});

cluster.add('slave', {
  host : '...',
  ...
});

// 옵션을 이용한 부스터 버전 방식
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
많은 경우에 마스터와 슬레이브 서버로 구성된 커넥션 풀을 운영하게 됩니다. 이 경우에 정의한 ID 기반이 아니라 Writer와 Reader라는 개념의 그룹으로 접근하면 사용이 편리해집니다. 옵션(`nodes` > `clusterType`)으로 정의하거나 `addWriter`, `addReader`, `add` 함수를 이용해서 추가할 수 있습니다.

{% highlight javascript %}
/**
 * 총 4개의 노드가 생성됩니다.
 * # Writer : ID 미지정
 * # Reader : main, sub, sub2
 */
const cluster = mysql.createPoolCluster({
  ....,
  nodes : [{
    clusterType : 'writer', // clusterId 없이 Writer로 추가
    host : '...',
    ...
  }, {
    clusterType : 'reader', // clusterId를 지정해서 추가
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

// 바로 Writer의 커넥션을 구할 수 있고
cluster.getWriterConnection((err, connection) => {
  ...
});

// Writer 그룹을 구한 후 재사용하실 수도 있습니다.
const writerCluster = cluster.getWriter();
writerCluster.getConnection((err, connection) => {
  ...
});

// 모든 Reader 그룹의 노드들(`main`, `sub`, `sub2`)에서 커넥션을 구하거나
cluster.getReaderConnection((err, connection) => {
  ...
});

// 특정 Reader 그룹의 노드들(`sub`, `sub2`)을 대상으로 할 수도 있습니다.
// cluster.getReader('sub*').getConnection(...) 으로도 사용 가능합니다.
cluster.getReaderConnection('sub*', (err, connection) => {
  ...
});
{% endhighlight %}

## 3. 샤딩 로직
어플리케이션 수준의 기본적인 샤딩을 구성할 때 사용할 대상 서버를 판단하는 로직(룰)을 정의하고 사용할 수 있는 기능을 제공합니다. 옵션(`shardings`)으로 정의하거나 `addSharding` 함수를 이용해서 추가할 수 있습니다.

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

`getSharding`이나 `getShardingConnection`(`getShardingReaderConnection`, `getShardingWriterConnection`) 함수로 매개변수에 해당하는 풀의 커넥션을 구할 수 있습니다.

{% highlight javascript %}
const user = {
  seq : 50000
};

// 전달되는 user를 기반으로 커넥션이 전달됩니다. (이 경우에는 old)
// cluster.getSharding('byUserSeq', user).getConnection(...) 으로도 사용 가능합니다.
cluster.getShardingConnection('byUserSeq', user, (err, connection) => {
  ...
});

// 매개변수가 복수일 경우 꼭 배열로 묶어 전달해야 합니다.
cluster.getShardingConnection('byTwoParams', [valueA, valueB], (err, connection) => {
  ...
});

// Writer&Reader로 정의한 노드라면 getReaderConnection(), getWriterConnection()을 사용해야 합니다.
// cluster.getShardingReaderConnection('byUserNumber', user, ...) 으로도 사용 가능합니다.
cluster.getSharding('byUserNumber', user).getReaderConnection((err, connection) => {
  ...
});
{% endhighlight %}

# # 개선의 결과

위에서 다뤘던 예를 개선된 버전으로 작성하면 아래와 같습니다. 조금 간결해진 것 같죠?

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

# # 개선된 버전 사용하기

`mysql` 모듈이 설치된 상태에서 `mysql-pool-booster` 모듈을 설치하고

{% highlight bash %}
npm install mysql-pool-booster
{% endhighlight %}

기존의 mysql 객체를 변환해주면 개선된 기능들을 사용할 수 있습니다.

{% highlight javascript %}
let mysql = require('mysql');

// 기존의 mysql 객체를 변환하고
const MysqlPoolBooster = require('mysql-pool-booster');
mysql = MysqlPoolBooster(mysql);

// 사용하던 그대로 쓰면 됩니다.
mysql.createPool({ ... });
mysql.createPoolCluster({ ... });
{% endhighlight %}

# # 문의
사용해보시고 문제가 발생할 경우 github에 issue를 남겨주시거나 [ifsnow@gmail.com](mailto:ifsnow@gmail.com)으로 보내주시면 최대한 빨리 확인해보겠습니다. 문제없이 동작할 수 있도록 만들기 위해 노력했지만 이제 막 시작된 프로젝트이기 때문에 검증하는 단계를 충분히 거친 후 실 서비스에 적용하시기 바랍니다.

[View on github](https://github.com/ifsnow/mysql-pool-booster)

누군가에겐 도움이 되길 바랍니다. 감사합니다.
