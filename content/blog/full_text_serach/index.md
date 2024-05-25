+++
title="[PostgreSQL] 검색을 위한 Full Text Search"
date = 2024-05-19
updated = 2024-05-19
description = "PostgreSQL에서 제공하는 Full Text Search 기능을 사용하여 검색 기능을 구현하는 방법을 알아봅니다."

[taxonomies]
tags = ["full-text-search", "index", "database", "postgresql", "search"]

[extra]
giscus = true
quick_navigation_buttons = true
toc = true
+++


# Full Text Search란?

**전체 텍스트 검색(Full Text Search)** 은 자연어(Natural Language)로 작성된 질의를 사용하여 정보를 검색하는 방법입니다. 이 방법은 선택적으로 질의와의 관련성을 평가하고, 그에 따라 검색 결과를 정렬하는 데 사용됩니다.

가장 일반적인 검색 유형이라고 한다면, 주어진 검색어가 포함된 모든 문서를 찾아서 그 문서들이 검색어와 얼마나 유사한지에 따라 순서대로 반환하는 것입니다.

사용자가 "수원시 맛집"이라는 검색한다고 가정해 봅시다.

1. 검색어 처리: 검색 엔진은 "수원시"와 "맛집"이라는 두 단어로 검색어를 분리합니다.
2. 문서 검색: DB에서 이 두 단어가 포함된 모든 문서를 찾습니다.
3. 관련성 평가: 각 문서에 대해 "수원시"와 "맛집"이 얼마나 자주 나타나는지를 평가합니다.
4. 결과 정렬: 등장 빈도가 높은 문서일수록 더 관련성이 높다고 판단하여 상위에 배치합니다.

위 연산에 대한 결과로, 

- 문서 A에는 "수원시"가 5번, "맛집"이 3번 나타나고,
- 문서 B에는 "수원시"가 2번, "맛집"이 7번 나타난다면,

검색 결과는 B가 A보다 더 상위에 표시될 것입니다. (5+3 < 2+7)

정말 간단한 예시이지만, 이렇게 검색어와 문서의 관련성을 평가하여 검색 결과를 정렬하는 것이 **Full Text Search** 의 기본 원리입니다.

이와 같이 **Full Text Search** 는 사용자가 입력한 검색어와 가장 관련성이 높은 문서를 신속하게 찾아줍니다. 

## 과거의 검색 방법의 문제점

Textual 검색 연산자는 과거로부터 데이터베이스에 존재해 왔습니다. PostgreSQL에는 텍스트 데이터타입에 대한 `~`, `~*`, `LIKE`, `ILIKE` 연산자가 있지만, 최근의 검색 엔진의 요구사항을 충족하기에는 부족합니다.

1. 언어적 최적화가 되어있지 않습니다. 예를 들어 정규식 표현에서는 "satisfies", "satisfy" 와 같은 파생어를 찾아내지 못한다는 부분이 있습니다. 만약 "satisfy" 를 검색한다면 "satisfies" 가 포함된 문서를 찾고 싶어도 놓치는 경우가 발생한다는 문제가 있죠.

    물론 `OR` 연산자를 사용하여 "satisfies" 와 "satisfy" 를 모두 검색할 수 있지만, 일부 단어에서는 수천개의 파생형이 존재할 수 있기 때문에 이 방법은 비효율적이라고 볼 수 있습니다.

2. 그리고 검색 결과의 순서(Ranking)를 제공하지 않기 때문에 일치하는 문서가 수천개가 발견될 대 효율적이지 않습니다.

3. 마지막으로 Index 지원이 없기 때문에 모든 검색에 대해 전체 테이블을 스캔해야 합니다. 이는 대량의 데이터를 가진 테이블에서는 **매우 느립니다**.

## Full Text Search Index

Full Text Indexing은 문서를 사전에 적절하게 처리한 뒤 나중에 빠르게 검색할 수 있도록 Index를 만드는 방법입니다.

전처리 과정에서는 다음과 같은 작업을 수행합니다.

1. **문서를 토큰으로 파싱하기. (Tokenization)**: 숫자, 단어, 특수문자, 이메일 주소 등 다양한 클래스의 토큰을 식별해서 이 클래스들이 다르게 처리되도록 합니다. 원칙적으로는 Token 클래스는 특정 애플리케이션에 따라 다를 수 있지만, 대부분의 경우 미리 정의된 클래스 셋을 사용하는 것이 적절합니다. 

    PostgreSQL은 *parser*를 사용해서 이 작업을 수행합니다. 기본적으로 standard parser가 제공되며 사용자 정의 *parser*를 만들 수도 있습니다.

    - 문장: "AI is transforming the world"
    - 토큰: ["AI", "is", "transforming", "the", "world"]

2. **토큰을 렉셈으로 변환하기. (Lexical Analysis)**: 어휘(Lexeme)는 Token과 마찬가지로 문자열이지만, 같은 단어의 다른 형태가 비슷하게 만들어지도록 정규화되어 있습니다. 예를 들어, 정규화에는 대문자를 소문자로 변환하거나, 복수형을 단수형으로 변환하는 접미사(영어의 s 또는 es)를 제거하는 등의 작업이 포함됩니다.

    이렇게 하면 검색에서 "satisfy"를 입력했을 때 "satisfies"가 포함된 문서도 검색 결과로 나타낼 수 있습니다.

    또한, 이 단계에서 불용어(Stopword)를 제거할 수도 있습니다. 불용어는 검색 결과에 쓸모가 없는 단어로, 검색 결과에 포함되지 않도록 처리합니다. (간단히 말해 Token은 문서 텍스트의 원시 조각인 반명에 Lexeme은 Token의 Indexing 및 검색에 유용하다고 여겨지는 단어입니다.) 

    PostgreSQL은 *dictionary*를 사용해서 이 작업을 수행합니다. *parser*와 마찬가지로 standard dictionary가 제공되며 사용자 정의 dictionary를 만들 수도 있습니다.

    - 정규화: "AI" → "ai", "transforming" → "transform"
    - 불용어 제거: ["is", "the"] 제거
    - 렉셈: ["ai", "transform", "world"]

3. 검색을 최적화하기 위해 문서를 적절히 전처리하여 저장합니다. 예를 들어 각 문서는 정규화된 렉셈의 정렬된 리스트로 표현할 수 있습니다. 

    추가로 높은 검색 성능을 위해서 렉셈과 함께 **위치 정보를 저장해서** 쿼리 텍스트와 일치하는 단어들이 밀집된 영역을 포함하는 문서가 흩어져 있는 문서보다 높은 순위에 랭크될 수 있도록 할 수 있습니다.

    - 정렬된 렉셈 리스트: ["ai", "transform", "world"]
    - 위치 정보: {"ai": [0], "transform": [1], "world": [2]}

## Dictionary

앞에 얘기한 Dictionary를 잘 사용한다면 Token 정규화를 더 세밀하게 제어할 수 있습니다. 

이를 위해 다음과 같은 작업을 수행할 수 있죠.

1. **Stopword**: 불용어를 제거하는 데 사용됩니다. 불용어는 검색 결과에 쓸모가 없는 단어로, 검색 결과에 포함되지 않도록 처리합니다.

    - 불용어는 "a", "the", "and" 같은 매우 자주 등장하지만 검색에 도움이 되지 않는 단어들입니다.
    - 이런 단어를 제거하면 인덱스 크기를 줄이고 검색 속도를 높일 수 있습니다.
    - 예를 들어 "kwanok is a developer" 라는 문장이 있다면, "is"와 "a"는 불용어로 처리하여 인덱스에 포함하지 않습니다.

2. **Synonym**: 동의어를 처리하는 데 사용됩니다. 동의어는 같은 의미를 가지는 단어들을 의미합니다.

    - 예를 들어 "search"와 "find"는 같은 의미를 가지는 단어입니다. 이런 경우 "search"를 입력했을 때 "find"가 포함된 문서도 검색 결과로 나타낼 수 있습니다.
    - [Ispell](https://en.wikipedia.org/wiki/Ispell) 사전을 사용하여 동의어를 하나의 단어로 매핑할 수 있습니다.

3. **Thesaurus**: 유의어를 처리하는 데 사용됩니다. 유의어는 비슷한 의미를 가지는 단어들을 의미합니다.

    - **구(phrases)**는 여러 단어로 이루어진 표현입니다.
    - 유의어 사전(Thesaurus)을 사용하여 이러한 구를 하나의 단어로 매핑할 수 있습니다.
    - 예를 들어 "Amazon Web Services"와 "AWS"는 같은 의미를 가지는 구입니다. 이런 경우 "Amazon Web Services"를 입력했을 때 "AWS"가 포함된 문서도 검색 결과로 나타낼 수 있습니다.

4. **Canonicalization**: 단어의 형태를 통일하는 데 사용됩니다. 예를 들어, 대문자를 소문자로 변환하거나, 복수형을 단수형으로 변환하는 등의 작업이 포함됩니다.

    - 예를 들어 "run", "runs", "running"은 모두 같은 의미를 가집니다.
    - 이런 경우 "run", "runs", "running" 을 모두 "run"으로 매핑하여 검색 결과를 통일할 수 있습니다.

5. **Stemming**: 어근을 추출하는 데 사용됩니다. 어근은 단어의 핵심 부분을 의미합니다.

    - **스테머(Stemmer)** 는 단어의 어근을 추출하는 알고리즘입니다.
    - Snowball stemmer 룰을 사용해서 단어의 변형된 형태를 정형화된 형태로 매핑할 수 있습니다.
    - 예를 들어 "connect", "connected", "connecting"은 모두 "connect"로 매핑할 수 있습니다.


# 마치며

- [Text Search Types](https://www.postgresql.org/docs/current/datatype-textsearch.html): tsvector, tsquery 비교
    - tsvector: 전처리된 문서를 저장하는 데이터 타입
    - tsquery: 전처리된 쿼리를 저장하는 데이터 타입
- 매치 연산자(@@): 문서와 질의의 일치 여부를 확인하는 연산자 ([Match Operator](https://www.postgresql.org/docs/current/textsearch-intro.html#TEXTSEARCH-MATCHING))
- 인덱스: 전체 텍스트 검색을 위한 인덱스를 생성하는 방법 ([Indexing](https://www.postgresql.org/docs/current/textsearch-indexes.html))

해당 기능들은 PostgreSQL의 다른 공식문서에서 자세히 다루고 있으며 이번 포스트에서는 간단히 소개만 했습니다.

앞으로의 포스트는 Full Text Search에 대해 더 깊이있게 다루어보겠습니다.


# 참고자료

- [PostgreSQL Full Text Search](https://www.postgresql.org/docs/current/textsearch-intro.html)