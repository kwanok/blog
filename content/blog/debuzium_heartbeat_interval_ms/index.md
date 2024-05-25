+++
title="Debezium을 운영할 때 알면 좋은 팁 - heartbeat.interval.ms"
date=2023-10-25
updated=2023-10-25
description="Debezium을 운영할 때 알면 좋은 팁 중 하나인 heartbeat.interval.ms에 대해 알아봅니다."

[taxonomies]
tags=["debezium", "database", "cdc", "postgresql"]

[extra]
giscus = true
quick_navigation_buttons = true
toc=true
+++

Debezium은 데이터베이스에서 연산을 완료한 뒤에 작성하는 로그(Write-Ahead Log|WAL)를 읽는 방식으로 이벤트를 감지하기 때문에 트랜잭션 오류나 커넥션이 끊긴다는 등의 과정에서 문제에서는 안전하다고 볼 수 있습니다.

다만, 이 WAL은 Debezium 이외의 레플리케이션 등 다른 작업에서도 사용하기 때문에 읽기 연산이 많아질 경우 데이터베이스 인스턴스에도 영향을 줄 수 있습니다.


- FAQ에서도 MySQL의 예로 binlog 파일을 읽어서 동작하고, 그에 따른 권한을 할당해야한다고 나와 있고, 주의 사항도 있습니다
![](https://velog.velcdn.com/images/noh0907/post/7a9a757f-9a93-46c4-b859-7bb91e4de8ed/image.png)



> Postgres의 경우, [WAL Disk Space Consumption](https://debezium.io/documentation/reference/1.0/connectors/postgresql.html#wal-disk-space) 을 참고하면 운영하는 데 도움이 될겁니다.


### LSN(Log Sequence Number) 모니터링하기


LSN은 `pg_replication_slots` 테이블의 `confirmed_flush_lsn` 컬럼과 `restart_lsn` 의 비교를 통해서 확인할 수 있습니다.

CDC가 정상적으로 처리되고 있다면  `confirmed_flush_lsn` > `restart_lsn` 이 성립돼야 합니다.

```sql
select restart_lsn, confirmed_flush_lsn from pg_replication_slots;  
-- 위 명령어로 lsn 정보를 확인할 수 있습니다
```


**컬럼 설명**

`restart_lsn` : Postgres에서 관리되는 값으로 debezium 입장에서는 **“너 적어도 여기까지는 반영해야 해”** 라는 값입니다. 만약 debezium이 `restart_lsn` 까지 읽지 않은 경우 Postgres는 debezium이 아직 못 읽은 WAL로그를 못버리고 들고 있어야 하기 때문에 부담이 되죠

`confirmed_flush_lsn` :  debezium이 마지막으로 확인한 WAL 이벤트 위치입니다. `restart_lsn` 값보다 크다면 Postgres가 내준 숙제는 제대로 하고 있다는 의미니까 제대로 데이터 변경을 캡처하고 있는 겁니다.

그래서 LSN에 문제가 있다면, debezium의 퍼포먼스가 낮다는 걸 의미하고 성능을 점검할 필요가 있다는 신호입니다.

### DB에서 일어나는 모든 업데이트 >>>>>>>> 캡쳐 중인 테이블의 업데이트


데이터베이스에는 많은 업데이트가 발생하지만, 실제로 debezium에 의해 모니터링되는 테이블에 대한 변경이 아주 적은 경우엔 WAL이 빠르게 채워질 수 있습니다.

debezium에 의해 모니터링되는 테이블에 대한 변경이 잦은 경우에는 일정 주기마다 WAL을 flush해주기 때문에 WAL용량을 작게 유지할 수 있지만, 위에서 배웠듯이 debezium은 기본적으로 업데이트가 일어날 때마다 `confirmed_flush_lsn`  가 업데이트되기 때문에 빠르게 커지고 있는 WAL이 flush가 안 돼서 디스크를 많이 차지하게 되는 거죠.

이걸 해결하기 위한 옵션이 바로 `heartbeat.interval.ms` 입니다.

**“나 여기 있어, 작동 중이야”** 라고 데이터베이스에 일정 주기로 알리면서 모니터링 중인 테이블에 변경이 없어도 WAL을 flush 할 수 있게 하는 옵션입니다.


## Reference

- [https://debezium.io/documentation/faq/](https://debezium.io/documentation/faq/)
- [https://stackoverflow.com/questions/70974802/what-is-the-difference-between-restart-lsn-and-confirmed-flush-lsn-in-postgresql](https://stackoverflow.com/questions/70974802/what-is-the-difference-between-restart-lsn-and-confirmed-flush-lsn-in-postgresql)
- [https://rapportlabs.oopy.io/article/2](https://rapportlabs.oopy.io/article/2)