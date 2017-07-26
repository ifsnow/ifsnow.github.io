---
layout: default
title:  "Performance booster for the pool of mysql (node.js)"
og_title : "mysql-pool-booster"
og_description: "Performance booster for the pool of mysql (node.js)"
og_image: "ifsnow.png"
og_image_width: 151
og_image_height: 151
date:   2017-07-26 00:00:00
categories: nodejs
---

# # mysql-pool-booster
Node.js + mysql 모듈 환경에서 커넥션 풀을 사용하는 분들을 위한 모듈입니다. 단 몇 줄만 추가하면 기본으로 제공되는 것보다 더 좋은 성능과 유용한 옵션의 커넥션 풀을 사용할 수 있게 됩니다.

# # 만들게 된 이유
심심할 때마다 mysql 모듈 프로젝트에 기여하곤 했는데 오랜 역사와 많은 사용자를 가진 프로젝트이기 때문인지 조금 큰 변화의 요구([pull request #1779](https://github.com/mysqljs/mysql/pull/1779))엔 조심스럽다는 느낌을 받았습니다. 오픈 소스 프로젝트를 유지하는 분들의 선의와 고충을 이해하고 저라도 그렇게 할 것 같습니다.

그래서, 기존 모듈에 영향을 주지 않으면서 제안했던 기능을 사용할 수 있는 신규 모듈을 만들게 됐습니다. 어쩌면 저 말고도 필요한 분이 한두 분은 계실지 모르겠단 마음으로요.

# # 제공되는 기능

## 1. 성능 향상

추가 작업 없이 적용만으로 여러분의 서비스가 80% 이상 효율적으로 변할 수 있습니다. (어디서 약을 팔..)

3만 개의 요청을 처리하는 [성능 측정 코드](https://gist.github.com/ifsnow/5cc2a628574c2708eb91231c1abe92cd)의 결과입니다.

|  | 오리지널 | 부스터 버전  |
| --- | --- | --- |
| 초당 처리 | 5,877  | 10,650 (181% ↑) |

원격의 mysql 서버를 사용하면서 동시 요청이 많은 서비스에서 개선된 효과를 더 체감하실 수 있습니다.

## 2. 유용한 옵션

mysql 모듈의 커넥션 풀에서 기본적으로 제공하는 옵션은 거의 없습니다. 최대 생성 가능한 커넥션의 수인 `connectionLimit`과 가용 커넥션이 없는 경우 대기하는 큐의 최대 크기인 `queueLimit` 정도입니다. 이 옵션으로는 순간적으로 사용자가 몰려서 생성된 커넥션을 불필요한 순간에 제거하지 못해 계속 유지시켜야 했습니다. `mysql-pool-booster`를 적용하면 자바의 DBCP와 같은 여러 가지 옵션으로 유연하게 운영할 수 있습니다.

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

- 기존 mysql 객체 변환

{% highlight javascript %}
var mysql = require('mysql');

// 기존의 mysql 객체를 변환하고
var MysqlPoolBooster = require('mysql-pool-booster');
mysql = MysqlPoolBooster(mysql);

// 기존에 사용하던 그대로 사용하면 됩니다.
mysql.createPool({ ... });
{% endhighlight %}

- 기존에 사용하던 그대로 mysql 객체 사용

# # 소스
[View on github](https://github.com/ifsnow/mysql-pool-booster)
