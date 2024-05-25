+++
title="[PostgreSQL] 검색을 위한 tsvector와 tsquery"
date=2024-05-20 00:10:00
updated=2024-05-20
description="PostgreSQL에서 Full Text Search를 위한 데이터 타입인 tsvector와 tsquery에 대해 알아봅니다."

[taxonomies]
tags=["full-text-search", "index", "database", "postgresql", "search"]

[extra]
toc=true
giscus=true
quick_navigation_buttons=true
+++

# Text Search 타입들

PostgreSQL에서는 Full Text Search를 위한 데이터 타입으로 `tsvector`와 `tsquery`를 제공합니다.

## tsvector

`tsvector`는 렉셈(Lexemes, key words)의 리스트로 볼 수 있습니다. 중복이 제거되고 정렬된 형태로 저장됩니다.

```sql
SELECT 'legends never die when the world is calling you'::tsvector;
                    tsvector
----------------------------------------------
'calling' 'die' 'is' 'legends' 'never' 'the' 'when' 'world' 'you'
```

공백이나 문장 부호가 포함된 렉셈을 표현하려면 따옴표로 묶어서 표시합니다.
    
```sql
SELECT $$don't '     ' ever say it's over if i'm breathin'$$::tsvector;
                    tsvector
----------------------------------------------
'     ' 'breathin''' 'don''t' 'ever' 'i''m' 'if' 'it''s' 'over' 'say'
```

위에 예시를 보면 이상한 점이 있습니다 - `don't`가 `don''t`로 변환되었습니다. 이는 PostgreSQL에서 특수 문자를 처리하는 방법 때문입니다. 특수 문자는 두 번 반복되어 저장됩니다.

이건 선택사항인데 정수형 포지션 값을 렉셈에 추가할 수 있습니다. 이는 렉셈이 원본 텍스트에서 어디에 위치하는지를 나타냅니다.

```sql
SELECT $$Every:1 time:2 you:3 pop:4 off:5 they:6 hopin':7 that:8 you:9 fall:10 hard:11$$::tsvector;
                    tsvector
----------------------------------------------
'Every':1 'fall':10 'hard':11 'hopin''':7 'off':5 'pop':4 'that':8 'they':6 'time':2 'you':3,9
```

일반적으로 포지션 값은 렉셈이 원본 텍스트에서 나타나는 순서를 나타냅니다. 포지션 값은 __proximity ranking__ 에 사용되고, 값은 1 ~ 16383 사이의 정수입니다. 숫자가 클수로 자동으로 16383으로 세팅됩니다. 동일한 어휘에 대한 중복되는 위치는 무시됩니다.

포지션 값을 가지는 렉셈은 가중치를 추가로 지정할 수 있으며 A, B, C 또는 D가 될 수 있습니다. 기본값은 D이고, 출력에는 표시되지 않습니다.

```sql
SELECT $$Ezreal:1A is:2B an:3C explorer:4D$$::tsvector;
                    tsvector
----------------------------------------------
'Ezreal':1A 'an':3C 'explorer':4 'is':2B
```

가중치는 일반적으로 제목같은 본문 내용과 구분해서 표시할 때 사용할 수 있습니다. 텍스트 검색 

`tsvector` 자체는 렉셈 정규화를 수행하지 않으며, 주어진 단어가 적합하게 정규화되었다고 가정합니다.

```sql
SELECT 'The herald of a new age of technology'::tsvector;
                    tsvector
----------------------------------------------
'The' 'a' 'age' 'herald' 'new' 'of' 'technology'
```

대부분의 영어 텍스트 검색에서는 위의 단어는 정규화되지 않은 것으로 간주됩니다. 예를 들어, `The`는 `the`로 변환되어야 하죠.

그래서 원본 문서의 텍스트는 검색에 적합하지 않기 때문에 `to_tsvector()` 함수를 사용하여 `tsvector`로 변환해야 합니다.

```sql
SELECT to_tsvector('The herald of a new age of technology');
                    to_tsvector
---------------------------------------------------
'age':6 'herald':2 'new':5 'technolog':8
```

## tsquery

`tsquery`는 렉셈의 집합으로 볼 수 있습니다. 렉셈은 `&`, `|`, `!`, `(`, `)` 그리고 구문 검색 연산자인 `<->(FOLLOWED BY)` 를 사용해서 조합할 수 있습니다.

검색중인 두 렉셈 사이의 거리를 나타내는 `<N>` 연산자의 변형인 `<->`도 있습니다. (`<->` 은 `<1>`과 동일합니다.)

그리고 괄호를 사용해서 이러한 연산자들을 그루핑할 수 있습니다. 괄호가 없는 경우엔 !(NOT)이 가장 높은 우선순위를 가지고 <->(FOLLOWED BY)가 그 다음으로 묶이고, 그 다음으로는 &(AND), |(OR)가 묶입니다.

```sql
SELECT 'new & age'::tsquery;
            tsquery
----------------------------
'new' & 'age'

SELECT 'technology & (new | age)'::tsquery;
            tsquery
----------------------------
'technology' & ( 'new' | 'age' )

SELECT 'technology & new & !age'::tsquery;
            tsquery
----------------------------
'technology' & 'new' & !'age'
```

추가적으로 `tsquery`의 렉셈은 하나 이상의 가중치를 지정해서 해당 가중치중 하나를 가진 `tsvector`와 렉셈과만 일치하도록 제한할 수도 있습니다.

```sql
SELECT 'Dark:ab & Binding'::tsquery;
            tsquery
----------------------------
'Dark':AB & 'Binding'
```

또한 `tsquery`의 렉셈은 뒤에 *를 붙여서 해당 렉셈으로 시작하는 모든 렉셈과 일치하도록 제한할 수도 있습니다.

다음 예시는 `Freljord`로 시작하는 렉셈과 `Supporter` 렉셈을 찾습니다.

```sql
SELECT 'Freljord:* & Supporter'::tsquery;
            tsquery
----------------------------
'Freljord':* & 'Supporter'
```

쿼팅도 앞에 설명한 `tsvector`와 동일하게 작동합니다.

```sql
SELECT $$'he''s'$$::tsquery;
            tsquery
----------------------------
'he''s'
```

`tsquery`는 `to_tsquery()` 함수를 사용해서 `tsquery`로 변환할 수 있습니다.

```sql
SELECT to_tsvector('ThreshLantern') @@ to_tsquery('Thresh:*');
----------------
t
```

위 예시는 `ThreshLantern`이 `Thresh`로 시작하는 렉셈과 일치하는지 확인합니다.

# 참고자료

- [PostgreSQL Text Search Types](https://www.postgresql.org/docs/current/datatype-textsearch.html)