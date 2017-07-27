---
layout: default
title:  "Performance booster for the pool of mysql (node.js)"
og_title : "mysql-pool-booster"
og_description: "Performance booster for the pool of mysql (node.js)"
og_image: "ifsnow_ver2.png"
og_image_width: 200
og_image_height: 200
fb_like: true
fb_comments: true
date:   2017-07-26 00:00:00
categories: nodejs
meta_description: "It easily converts the pool of mysql to more faster and improved version. By applying one line, you can use everything. Please try it once if you're already using the pool of mysql."
---

# # mysql-pool-booster
Node.js + mysql 모듈 환경에서 커넥션 풀을 사용하는 분들을 위한 모듈입니다. 단 몇 줄만 추가하면 기본으로 제공되는 것보다 더 좋은 성능과 유용한 옵션의 커넥션 풀을 사용할 수 있게 됩니다.

# # 만들게 된 이유
심심할 때마다 mysql 모듈 프로젝트에 기여하곤 했는데 오랜 역사와 많은 사용자를 대상으로 한 프로젝트이기 때문에 조금 큰 변화의 요구([pull request #1779](https://github.com/mysqljs/mysql/pull/1779))엔 조심스럽다는 느낌을 받았습니다. 오픈 소스 프로젝트를 유지하는 분들의 선의와 고충을 이해하고 저라도 그렇게 할 것 같습니다.

그래서, 기존 모듈에 영향을 주지 않으면서 제안했던 기능을 사용할 수 있는 신규 모듈을 만들게 됐습니다. 어쩌면 저 말고도 필요한 분이 한두 분은 계실지 모르겠단 마음으로요.

# # 제공되는 기능

## 1. 성능 향상

추가 작업 없이 적용만으로 여러분의 서비스가 80% 이상 효율적으로 개선될 수 있습니다. (어디서 약을 팔..)

3만 개의 요청을 처리하는 [성능 측정 코드](https://gist.github.com/ifsnow/5cc2a628574c2708eb91231c1abe92cd)의 결과입니다.

|  | 오리지널 | 부스터 버전  |
| --- | --- | --- |
| 초당 처리 | 5,877  | 10,650 (181% ↑) |

원격 mysql 서버를 사용하면서 동시 요청이 많은 서비스에서는 더 개선된 효과를 체감하실 수 있습니다.

## 2. 유용한 옵션

mysql 모듈에서 커넥션 풀 운영을 위해 기본적으로 제공하는 옵션은 간단합니다. 생성 가능한 최대 커넥션의 수인 `connectionLimit`과 가용 커넥션이 없는 경우 대기하는 큐의 최대 크기인 `queueLimit` 정도입니다.

일반적인 사용에는 큰 무리가 없지만 저 같은 경우에는 순간적으로 사용자가 몰려 추가 생성된 커넥션을 불필요해진 시점에 제거하지 못하고 계속 유지시켜야 하는 부분들이 걸렸습니다. `mysql-pool-booster`를 적용하면 자바의 DBCP와 유사한 여러 가지 옵션으로 유연하게 풀을 운영할 수 있게 됩니다.

| 옵션  | 기본값 | 설명 |
| --- | --- | --- |
| queueTimeout | 0 (off) | 가용 커넥션이 없는 경우 요청을 큐에 넣고 순차적으로 처리하는데 이 경우의 최대 대기시간 (밀리세컨드) |
| testOnBorrow | true | 풀에서 커넥션을 꺼낸 후 요청에 전달하기 전에 커넥션의 유효성을 검사할지 말지 여부. 오류가 발생한 커넥션은 풀에서 제거됩니다. |
| testOnBorrowInterval | 20000 | 마지막으로 사용된지 설정한 시간이 지난 커넥션을 대상으로 유효성 검사 (밀리세컨드). 이 값이 너무 작을 경우 트래픽이 많은 경우 성능이 저하될 수 있습니다. 0으로 설정하면 매번 검사하게 됩니다. |
| initialSize | 0 (off) | 풀이 시작되면서 초기 생성시킬 커넥션의 수 |
| maxIdle | 10 | 풀에서 유지하는 유휴 상태 커넥션의 최대 수. 0으로 설정하면 제한이 없습니다. |
| minIdle | 0 (off) | 풀에서 유지하는 유휴 상태 커넥션의 최소 수. 0으로 설정하면 최소 유휴 커넥션을 유지하지 않습니다. |
| maxReuseCount | 0 (off) | 커넥션을 재사용할 수 있는 최대 수. 설정한 값보다 많이 재사용된 경우 해당 커넥션은 종료되고 풀에 반납되지 않습니다. |
| timeBetweenEvictionRunsMillis | 0 (off) | 유휴 상태의 커넥션을 검사하는 eviction timer의 간격 (밀리세컨드). 해당 타이머에서 `minEvictableIdleTimeMillis`를 참조해서 사용된지 오래된 유휴 커넥션을 제거하고, `minIdle`를 유지하기 위한 신규 커넥션을 생성합니다. |
| minEvictableIdleTimeMillis | 1800000 | 유휴상태로 풀에 최대한 유지될 수 있는 시간 (밀리세컨드). 0으로 설정하면 제한이 없습니다. |
| numTestsPerEvictionRun | 3 | eviction timer 실행 시 검사 대상으로 할 커넥션의 최대 수 |

# # 사용법

- mysql-pool-booster 설치

{% highlight bash %}
npm install mysql-pool-booster
{% endhighlight %}

- mysql 객체 변환

{% highlight javascript %}
var mysql = require('mysql');

// 기존의 mysql 객체를 변환하고
var MysqlPoolBooster = require('mysql-pool-booster');
mysql = MysqlPoolBooster(mysql);

// 기존에 사용하던 그대로 사용하면 됩니다.
mysql.createPool({ ... });
{% endhighlight %}

- 기존에 사용하던 그대로 mysql 객체 사용

# # 사용의 제한
mysql 모듈에서 제공하는 모든 테스트를 통과하고 100% 호환성을 제공하기 때문에 일반적인 사용에는 아무런 문제가 없습니다. 단, 일반적이지 않은 패턴으로 사용하고 있었다면 정상적으로 동작하지 않을 수 있습니다. 예를 들어 아래와 같은 경우입니다.

{% highlight javascript %}
// 풀 객체의 private 변수에 직접 접근하는 것은 잘못된 사용입니다.
if (pool._allConnections.length > 0) {
  ...
}

// 이 경우에는 아래와 같은 방법을 사용하셔야 합니다.
if (pool.getStatus().all > 0) {
  ...
}
{% endhighlight %}

# # 문의
사용해보시고 문제가 발생할 경우 github에 issue를 남겨주시거나 [ifsnow@gmail.com](mailto:ifsnow@gmail.com)으로 보내주시면 최대한 빨리 확인해보겠습니다. 문제없이 동작할 수 있도록 만들기 위해 노력했지만 이제 막 시작된 프로젝트이기 때문에 검증하는 단계를 충분히 거친 후 실 서비스에 적용하시기 바랍니다.

[View on github](https://github.com/ifsnow/mysql-pool-booster)

누군가에겐 도움이 되길 바랍니다. 감사합니다.
