+++
title="[PostgreSQL] 검색을 위한 Document"
date=2024-05-19 22:45:00
updated=2024-05-19 22:45:00
description="PostgreSQL에서 제공하는 Full Text Search 기능을 사용하여 검색 기능을 구현하는 방법을 알아봅니다."

[taxonomies]
tags=["document", "postgresql", "database", "search"]

[extra]
giscus = true
quick_navigation_buttons = true
toc = true
+++

# Document란?

**Document**는 Full Text 서치에서 사용하는 검색의 단위입니다.

잡지 기사, 이메일 메시지, 웹 페이지 등이 될 수 있죠. 텍스트 검색 엔진은 Document의 구문을 분석하고 렉셈(Lexemes, key words)의 상위 Document의 연관성을 저장할 수 있어야 합니다.

그리고 이러한 연관성은 우리가 찾으려는 텍스트가 포함된 Document를 검색ㅎ할 때 사용됩니다.

PostgreSQL 내 검색에서 Document는 일반적으로 텍스트형 필드라고 생각할 수 있습니다. 또는 여러 테이블에 저장된 필드의 결합이나 연결이 될 수도 있습니다.

이메일을 검색한다고 하면, 테이블에는 `title`, `content`, `sender`, `receiver` 등의 필드가 있을 수 있습니다. 이 필드들이 Document를 구성하고 Full Text 서치를 통해 여러 필드를 한 번에 검색할 수 있다는 의미입니다.

```sql
SELECT title || ' ' || content || ' ' || sender || ' ' || receiver AS document
FROM email_table
WHERE email_id = 1;

SELECT e.title || ' ' || e.content || ' ' || u.name AS document
FROM email_table e, user_table u
WHERE e.sender_id = u.user_id AND e.email_id = 1;
```

> 예제에서 단일 NULL 속성으로 인해 `||` 연산자를 사용하면 결과가 NULL이 될 수 있습니다. 이 경우 `COALESCE` 함수를 사용하여 NULL을 처리할 수 있습니다.

다른 방법 중 하나는 Document를 파일 시스템에 간단한 텍스트 파일로 저장하는 것입니다. 이런 경우 데이터베이스를 사용해서 Full Text Index를 저장하고, 검색할 수 있으며 Unique ID를 사용해서 파일시스템에서 Document를 검색할 수 있습니다.

하지만 데이터베이스 외부의 파일을 검색하는 것은 슈퍼유저 권한이나 특별한 권한이 필요할 수 있습니다. 그래서 일반적으로 데이터베이스에서 관리하는 것이 더 편리하다고 볼 수 있습니다.

또한 모든 데이터를 데이터베이스 내부에 보관하면 Document의 메타데이터에 쉽게 접근하여 Index를 활용한 검색을 할 수 있습니다.

텍스트 검색을 위해 각 문서는 전처리된 `tsvector` 형식으로 축소돼야 합니다. 검색과 랭킹은 이 `tsvector`를 기반으로 수행되며, 원본 텍스트는 사용자에게 보여줄 때만 사용됩니다. 따라서 우리는 종종 `tsvector`를 Document라고 말하지만, 이는 Document를 압축적으로 표현한 것으로 보면 됩니다.

# 참고자료

- [PostgreSQL Full Text Search #TEXTSEARCH-INTRO](https://www.postgresql.org/docs/current/textsearch-intro.html#TEXTSEARCH-INTRO)