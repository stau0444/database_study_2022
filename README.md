## Database study 2022 

---

### `trigger`

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
- 단일한 논리적 작업 단위를 말한다 (a single logical unit of work)
- 논리적인 이유로 여러 SQL 문들을 단일 작업으로 묶어서 나눠질 수 없게 만드는 것이 transaction 이다.
- transaction의 SQL문들 중에 일부만 성공해서 DB에 반영되는 일은 일어나지 않는다.
- spring의 경우 @transactional이 붙은 메서드는 transaction을 통해 작업이 이뤄진다.
---

#### `일반적인 트랜잭션의 사용 패턴`

1. transaction을 시작(begin)한다   (start transaction)
2. 데이터를 읽거나 쓰는 등의 SQL문들을 포함해서 로직을  수행한다.
3. 일련의 과정들이 문제없이 동작했다면 transaction을 commit 한다 (영구적 저장)
4. 중간에 문제가 발생한다면 transaction을 rollback 한다 (원상 복구)


#### `ACID(Atomicity Consistency Isolation Durability)`

- 트랜잭션이 지녀야하는 4가지 속성을 말한다. 

> 1. Atomicity(원자성)
- All or Nothing
- transaction은 논리적으로 쪼개질 수 없는 작업 단위이기 때문에 내무의 SQL문들이 모두 성공해야 한다.
- 중간에 SQL문이 실패하면 지금까지의 작업을 모두 취소하여 아무 일도 없었던 것처럼 rollback 한다.

> 2.Consistency(일관성)
- transaction은 DB 상태를 consistent 상태에서 또 다른 consistent 상태로 바꿔주는 것을 의미한다.
- constraints , trigger 등을 통해 DB에 정의된 rules을 transaction이 위반했다면 rollback해야한다.
- transaction 이 DB에 정의된 rule을 위반했는지는 DBMS가 commit 전에 확인하고 알려준다.
- 그 외의 application 관점에서 transaction이 consistent 하게 동작하는지는 개발자가 챙겨야한다.

> 3. Isolation(격리)
- 여러 transaction 들이 동시에 실행될 때도 혼자 실행되는 것처럼 동작하게 만들어야 한다.
- DBMS는 여러 종류의 isolation level을 제공한다.
- 개발자는 Isolation level 중 어떤 level로 transaction을 동작시킬지 설정할 수 있다.
- concurrency control의 주된 목표가 isolation 이다.

> 4.Durability(영존성)
- DB system에 문제가 생겨도 commit된 transaction은 db에 영구적으로 저장된다.
- 영구적으로 저장한다 라고 할때 일반적으로 비휘발성 메모리(hdd , sdd)에 저장함을 의미한다.
- 기본적으로 transaction의 durability는 DBMS가 보장한다.


---

### Concurrency control
- 어떤 schdule도 serializable하고 recoverity하게 만드는 것을 의미한다.
- transaction의 속성인 ACID 중 Isolation과 밀접한 관련이 있다. 
---

#### ` schedule`

- 여러 transaction들이 동시에 실행될 때 각 transaction에 속한 operation들의 실행순서를 말한다.
- 각 transaction내의 opertaion들의 순서는 바뀌지 않는다.
- transaction들이 겹치지 않고 순차적으로 실행되는 스케쥴을 serial schedule이라 한다.
- transaction들이 겹쳐서 순차적이지 않게 실행되는 스케쥴을 non-serial schedule이라 한다.
- non-serial schedule은 non-block으로 동작하기 때문에 동시성이 높아 같은 시간 동안 더많은 transaction들을 처리할 수 있다.
- 하지만 non-serial schedule을 사용할 경우 데이터의 불일치가 일어나는 현상이 일어날 수 있다.
- 개발자는 Isolation level을 통해 데이터의 정합성과 , 처리 속도 사이에서 trade 할 수 있다.


#### `conflict`

- 두개의 operation이 아래의 세가지 조건을 모두 만족하면 conflict라 할 수 있다.
- 하나는 read 하나는 write로 이뤄진 conflict를 read-write conflict라 한다.
- conflict operation은 순서가 바뀌면 결과도 바뀐다.

1. 서로 다른 transaction 소속이다
2. 같은 데이터에 접근한다.
3. 최소 하나는 write operation이여야 한다.

#### `conflict equipvalent`

- serial-schedule만 사용하기엔 성능에 문제가 있기 때문에 non-serial-schedule을 같이 사용할 수 있는 방법이 필요했다
- 하지만 non-serial-schedule은 데이터 정합성에 문제가 생길 수 있기 때문에 이를 해결하기 위해 
- conflict serializable한 non-serial-schedule만 실행을 허용하도록하는 방식을 사용하여 충돌을 막으며
- 이러한 두 스캐쥴을 conflict equipvalent 하다 한다.

#### `conflict equipvalent의 조건`

<img width="498" alt="스크린샷 2022-12-25 오전 1 10 20" src="https://user-images.githubusercontent.com/51349774/209443663-e3bcdab5-2f43-4c8e-880a-d0a8ed7f2320.png">


- 두 schedule은 같은 transaction들을 가진다.
- 어떤 conflicting operations의 순서도 양쪽 schedule 모두 동일하다.
- non-serial schedule이 serial schedule과  conflict equivalent할때 이를 conflict serializable 하다고 한다

#### `RDBS에서 conflict serializable을 구현하는 방식`

- 여러 transaction을 동시에 실행해도 schedule이 conflict serializable하도록 보장하는 프로토콜을 적용하는 방식으로 구현된다.


#### `Recoverivility`

> unrecoverable schedule
- schedule 내에서 commit된 transaction이 rollback된 transaction이 write한 데이터를 읽은 경우를 말한다.
- rollback을 해도 이전 상태로 회복이 불가능할 수 있기 때문에 DBMS에서는 이런 schedule을 허용해선 안된다.


<img width="320" alt="스크린샷 2022-12-25 오전 3 18 56" src="https://user-images.githubusercontent.com/51349774/209447347-93500416-e4e5-40b4-9044-358bc8d90e78.png">


> recoverable schedule
- schedule안에서 어떤 transaction도 자신이 읽은 데이터를 write한 transaction이 먼저 commit/aboart 하기 전까지는
- commit을 하지 않아야 하며 이런 스캐쥴을 recoverable schedule이라 하며 Recoverivility를 갖었다고 한다.

> casacading rollback
- 하나의 transaction이 rollback 하면 의존성이 있는 다른 transaction도 rollback하는 rollback 방식을 말한다.
- 여러 transaction의 rollback이 연쇄적으로 일어나면 처라하는 비용이 많이 든다.

> cascadless schedule
- schedule 내에서 어떤 transaction도 commit 되지 않은 transaction들이 write 한 데이터는 읽지 않는 스캐쥴을 말한다.
- avoid cascade rollback 이라고도 부른다.

> strict schedule 
- schedule 내에서 어떤 transaction도 commit 되지 않은 transaction들이 write 한 데이터는 읽지도 쓰지도 않는 스캐쥴을 말한다.

 
#### `Isolation level`
- 애플리케이션 설계자는 Isolation level을 통해 전체 처리량과 데이터 일관성 사이에 어느 정도 거래(trade)를 할 수 있다
- dirty read,non-repeatable read,phantom read 세가지 중 어떤 것을 허용하는지에 따라 level이 구분된다.

> 여러개의 transaction 동시에 실행될 때 발생할 수 있는 이상 현상

- dirty read :한 트랜잭션이 write한 커밋되지 않은 데이터를 읽어 다른 트랜잭션의 값이 오염되는 경우를 말한다.
- non-repeatable read (fuzzy read): 같은 데이터를 한 트랜잭션에서 두번 읽었을때 다른 트랜잭션의 영향을 받아 두개의 값이 다른 경우를 말한다.(isolation 관점에선 여러 트랜잭션이 동시에 실행되도 각각의 트랜잭션은 독립적으로 실행되어야한다.)
- phantom read : 같은 데이터를 한 트랜잭션에서 두번 읽었을때 다른 트랜잭션의 영향을 받아 없던 데이터가이 생기는 경우를 말한다.

> Isolation level 분류
- 위의 이상 현상은 모두 발생하지 않도록 만들 수 있지만 제약사항이 많아져 동시성이 낮아지고 처리 가능한 트랜잭션 수가 줄어 db전체 처리량(throughput)이 하락하게된다
- 때문에 일부 이상현상은 허용하는 몇가지 level을 만들어서 사용자가 필요에 따라 적절하게 선택할 수 있도록 구분한 것이 Isolation level이다.



|Isolation level|Dirty read|Non-repeatable read|phantom read|
|----|----|----|----|
|Read uncommitted|O|O|O|
|Read committed|X|O|O|
|Repeatable read|X|X|O|
|Serializable|X|X|X|

#### `snapshot isolation (다시 정리 필요)`


#### `data lock`

- 운영체제에서의 데이터 락과 같은 개념이다 
- 같은 데이터에 대한 

> `read-lock 과 write-lock`

- read-lock은 read 작업에 대해선 여러 트랜잭션의 접근을 허용하지만  write작업은 허용하지 않는다(읽은 데이터를 보호하기 위함).
- write-lock은 read/write 작업 모두에 대해 다른 트랜잭션의 접근을 막는다.

> `data lock을 사용한 동시성 제어`
- lock 만으로 serializability를 보장할 수는 없다.
- 프로토콜을 통해 데이터 락을 구현할 수 있다.

> `2PL protocol (two phase locking)`

- 데이터 락으로 인해 발생하는 데이터 이상 현상을 막기위한 프로토콜이다.
- 초창기 DB에 주로 사용되던 방식이다 (오늘날 DB는 MVCC를 주로 사용)
- lock을 취득하는 phase(expanding phase(growing phase))와  lock을 반환하는 phase(shrinking phase (contracting phase))를 구분한다
- 트랜잭션에서 모든 locking operation이 최초의 unlock operation보다 먼저 수행되도록 한다 
- 한번 unlock이 실행되면 그 후론 lock을 취득하려하지 않아야한다.


|2PL protocol의 종류 |    |
|----|----|
|conservative 2PL| * 모든 lock을 취득한 뒤 트랜잭션을 시작한다 <br/> *deadlock-free하다 <br/> *실용적이지 못하다(락을 한번에 모두 취득하기 어려운 상황이 발생할 수 있다.)|
|stript 2PL|* strict schedule을 보장하는 2PL이다. <br/> * recoverablility를 보장하며 write-lock을 commit/rollback시에 반환하도록한다.
|strong stript 2PL|* strict schedule을 보장하는 2PL이다. <br/> * recoverablility를 보장하며 write-lock/read-lock을 commit/rollback시에 반환하도록한다.|

MVCC

---

### `데이터 정규화` (보충 필요)

---

#### `DB schema 설계가 잘못되면 생기는일들`

> 1. 중복 데이터 문제
- 하나의 테이블에 여러 관심사의 정보를 하나의 테이블에 모아놓을 경우 
- 중복된 데이터가 발생할 수 있으며 저장 공간 낭비 , 데이터 불일치 가능성이 존재하게된다.
- insert(insertion anomalies),delete(deletion anomalies),update(update anomalies)시에 null 값을 많이 사용하게되며 로우가 지저분해진다.

> 2.spurious tuples (가짜 튜플)
- natural join시 가짜 튜플들이 생겨 날 수 있다.

> 3. null 값이 많아짐으로 인한 문제점들
- null 값이 있는 column으로 join 하는 경우 상황에 따라 예상과 다른 결과가 발생한다.
- null 값이 있는 colmn에 aggregate function을 사용했을 때 주의가 필요하다.
- 불필요한 storage 낭비가 일어난다

#### `바른 DB schema 설계 규칙`

1. 의미적으로 관련있는 속성들끼리 테이블을 구성한다
2. 중복 데이터를 최대한 허용하지 않도록 설계
3. join 수행시 가짜 데이터가 생기지 않도록 설계
4. 되도록이면 null 값을 줄일 수 있는 방향으로 설계


#### `functional dependency`
- 한 테이블에 있는 두 개의 attribute 집합 사이의 제약


---
### `INDEX`
---
- 조건에 만족하는 튜플들을 빠르게 조회하기 위해 사용되는 자료구조이다.
- 상황에 따라서는 빠르게 정렬(order by) 혹은 그룸핑(group by)하기 위해서도 사용된다.
- 쿼리문에 where 절의 컬럼에 인덱스가 걸려 있지 않다면 , full scan(table scan)으로 데이터가 찾아진다
- 이떄의 시간복잡도는 데이터 개수(N)만큼 O(N)[big O n]이 된다.
- 만약 해당 컬럼에 인덱스가 걸려있고 b-tree 기반이라면 시간복잡도는 O(logN)으로 줄어든다
- 대부분의 RDBMS에서는 primary key에는 index가 자동으로 달린다.

``` java

player table

+------------+--------------+------+-----+---------+-------+
| Field      | Type         | Null | Key | Default | Extra |
+------------+--------------+------+-----+---------+-------+
| id         | int          | NO   | PRI | NULL    |       |
| name       | varchar(255) | YES  |     | NULL    |       |
| team_id    | int          | YES  |     | NULL    |       |
| backnumber | int          | YES  |     | NULL    |       |
+------------+--------------+------+-----+---------+-------+

//index 생성 (이미 만들어진 테이블에 인덱스 걸어주기)

// 중복이 허용되는 컬럼에 인덱스를 걸어줄 때
mysql> create index player_name_idx on player(name);

// 튜플을 유니크하게 식별할수 있는 키에 인덱스를 걸어줄 떄
// 아래같이 여러 컬럼이 한번에 묶여 인덱스로 사용되는 것을 multicolumn index 혹은 composite index라 한다.
// 
mysql> create unique index team_id_backnum_idx on player(team_id,backnumber);


//테이블 생성하며 인덱스 함께 생성하는 방식

```java

CREATE TABLE player( 
        id int primary key, 
        name varchar(20) not null, 
        team_id int, 
        backnumber int,
        INDEX player_name_idx (name), // 이 방식에서는 인덱스 이름이 생략가능하다.
        UNIQUE INDEX team_id_backnumber_idx (team_id,backnumber) 
    );

```

//인덱스 사용 명령문 
SHOW INDEX FROM player;

```

### `b-tree 기반 index 동작 방식`

<img width="536" alt="스크린샷 2022-12-28 오전 1 05 22" src="https://user-images.githubusercontent.com/51349774/209691603-08788bb4-2cc7-4fc2-bb3e-a27c1255c458.png">


- 위 그림처럼 a에 대한를 걸면 인덱스 테이블이 생긴다.
- 인덱스 테이블에는 인덱스가 걸린 테이블 기준으로 데이터가 정렬되고 pointer라는 컬럼에는 오른쪽의 실제 튜플을 가리키는 값이 저장된다.
- 만약 where a = 4; 라는 쿼리가 날라오면 db에서는 인덱스 테이블에서 binary search(이진 탐색)으로 데이터를 찾아낸다.
- 만약 where a= 2 and b = 20; 이라는 쿼리가 날라온다면 a에 대하여 인덱스 테이블을 통해 찾고 찾아진 튜플들에 대해 실제 테이블로 찾아가 b 값을 비교하여 찾아낸다.
- where a= 2 and b = 20; 같은 쿼리에서는 인덱스가 걸린 a의 경우 빠르게 찾아내지만 b를 찾아내는 과정에서 결국 full scan이 일어난다.
- composite index를 사용하여 a , b를 묶어서 인덱스로 관리해야 한다.
- composite index의 경우 생성할때 설정한 컬럼중 왼쪽 컬럼을 기준으로 먼저 정렬이되고 중복되는 값들에 대해서 오른쪽 컬럼을 기준으로 다시한번 정렬된다.
- 한 컬럼에 인덱스가 여러개 걸려있다면 DB의 optimizer가 최적의 index를 찾아서 사용한다.
- 간혹 optimizer가 이상한 인덱스를 고르는 경우 직접 사용될 Index 명시하고 싶다면 쿼리 문에 USE INDEX(인덱스명) 혹은 FOR INDEX(인덱스명)을 사용할 수 있다.
- FORCE INDEX는 더 강제적이며 무조건 해당 인덱스가 사용되는 것은 아니고 Optimaizer의 판단하에 아니다 싶으면 fullscan으로 동작한다.
- 특정 인덱스를 제외시키고 싶다면 IGNORE INDEX(인덱스명)을 사용할 수 있다.
- ex)EXPLANE SELECT * FROM player USE INDEX(backnumber_idx) WHERE backnumber = 7;
- ex)EXPLANE SELECT * FROM player FORCE INDEX(backnumber_idx) WHERE backnumber = 7;
- ex)EXPLANE SELECT * FROM player IGNORE INDEX(backnumber_idx) WHERE backnumber = 7;
- 실제 어떤 인덱스가 사용되는지는 EXPLANE + 쿼리문으로 확인할 수 있다.
- ex)EXPLANE SELECT * FROM player WHERE backnumber = 7;
그림고

#### `covering index`
- 조회하는 attribute를 index가 모두 cover 하는 인덱스를 말한다.
- 따로 본테이블에 가서 데이터를 조회하지 않아도 되기 때문에 조회 성능이 더 빠르다.

#### `hash index`

- hash table을 사용하여 index를 구현한다
- 시간복잡도 O(1)의 성능
- rehashing(데이터가 모두차서 hash 테이블의 size를 늘려줄때)
- equality( = , !=) 비교만 가능하다, range 비교(<= , => ,< ,>)는 불가능함
- multicolumn index의 경우 전체 attributes에 대한 조회만 가능하다 (b-tree 에서는 일부분에 해당하는 조건도 인덱스 사용이 가능)

#### `full scan이 좋은 경우`
- 데이터가 적을때
- 조회하려는 데이터가 테이블의 상당 부분을 차지할 때


#### `index 사용 주의점`

- 인덱스가 생기는 것은 해당 테이블에 연관된 또다른 테이블이 생기는 것이고 원래 테이블에 write(update , delete, insert) 작업이 일어난다면
- 연관된 index 테이블에도 write 작업이 모두 일어나고 정렬도 새로되야 되기 때문에 오버헤드가 발생한다.
- order by나 group by 에도 index 가 사용될 수 있다.
- foreign key에는 index가 자동으로 생성되지 않을 수 있다. (mysql에서는 자동적으로 생성됨)
- 이미 데이터가 몇 백만건 이상있는 테이블에 인덱스를 생성하는 경우 시간이 몇 분이상 소요될 수도 있고 DB성능에 악영향을 미치기 때문에
  잘 고려해서 추가해줘야한다.
- 
 
- 인덱스가 생기는 것은 해당 테이블에 연관된 또다른 테이블이 생기는 것이고 원래 테이블에 write 작업이 일어난다면상
- 인덱스가 생기는 것은 해당 테이블에 연관된 또다른 테이블이 생기는 것이고 원래 테이블에 write 작업이 일어난다면
