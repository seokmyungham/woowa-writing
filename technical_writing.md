# MySQL, JPA Bulk Query & Batch Processing 성능 개선기

## Intro

안녕하세요. 우아한테크코스 6기 및 팀 모모에서 백엔드 개발을 하고 있는 재즈입니다.  
  
모모는 여러 사람들과의 일정을 한눈에 파악하고 만날 수 있는 시간을 취합하는 서비스인데요, JPA의 `saveAll()`, `deleteAll()`을 사용하면서 발생했던 성능 문제를 해결하며 얻은 경험과 지식을 공유해드리려고 합니다.
  
관련된 PR 본문은 [github](https://github.com/woowacourse-teams/2024-momo/pull/273) 에서 확인할 수 있습니다.

### 성능 저하 문제 인식

저희 팀은 데이터 저장소로 MySQL을 사용하면서, JPA의 `saveAll()`, `deleteAll()` 메서드를 사용해 시간 데이터를 일괄 처리하는 기능을 구현했었습니다. 하지만 대량의 데이터를 처리할 때, 의도하지 않은 개별 쿼리가 실행되면서 네트워크 통신 비용이 증가하였고, 쿼리 속도가 크게 저하 되었는데요. 이로 인해 서비스 속도가 현저히 느려지는 문제를 경험하게 되었습니다.

쿼리가 개별적으로 실행되면 왜 성능이 저하될까요? 가장 큰 이유는 네트워크와 데이터베이스간의 빈번한 I/O 작업 때문입니다. 각 쿼리마다 별도의 네트워크 요청이 발생하면서, 네트워크 왕복 시간이 누적되고 데이터베이스의 트랜잭션 관리 비용을 증가하게 됩니다. 또한 이러한 반복적인 요청은 데이터베이스의 리소스 사용을 급격히 증가시켜 병목 현상을 일으키며, 전체적인 처리 속도를 저하시킵니다.

서비스 성능을 최적화하기 위한 가장 일반적인 방법은 불필요한 I/O를 줄이는 것입니다. 불필요한 I/O를 줄인다는 것은 네트워크와 데이터베이스 간의 통신을 최소화하는 것을 말합니다. 개별적으로 실행되는 다수의 쿼리를 한 번에 처리할 수 있다면 누적되는 네트워크 병목 현상을 줄이고 요청 처리 속도는 물론 시스템의 전반적인 효율성을 향상시킬 수 있게 됩니다.

## JPA를 활용한 Batch 처리

배치 작업은 대량의 데이터를 한 번에 처리하는 작업을 의미합니다. 여러 개의 요청을 각각 처리하는 대신 하나의 트랜잭션로 묶어 한 번에 처리함으로써 데이터베이스와의 통신을 최소화하는 것이 가능합니다. 자바와 스프링을 사용하는 경우 배치 프로세스를 구축하기 위해 보통 JDBC나 Spring Batch 프레임워크를 많이 사용하는데요. JPA, Hibernate를 사용하는 경우에도 배치 기능을 사용할 수 있습니다. 

적용하는 방법은 생각보다 매우 간단합니다. application.yml 파일에 `hibernate.jdbc.batch_size`를 설정하여 배치 크기를 지정하면 Hibernate는 지정된 배치 크기만큼 INSERT, UPDATE, DELETE 쿼리에 대해 배치 작업을 수행합니다. 

```yml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50
```

Hibernate는 설정한 배치 크기에 도달할 때 까지 [PreparedStatement.addBatch()](https://docs.oracle.com/en/java/javase/11/docs/api/java.sql/java/sql/Statement.html#addBatch(java.lang.String))를 호출하여 각각의 쿼리를 실행하지 않고, 여러 개의 SQL 쿼리들을 배치로 모아두는 작업을 수행합니다. 이후 설정한 배치 개수에 도달하거나 모든 엔티티에 대해 모아두는 작업을 마치게 되면[PreparedStatement.executeBatch()](https://docs.oracle.com/javase/8/docs/api/java/sql/Statement.html#executeBatch--)를 호출합니다. `executeBatch()` 메서드는 `addBatch()`로 모아둔 SQL 쿼리들을 한꺼번에 묶어 데이터베이스로 한번에 전송하게 됩니다.

그리고 JPA는 데이터를 저장하기 위한 `save()`, `saveAll()` 메서드를 제공합니다. 이 두 메서드가 내부적으로 동작하는 원리는 같습니다. 

```java
@Transactional
public <S extends T> S save(S entity) {
    Assert.notNull(entity, "Entity must not be null");
    if (this.entityInformation.isNew(entity)) {
        this.entityManager.persist(entity);
        return entity;
    } else {
        return this.entityManager.merge(entity);
    }
}
```
```java
@Transactional
public <S extends T> List<S> saveAll(Iterable<S> entities) {
    Assert.notNull(entities, "Entities must not be null");
    List<S> result = new ArrayList();
    Iterator var4 = entities.iterator();

    while(var4.hasNext()) {
        S entity = (Object)var4.next();
        result.add(this.save(entity));
    }

    return result;
}
```

`saveAll()` 메서드 내부 구현을 살펴보면 저장할 데이터에 대해 반복문을 돌면서 `this.save()`를 호출하는 것을 알 수 있습니다. 하나의 트랜잭션 안에서 단지 `save()`를 여러 번 호출하는 것과 다를 게 없는데요. 스프링의 트랜잭션 전파 기본 속성은 `REQUIRE`로 기존 트랜잭션이 없으면 트랜잭션을 생성하고, 기존 트랜잭션이 있으면 기존 트랜잭션에 참여하는 방식입니다. 트랜잭션 안에서 `save()` 여러 번 호출해도 새로운 트랜잭션을 생성하는 비용이 없으니 100번의 `save()`를 반복적으로 호출하던, 100개의 entities에 대해 `saveAll()` 1번 호출하던 결과는 동일하다는 것을 알 수 있습니다.

## rewriteBatchedStatements
지금까지 `hibernate.jdbc.batch_size` 옵션을 통해 Hibernate가 제공하는 배치 기능과 데이터를 저장하는 방식에 대해 알아봤습니다. 배치란 결국 개별 쿼리들을 원하는 개수만큼 하나의 트랜잭션에서 처리하는 작업을 말합니다. 그런데 여기서 헷갈리지 말아야 하는 부분은 배치란 네트워크 왕복을 줄여 성능을 개선하는 것이지 한 번의 쿼리를 실행한다는 의미가 아니라는 점입니다. 

Hibernate 배치 기능을 활성화하고 직접 SQL을 실행해보면 알 수 있는데요. 실제로 INSERT 쿼리가 단건식 출력되는 모습을 확인할 수 있습니다.

```sql
org.hibernate.SQL                        : 
    insert 
    into
        member
        (name, age, id) 
    values
        (?, ?, ?)
org.hibernate.SQL                        : 
    insert 
    into
        member
        (name, age, id) 
    values
        (?, ?, ?)
org.hibernate.SQL                        : 
    insert 
    into
        member
        (name, age, id) 
    values
        (?, ?, ?)
```

네트워크 왕복 횟수를 줄여 오버헤드를 감소시키기는 했지만, 데이터베이스에서는 여전히 각 SQL 쿼리를 개별적으로 실행해야 하는 상황입니다. MySQL 기준으로 데이터베이스에 데이터를 삽입하려면 클라이언트 스레드 할당 –> 쿼리 파서 및 전처리 -> 옵티마이저의 실행 계획 수립 -> 락 획득 -> 데이터 저장 -> 락, 스레드 반납과 같은 과정을 거쳐야 합니다. 아무리 단시간에 처리되는 간단한 쿼리라도 쿼리마다 매번 이러한 과정이 발생하면 성능 상 손해를 볼 수 밖에 없습니다. 

예를 들어, 한 쿼리에 0.01초가 걸린다고 해도 이를 100번 반복하면 총 1초가 소요됩니다. 단순한 계산으로도 배치를 통해 얻는 성능상의 이점을 넘어서는 수치임을 알 수 있습니다.

그런데 MySQL이 제공하는 `rewriteBatchedStatements` 옵션을 활용한다면 여러 건의 INSERT 쿼리를 하나의 BULK INSERT 쿼리로 개선할 수 있습니다.

```
jdbc:mysql://localhost:3306/momo?rewriteBatchedStatements=true
```

MySQL JDBC URL 에 `rewriteBatchedStatements=true` 옵션을 추가해주면 MySQL JDBC 드라이버는 배치로 전달된 여러 개의 개별 INSERT 문을 하나의 BULK INSERT 문으로 재작성하는 과정을 가지고 아래와 같이 한 번의 SQL 문으로 여러 행을 삽입할 수 있습니다.

```sql
INSERT INTO member (name, age) 
VALUES ('jazz', 26), ('pedro', 26), ('baeky', 26), ('mark', 27), ('daon', 28);
```

실제 MySQL이 생성한 쿼리는 MySQL JDBC URL에 `profileSQL`, `logger`, `maxQuerySizeToLog` 옵션을 추가로 명시하여 확인할 수 있습니다. `
```
&profileSQL=true&logger=Slf4JLogger&maxQuerySizeToLog=9999`
```

### ORM과 Batch 철학의 충돌

지금까지 Hibernate가 제공하는 배치와 MySQL BULK INSERT 하는 방법에 대해 알아봤습니다. 바로 결론부터 말씀드리면 만약 기본 키에 대해 자동생성 방법 중 IDENTITY 전략을 취할 경우 위 방법들은 사용할 수 없습니다.

`@GeneratedValue(strategy = GenerationType.IDENTITY)` 전략을 사용할 경우 Hibernate의 `쓰기 지연(Write-Behind)` 철학과 `Batch Processing` 간의 충돌이 발생하고, Hibernate는 JDBC 레벨에서 Batch Processing을 비활성화 시킵니다.

> [12.2. Session batching](https://docs.jboss.org/hibernate/orm/5.4/userguide/html_single/Hibernate_User_Guide.html#batch-session-batch)  
> Hibernate disables insert batching at the JDBC level transparently if you use an identity identifier generator.

데이터베이스에 ID 생성 책임을 위임하는 `IDENTITY` 전략과 서로 이해관계가 맞지 않기 떄문입니다.기본적으로 엔티티를 영속성 컨텍스트에 영속화 하기 위해서는 값을 식별하기 위한 ID가 반드시 필요합니다. `IDENTITY` 전략에서 ID를 획득할 수 있는 유일한 방법은 SQL을 실행시키는 것입니다. 따라서 persist 메서드가 호출될 때 INSERT가 실행되며 flush 시점까지 지연시킬 방법이 없습니다. 개별 엔티티들을 영속화 하는 시점에 `em.flush`가 호출되게 되어 배치 작업이 이루어지지 않고 개별 쿼리가 발생하게 되는 것입니다.

만약 `SEQUENCE` 전략일 경우 어떻게 될까요? `SEQUENCE` 전략이란 데이터베이스의 시퀀스 객체를 사용하여 기본 키를 생성하는 방식을 말합니다. MySQL에서는 `SEQUENCE` 전략을 사용할 수 없지만, `allocationSize` 속성을 통해 한 번의 시퀀스 호출로 여러 개수의 식별자를 메모리로 가져올 수 있어 MySQL을 제외한 다른 데이터베이스에서는 `IDENTITY` 만큼 자주 사용됩니다. 엔티티를 영속성 컨텍스트에 추가하려고 시도하면 즉시 시퀀스를 호출하여 새로운 ID를 할당 받는데 `IDENTITY`와는 달리 flush를 통한 데이터 삽입이 일어나지 않기 때문에 Hibernate의 지연 로딩을 활용할 수 있고 배치 철학과 통합할 수 있습니다.

결론적으로 기본 키 생성 전략을 IDENTITY 로 사용할 경우, JPA 레벨에서 배치 기능을 사용할 방법은 존재하지 않습니다. 또한 애초에 `ORM(Object-relational mapping)`이 내세우는 장점들을 생각해보면 ORM은 배치 처리나 BULK 쿼리를 다루기에 적합한 도구가 아님을 알 수 있습니다. 

다음은 Hibernate 공식 문서의 내용입니다.

> [12.2.1. Batch inserts](https://docs.jboss.org/hibernate/orm/5.4/userguide/html_single/Hibernate_User_Guide.html#batch-session-batch-insert)  
> When you make new objects persistent, employ methods flush() and clear() to the session regularly, to control the size of the first-level cache.

```java
EntityManager entityManager = null;
EntityTransaction txn = null;
try {
	entityManager = entityManagerFactory().createEntityManager();

	txn = entityManager.getTransaction();
	txn.begin();

	int batchSize = 25;

	for ( int i = 0; i < entityCount; i++ ) {
		if ( i > 0 && i % batchSize == 0 ) {
			//flush a batch of inserts and release memory
			entityManager.flush();
			entityManager.clear();
		}

		Person Person = new Person( String.format( "Person %d", i ) );
		entityManager.persist( Person );
	}

	txn.commit();
} catch (RuntimeException e) {
	if ( txn != null && txn.isActive()) txn.rollback();
		throw e;
} finally {
	if (entityManager != null) {
		entityManager.close();
	}
}
```

내부 구현을 살펴보면 `batchSize`를 기준으로 `EntityManager`가 주기적으로 `flush()`와 `clear()`를 호출하며 영속성 컨텍스트를 초기화하는 것을 확인할 수 있습니다. `batchSize`가 제한되어 있지 않다면 영속성 컨텍스트에 모든 엔티티가 올라가게 되고 최악의 경우 `OutOfMemoryException`이 발생할 수 있기 때문입니다. Hibernate는 영속성 컨텍스트에 한 번에 많은 양의 엔티티를 올리는 안티패턴을 공식적으로 경고하고 있습니다.

> Example 431. Naive way to insert 100 000 entities with Hibernate

```java
EntityManager entityManager = null;
EntityTransaction txn = null;
try {
	entityManager = entityManagerFactory().createEntityManager();

	txn = entityManager.getTransaction();
	txn.begin();

	for ( int i = 0; i < 100_000; i++ ) {
		Person Person = new Person( String.format( "Person %d", i ) );
		entityManager.persist( Person );
	}

	txn.commit();
} catch (RuntimeException e) {
	if ( txn != null && txn.isActive()) txn.rollback();
		throw e;
} finally {
	if (entityManager != null) {
		entityManager.close();
	}
}
```

> [12.2. Session batching](https://docs.jboss.org/hibernate/orm/5.4/userguide/html_single/Hibernate_User_Guide.html#batch-session-batch)  
> Hibernate caches all the newly inserted Customer instances in the session-level cache, so, when the transaction ends, 100 000 entities are managed by the persistence context. If the maximum memory allocated to the JVM is rather low, this example could fail with an OutOfMemoryException. The Java 1.8 JVM allocated either 1/4 of available RAM or 1Gb, which can easily accommodate 100 000 objects on the heap.  
> long-running transactions can deplete a connection pool so other transactions don’t get a chance to proceed.  
> JDBC batching is not enabled by default, so every insert statement requires a database roundtrip. To enable JDBC batching, set the hibernate.jdbc.batch_size property to an integer between 10 and 50.

Hibernate는 공식적으로 10에서 50사이의 `batchSize`를 권장하고 있습니다. 배치 처리 기능을 지원하기는 하지만 위 내용들을 살펴봤을 때 Hibernate의 배치 기능은 애초에 대량 데이터를 처리하는 목적이 아님을 알 수 있습니다. 또한 MySQL을 사용하면 대부분 ID에 대해 `IDENTITY` 전략을 사용하게 됩니다. 

만약 Hibernate를 사용하는 환경에서 특정 API가 대량의 데이터를 다뤄야 하는 상황이 생기면 어떻게 해야할까요? 새로운 기술을 도입해야할까요? 키 생성 전략을 변경하거나 아예 다른 데이터베이스로 변경해야할까요? 

## JDBC를 활용한 Batch 처리
