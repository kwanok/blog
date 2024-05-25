+++
title="[PostgreSQL] 검색을 위한 Basic Text Matching"
date=2024-05-20 01:17:00
updated=2024-05-20
description="PostgreSQL에서 Full Text Search를 사용하여 기본적인 텍스트 매칭을 수행하는 방법을 알아봅니다."

[taxonomies]
tags=["basic-text-matching", "postgresql", "database", "search"]

[extra]
giscus = true
quick_navigation_buttons = true
toc = true
+++

# Basic Text Matching

PostgreSQL에서 Full Text Search는 `@@` 이라는 일치 연산자를 씁니다. `tsvector`(Document)가 `tsquery`(Query)와 일치하면 `true`를 반환하죠.

> 여기서 어떤 데이터 타입이 먼저 쓰이는지는 중요하지 않습니다. 예시로 보시죠.

```sql
SELECT 'legends never die'::tsvector @@ 'legends'::tsquery;
?column?
--------
t

SELECT 'legends'::tsquery @@ 'legends never die'::tsvector;
?column?
--------
t

-- 순서가 바뀌어도 결과는 같습니다.
```

위의 예시를 보면, `tsquery`와 `tsvector`는 단순히 텍스트가 아니라는 걸 알 수 있습니다.

`tsquery`는 검색어를 나타내고, 이미 정규화된 렉셈을 포함하고 있습니다. 그리고 AND, OR, NOT, FOLLOWED BY 같은 연산자를 사용해서 복합적인 검색을 수행할 수 있습니다.

`tsvector`는 검색할 문서를 나타내며, 렉셈의 리스트로 구성되어 있습니다. 이 렉셈은 검색어로부터 파싱된 것이며, 검색어의 위치(Position)를 포함하고 있습니다.

다음 예시는 Tresh와 Lantern이라는 두 단어가 포함된 문서를 검색하는 예시입니다.

```sql
SELECT to_tsvector('english', 'Thresh Lantern is a skill of Thresh.') @@ to_tsquery('Thresh & Lantern');
?column?
--------
t
```

조금 더 복잡한 예시를 보겠습니다.

```sql
-- AND, OR, NOT 연산자를 사용한 예시
SELECT to_tsvector('english', 'Thresh Lantern is a skill of Thresh.') @@ to_tsquery('Thresh & (Lantern | Hook) & !Shield');
?column?
--------
t
```

위 예시는 "Thresh"와 "Lantern" 또는 "Hook"을 포함하지만 "Shield"는 포함하지 않는 문서를 찾습니다.

만약 "Shield"가 위 예시에 포함돼있다면 어떨까요? 결과는 `false`가 될 것입니다.

```sql
SELECT to_tsvector('english', 'Thresh Lantern is a skill of Thresh. Lantern grants a shield to nearby champions.') @@ to_tsquery('Thresh & (Lantern | Hook) & !Shield');
?column?
--------
f
```

또한 `@@` 연산자는 텍스트 입력도 지원해서 간단히 텍스트 자체를 `tsvector` 또는 `tsquery`로 명시적으로 변환해서 그냥 건너뛸 수도 있습니다.

```sql
tsvector @@ tsquery  -- 전처리된 Document와 전처리된 Query를 비교
tsquery  @@ tsvector  -- 전처리된 Query와 전처리된 Document를 비교
text @@ tsquery     -- text를 자동으로 tsvector로 변환해서 비교
text @@ text     -- 두 텍스트를 각각 tsvector와 plainto_tsquery로 변환해서 비교
```

**FOLLOWED BY** 연산자(`<->`)는 두 단어가 순서대로 인접하게 나오는지 확인합니다.

> (`<N>`) 연산자는 두 단어 사이의 거리를 나타냅니다.

```sql
-- "Thresh Hook"이라는 문구가 인접하게 나오나 확인하는 예시
SELECT to_tsvector('Thresh Hook is a powerful skill.') @@ to_tsquery('Thresh <-> Hook');
?column?
--------
t
```

위 예시는 "Thresh"와 "Hook"이 인접하게 나오는 문서를 찾습니다.

반대의 예시로 보여드리면 이렇게 `false`가 나옵니다.

```sql
SELECT to_tsvector('Hook is used by Thresh.') @@ to_tsquery('Thresh <-> Hook');
?column?
--------
f
```

추가로 `<N>` 연산자를 사용해서 두 단어 사이의 거리를 지정할 수 있습니다.

```sql
-- 다른 단어가 1개 껴있는 경우
SELECT to_tsvector('Thresh uses his Hook skill.') @@ to_tsquery('Thresh <2> Hook');
?column?
--------
t
```

## `phraseto_tsquery`

`phraseto_tsquery` 함수는 여러 단어 사이에 불용어(Stopword)가 포함된 문장을 검색할 때 쓸 수 있습니다.

```sql
SELECT phraseto_tsquery('Thresh uses Hook');
    phraseto_tsquery
-----------------------
'thresh' <-> 'use' <-> 'hook'

SELECT phraseto_tsquery('Thresh uses his Hook skill');
    phraseto_tsquery
-----------------------
'thresh' <-> 'use' <2> 'hook' <-> 'skill'

-- <2>: uses와 hook 사이에 정확히 1개의 단어가 껴있다. -> "uses his Hook"
```


# 참고자료

- [PostgreSQL Full Text Search #TEXTSEARCH-MATCHING](https://www.postgresql.org/docs/current/textsearch-intro.html#TEXTSEARCH-MATCHING)