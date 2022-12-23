## Database study 2022 

---

### trigger

- 데이터베이스에서 어떤 이벤트(insert,update,delete)가 발생했을때 자동적으로 실행되는 프로시저를 말한다.

#### `trigger 생성`

> 1. users 테이블에서 update가 일어날때 insert 쿼리가 날라가도록 설정하는 trigger

```
mysql> delimiter $$
mysql> create trigger log_user_nickname_trigger //trigger 이름
    -> before update          // 동작 시점 정의
    -> on users for each row  // 업데이트 되는 로우마다 동작 
    -> begin                  // 동작 정의
    -> insert into users_log values(OLD.id,OLD.nickname,now());
    -> end                    // 동작 정의 끝
    -> $$
mysql> delimiter ;  

* OLD는 업데이트가 되기전 튜플을 가리킨다  .
* delete 시에는 delete된 해당 튜플을 가리킨다.
```


> 2.buy테이블 insert시에 buy_user_stats 테이블에 구매 총액을 저장하는 trigger

```
mysql> delimiter $$
mysql> create trigger update_userprice_sum
    -> after insert // insert 직후
    -> on buy for each row 
    -> begin
    ->    declare total int; // 전체값 변수
    ->    declare user_id int default `new`.user_id; // user_id 저장
    ->    select sum(price) into total from buy where user_id = user_id; // 총액 계산
    ->    update user_buy_stats set price_sum = total where user_id = user_id; // 총액 변경
    -> end
    -> $$
Query OK, 0 rows affected (0.01 sec)

mysql> delimiter ;

* new는 insert된 tuple을 가리킨다.
* update 시에는 update된 후의 tuple을 가리킨다.


> 사용시 주의점
- 데이터 티어에서만 동작하기 때문에 로직티어의 소스코드에서는 해당 코드에 대해 파악하기 힘들다.
- 개발 , 관리, 문제 파악이 힘들어질 수 있다.
- 트리거가 너무 지나치게 사용하면 DB에 부담을 주고 응답이 느려질 수 있다.
- 디버깅이 어렵다.
- 문서 정리가 특히 중요하다.
```

---

## transaction

---

- 단일한 논리적 작업 단위를 말한다 (a single logical unit of work)
- 논리적인 이유로 여러 SQL 문들을 단일 작업으로 묶어서 나눠질 수 없게 만드는 것이 transaction 이다.
- transaction의 SQL문들 중에 일부만 성공해서 DB에 반영되는 일은 일어나지 않는다.
- spring의 경우 @transactional이 붙은 메서드는 transaction을 통해 작업이 이뤄진다.

> 일반적인 트랜잭션의 사용 패턴
1. transaction을 시작(begin)한다   (start transaction)
2. 데이터를 읽거나 쓰는 등의 SQL문들을 포함해서 로직을  수행한다.
3. 일련의 과정들이 문제없이 동작했다면 transaction을 commit 한다 (영구적 저장)
4. 중간에 문제가 발생한다면 transaction을 rollback 한다 (원상 복구)


> ACID(Atomicity Consistency Isolation Durability)

- 트랜잭션이 지녀야하는 4가지 속성을 말한다. 

1. Atomicity(원자성)
- All or Nothing
- transaction은 논리적으로 쪼개질 수 없는 작업 단위이기 때문에 내무의 SQL문들이 모두 성공해야 한다.
- 중간에 SQL문이 실패하면 지금까지의 작업을 모두 취소하여 아무 일도 없었던 것처럼 rollback 한다.

2.Consistency(일관성)
- transaction은 DB 상태를 consistent 상태에서 또 다른 consistent 상태로 바꿔주는 것을 의미한다.
- constraints , trigger 등을 통해 DB에 정의된 rules을 transaction이 위반했다면 rollback해야한다.
- transaction 이 DB에 정의된 rule을 위반했는지는 DBMS가 commit 전에 확인하고 알려준다.
- 그 외의 application 관점에서 transaction이 consistent 하게 동작하는지는 개발자가 챙겨야한다.

3. Isolation(격리)
- 여러 transaction 들이 동시에 실행될 때도 혼자 실행되는 것처럼 동작하게 만들어야 한다.
- DBMS는 여러 종류의 isolation level을 제공한다.
- 개발자는 Isolation level 중 어떤 level로 transaction을 동작시킬지 설정할 수 있다.
- concurrency control의 주된 목표가 isolation 이다.

4.Durability(영존성)
- DB system에 문제가 생겨도 commit된 transaction은 db에 영구적으로 저장된다.
- 영구적으로 저장한다 라고 할때 일반적으로 비휘발성 메모리(hdd , sdd)에 저장함을 의미한다.
- 기본적으로 transaction의 durability는 DBMS가 보장한다.


---

### Concurrency control

> schedule

- 여러 transaction들이 동시에 실행될 때 각 transaction에 속한 operation들의 실행순서를 말한다.
- 각 transaction내의 opertaion들의 순서는 바뀌지 않는다.
- transaction들이 겹치지 않고 순차적으로 실행되는 스케쥴을 serial schedule이라 한다.
- transaction들이 겹쳐서 순차적이지 않게 실행되는 스케쥴을 non-serial schedule이라 한다.
- non-serial schedule은 non-block으로 동작하기 때문에 동시성이 높아 같은 시간 동안 더많은 transaction들을 처리할 수 있다.
- 하지만 non-serial schedule을 사용할 경우 데이터의 불일치가 일어날 수 있기 떄문에 Concurrency control(동시성 제어)가 필요하다

> conflict

- 두개의 operation이 아래의 세가지 조건을 모두 만족하면 conflict 하다 할 수 있다.
- 하나는 read 하나는 write로 이뤄진 conflict를 read-write conflict라 한다.
- conflict operation은 순서가 바뀌면 결과도 바뀐다.
1. 서로 다른 transaction 소속이다
2. 같은 데이터에 접근한다.
3. 최소 하나는 write operation이여야 한다.

