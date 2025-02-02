+++
title="Django ORM 동작 방식"
date=2025-02-02
updated=2025-02-02
description="업무를 하며 궁금했던 Django ORM 동작에 대해 알아봤습니다."

[taxonomies]
tags=["django", "orm"]

[extra]
giscus = true
quick_navigation_buttons = true
toc=true
+++

# Django ORM의 동작 원리

이번 포스트는 평소 업무를 하며 궁금했던 Django ORM 동작을 소스코드를 보며 이해해보며 정리를 해봤습니다.

### 1. 모델(Model) 그리고 메타(_meta)

**모델 클래스 생성**

사용자가 model.Model을 상속받아서 모델 클래스를 정의하면, Django는 클래스 정의 시 내부적으로 해당 모델의 필드 정보와 관계 (ForeignKey, ManyToManyField), 그 외 메타 정보들 (테이블 명이나 인덱스) 를 모델의 `_meta` 속성에 저장합니다.

이런 모델이 있다고 가정해보겠습니다.

```python
from django.db import models

class User(models.Model):
    name = models.CharField(max_length=100)
    email = models.EmailField(unique=True)
    age = models.IntegerField()

class Profile(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    bio = models.TextField()

```

메타 데이터는 SQL 생성 시, 데이터베이스 테이블과 컬럼을 매핑하는데 사용되고, 이후 쿼리 컴파일러가 모델의 구조를 참고해서 SQL을 생성합니다.

### 2. 쿼리셋(QuerySet)과 쿼리(Query) 그리고 Lazy Evaluation

**QuerySet 생성**

예를 들어,

```python
이십살_이상_유저들 = User.objects.filter(age__gte=20)
```

이런 코드를 실행하면, 실제로 QuerySet 객체가 생성되지만, 내부적으로 쿼리가 실행되지 않습니다. 

이 QuerySet은 내부의 Query 객체를 포함하고 있으며, 이 Query 객체는 filter, order, limit, join 등의 쿼리 연산의 모든 조건을 트리 형태로 저장합니다.

```python
이십살_이상_유저들 = User.objects.filter(age__gte=20).order_by('-age')
```

이렇게 체인 형태의 메서드 호출은 모두 `Query` 객체에 조건을 추가하는 작업일 뿐, 실제 SQL 컴파일이나 실행은 **평가(evaluation)** 시점에 일어납니다.

그리고 이 것들은 Django ORM의 핵심 개념인 Lazy Evaluation 이라고 부릅니다.

### 3. SQL 컴파일 (Query → SQL)

- `get_compiler()` 호출
QuerySet이 평가되는 순간, 내부적으로 `Query.get_compiler(using=...)` 메서드가 호출되어서 우리가 연결한 데이터베이스에 맞는 `SQLCompiler` 객체를 생성합니다.

```python
# django/db/models/query __iter__ 예시
def __iter__(self):
    queryset = self.queryset
    db = queryset.db
    compiler = queryset.query.get_compiler(using=db)  # 요기

    result = compiler.execute_sql(
        chunked_fetch=self.chunked_fetch, chunk_size=self.chunk_size
    )

    ...

```

**그럼 SQLCompiler 역할은?**
- 쿼리 트리 순환: SQLCompiler는 Query 객체에 저장된 조건들을 순회하면서 각 조건을 SQL의 WHERE, JOIN, ORDER BY 등의 절로 변환합니다. (그래서 위에 과정이 필요하겠죠? ORM에선 PostgreSQL에 연결했는지 MYSQL에 연결했는지 모르니까요)
- 바인딩 파라메터 처리: 사용자로부터 받은 값들은 SQL Injection을 막기 위해 **파라메터 바인딩** ("%s" 또는 "?") 으로 처리되고 이 과정에서 파라메터 리스트가 함께 구성됩니다.

여기서 트리 순환이라고 하는 이유는 쿼리를 만들 때 그냥 코드의 순서대로 만들기보단, 내부에 저장된 쿼리의 구조 (filter, join, subquery 등을) 트리로 표현하고 각 노드를 순회하며 조립하는 과정이기 때문입니다.

예를 들어 다음과 같은 SQL이 있으면,

```python
User.objects.filter(
    Q(age__gt=20) & (Q(city='Seoul') | Q(city='Busan'))
)
```

```sql
WHERE (age > 20 AND (city = 'Seoul' OR city = 'Busan'))
```

이렇게 표현할 수 있을 겁니다.
```
        AND
       /    \
  (age > 20)   OR
              /   \
     (city = 'Seoul') (city = 'Busan')
```
- 루트 노드: AND
- 좌측 자식: age > 20
- 우측 자식: OR 노드, 그 아래에 두 개의 조건이 있음

SQL 컴파일러는 이 트리를 순회하면서 가장 앞부분에 AND 노드를 배치하고 그 다음 좌측과 우측을 각각 처리해서 최종적으로 하나의 SQL을 만드는 것입니다.



### 4. 데이터베이스 연결 및 SQL 실행

**DB Backend**
Django는 settings.py 에 설정하는 DATABASES 설정에 따라 DB 엔진과 연동합니다. (PostgreSQL, MySQL, sqlite3 ...)

**Cursor**
SQLCompiler가 생성한 SQL 쿼리와 파라메터는 django.db.connection의 cursor() 메서드를 통해 실제 DB 커서에 전달됩니다.

```python
# 내부 구현 코드
class RawQuery:
    ...

    def _execute_query(self):
        connection = connections[self.using]  # 여기서 사용중인 db 커넥션을 가져와서

        # Adapt parameters to the database, as much as possible considering
        # that the target type isn't known. See #17755.
        params_type = self.params_type
        adapter = connection.ops.adapt_unknown_value
        if params_type is tuple:
            params = tuple(adapter(val) for val in self.params)
        elif params_type is dict:
            params = {key: adapter(val) for key, val in self.params.items()}
        elif params_type is None:
            params = None
        else:
            raise RuntimeError("Unexpected params type: %s" % params_type)

        self.cursor = connection.cursor()  # 커서를 가져오고
        self.cursor.execute(self.sql, params)  # 파라메터와 함께 RawQuery를 실행
```

**Transaction Managing**
Django는 각 HTTP 요청 단위나 명시적으로 transaction.atomic() 블록 내에서 자동으로 트랜잭션을 관리하는데, SQL 실행 전후로 commit 또는 rollback을 처리합니다.
```python
# django/db/backends/base/base 구현 중...
class BaseDatabaseWrapper:
    ...

    def _commit(self):
        if self.connection is not None:
            with debug_transaction(self, "COMMIT"), self.wrap_database_errors:
                return self.connection.commit()

    ...

    @async_unsafe
    def close(self):
        """Close the connection to the database."""
        self.validate_thread_sharing()
        self.run_on_commit = []

        # Don't call validate_no_atomic_block() to avoid making it difficult
        # to get rid of a connection in an invalid state. The next connect()
        # will reset the transaction state anyway.
        if self.closed_in_transaction or self.connection is None:
            return
        try:
            self._close()
        finally:
            if self.in_atomic_block:
                self.closed_in_transaction = True
                self.needs_rollback = True
            else:
                self.connection = None
```

### 5. 결과 페칭 및 모델 인스턴스 생성

**데이터 수신**
DB에서 반환된 결과 (tuple list 로 나옴) → SQLCompiler가 생성한 SQL에 의해 가져온 raw 데이터입니다.

**모델 인스턴스 매핑**
Django ORM은 이 raw 데이터를 모델 인스턴스로 변환합니다. 그리고 이 과정에서
  - 정리: 각 행의 값들이 모델 필드 순서에 맞게 할당
  - **필드 변환**: 각 필드 클래스(CharField, IntegerField 등)는 `from_db_value()` 메서드를 통해 **데이터베이스 타입** 을 **파이썬 타입** 으로 캐스팅합니다.
  - 캐싱: QuerySet은 평가 후 결과를 _result_cache에 저장하고 이후 동일한 QuerySet의 반복에서 추가 SQL 실행을 방지합니다.
    ```python
    # 그래서 웬만한 코드에 다 이렇게 돼있음
    qs._fetch_all()
    return qs._result_cache # 를 사용한 무언가...
    ```

### 6. 그 외 추가 최적화 작업들

**Deffered Fields**

특정 필드를 나중에 로드하도록 지정하면, 초기 쿼리에서 해당 필드들을 제외하고, 실제로 필요할 때 추가 쿼리를 통해 가져옵니다.

**`select_related()`/`prefetch_related()`**

ForeignKey나 ManyToMany 관계 데이터를 가져오려면 SQL Join 또는 별도의 쿼리를 통해서 관계가 있는 데이터를 함께 로드하는 전략도 있습니다. (보통 N+1 방지)

**QuerySet 체이닝 & 내부 캐싱**

여러 번의 체이닝이나 필터링 작업이 `Query` 객체에 쌓이고 한 번 평가된 QuerySet은 위에서 말한 캐싱을 활용해서 불필요한 호출을 줄입니다.

# 전체 플로우 요약 ✅

1. 모델 정의: 모델 클래스 정의 → 메타 데이터(_meta) 생성
2. QuerySet 생성: `User.objects.filter(...)` → 내부적으로 Query 객체 생성 (Lazy Evaluation)
3. SQL 컴파일: QuerySet 평가 시 `get_compiler()`를 호출 → SQLCompiler가 내부 Query 트리를 SQL 쿼리 문자열과 파라미터로 변환
4. SQL 실행: 생성된 SQL과 파라미터가 데이터베이스 커넥션을 통해 실행 → 결과가 튜플 형태로 반환
5. 모델 인스턴스 생성: 반환된 행 데이터를 모델 인스턴스로 매핑 → 최종 결과를 사용자에게 제공
6. 최적화 및 캐싱: select_related, prefetch_related, deferred fields 등의 최적화 기법이 적용되어 성능 향상

