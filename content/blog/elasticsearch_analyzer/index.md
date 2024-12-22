+++
title="[Elasticsearch] Analyzer - 문서를 효율적으로 색인하는 방법"
date = 2024-12-22
updated = 2024-12-22
description = "Elasticsearch에서 제공하는 Analyzer를 사용하여 텍스트를 효율적으로 색인하는 방법을 알아봅니다."

[taxonomies]
tags = ["elasticsearch", "analyzer", "search"]

[extra]
giscus = true
quick_navigation_buttons = true
toc = true
+++

# Elasticsearch는 왜 성능이 좋을까요?

Elasticsearch는 **"강력한 검색 엔진"** 이라는 평가를 받으며, 많은 개발자들이 검색 기능을 구현할 때 많은 고민 없이 선택하는 솔루션 중 하나입니다.

많은 분들이 Elasticsearch가 역색인(Inverted Index) 때문에 성능이 좋다고 생각하는데요,

이번 포스트에서는 좀 더 들어가 성능을 최적화하는 방법 중 하나인 **Analyzer** 에 대해 분석해볼게요.

# 텍스트 검색은 왜 어려울까요?

우선 수능 국어 7등급인 제가 느끼기엔, 사람의 언어는 단순하지 않습니다. 

우리가 사용하는 단어, 문장, 표현은 **문맥**과 **형태**에 따라 다양한 모습을 가지고 있기 때문입니다.

그럼 언어가 복잡한 이유를 한국어로 예를 들어서 설명해볼게요.

### 형태가 다른 단어 처리

- **"튀다"**, **"탈주하다"**, **"도망가다"** 는 서로 다른 형태이지만, 같은 의미로 검색되어야 합니다.

- 그런데 여기서 **"튀다"** 의 경우 핑이 튀다~, 공이 튀다~, 물이 튀다~ 등 다양한 의미로 사용될 수도 있죠.

### 불필요한 단어

- **"은"**, **"는"**, **"이"**, **"가"** 와 같은 조사는 문맥을 파악하는데 도움이 되지만, 텍스트 매칭(검색 시)에 있어서는 방해가 되는 감이 있죠.

- 이런 불필요한 단어들을 **불용어(Stopwords)** 라고 부릅니다.

- 만약 **"여의도 맛집"** 을 검색했는데 블로그 글이 다음과 같으면 1번의 경우 단순 매칭으로는 검색이 되지 않을 수 있습니다.
    1. **여의도에서** 인생 솥밥 **맛집을** 찾다
    2. 포브스 선정 **여의도** 분위기 **맛집** 30선

    → 1번의 경우 역색인 구조에서 **"여의도"** != **"여의도에서"** 로 처리될 수 있기 때문입니다.

### 띄어쓰기 문제

- **"붕어빵은 팥붕이지"**, **"붕어빵은팥붕이지"** 은 같은 의미를 가지지만, 띄어쓰기의 차이로 검색 정확도가 떨어질 수 있습니다.

# 언어의 복잡성을 해결하기 위한 Analyzer

Analyzer는 Elasticsearch에서 제공하는 **텍스트 전처리 도구** 입니다.

한마디로, 검색하기 쉽게 만들어놓자라는 거죠.

## Analyzer의 작동 과정

Analyzer는 텍스트를 단순히 **토큰화(Tokenization)** 하는 것을 넘어서, 다양한 전처리 과정을 거쳐 텍스트를 처리합니다.

## 1. 문자 필터(Char Filter)

- Analyzing 에서 가장 먼저 일어나는 일입니다.
- 텍스트에서 특수 문자를 제거하거나, HTML 태그를 제거하는 등의 전처리 작업을 수행합니다.

→ 예: `<p>장송의 프리렌 2기 언제나오냐?</p>` → `장송의 프리렌 2기 언제나오냐?`

Request:

```bash
curl -L -X POST 'localhost:9200/_analyze' \
-H 'Content-Type: application/json' \
-d '{
  "char_filter": [
    {
      "type": "html_strip"  // HTML 태그를 제거합니다.
    }
  ],
  "text": [
    "<p>장송의 프리렌 2기 언제 나오냐?</p>"
  ]
}'
```

Response: `<p></p>` 태그는 새로운 줄을 나타내기 때문에 `\n` 으로 대체된 걸 볼 수 있습니다.

```json
{
  "tokens": [
    {
      "token": "\n장송의 프리렌 2기 언제 나오냐?\n",
      "start_offset": 0,
      "end_offset": 25,
      "type": "word",
      "position": 0
    }
  ]
}
```

또한, mapping을 사용해서 원하는 조건을 걸 수도 있습니다.

예) `1.`을 `First)` 으로 바꾸고 싶다면? → `"1.=>First)"` 이렇게 매핑을 추가하면 됩니다.

Request: 저는 `\n`을 제거하고 싶어요.

```bash
curl -L -X POST 'localhost:9200/_analyze' \
-H 'Content-Type: application/json' \
-d '{
  "char_filter": [
    {
      "type": "html_strip" 
    },
    {
      "type": "mapping",
      "mappings": [
        "\\n=>"  // \n을 제거합니다.
      ]
    }
  ],
  "text": [
    "<p>장송의 프리렌 2기 언제 나오냐?</p>"
  ]
}'
```

Response: `\n`이 제거된 걸 볼 수 있습니다.

```json
{
  "tokens": [
    {
      "token": "장송의 프리렌 2기 언제 나오냐?",
      "start_offset": 3,
      "end_offset": 25,
      "type": "word",
      "position": 0
    }
  ]
}
```

## 2. 토큰화(Tokenization)

- Char Filter를 거치고 나면, 텍스트를 토큰화합니다.
- 토큰화는 텍스트를 **의미 기준으로 나누는 작업**을 수행합니다.
- 근데 다들 아시다시피, 공통어인 영어의 경우 띄어쓰기로 나누면 되지만, 한국어의 경우 띄어쓰기만으로는 부족합니다.
    → 한글에서는 형태소 분석기를 사용해서 단어를 나누면 됩니다.

→ 예: **"장송의 프리렌 2기 언제나오지"** → [**"장송의", "프리렌", "2기", "언제", "나오냐"**]

기본적으로 Elasticsearch는 **Standard Analyzer** 를 사용합니다.

Request Body:

```json
{
  "tokenizer": "standard",
  "text": [
    "장송의 프리렌 2기 언제 나오냐?"
  ]
}
```

Response: 역시 그냥 띄어쓰기로와 구두점으로만 나눠진 걸 확인할 수 있어요.

```json
{
  "tokens": [
    {
      "token": "장송의",
      "start_offset": 0,
      "end_offset": 3,
      "type": "<HANGUL>",
      "position": 0
    },
    {
      "token": "프리렌",
      "start_offset": 4,
      "end_offset": 7,
      "type": "<HANGUL>",
      "position": 1
    },
    {
      "token": "2기",
      "start_offset": 8,
      "end_offset": 10,
      "type": "<ALPHANUM>",
      "position": 2
    },
    {
      "token": "언제",
      "start_offset": 11,
      "end_offset": 13,
      "type": "<HANGUL>",
      "position": 3
    },
    {
      "token": "나오냐",
      "start_offset": 14,
      "end_offset": 17,
      "type": "<HANGUL>",
      "position": 4
    }
  ]
}
```

### 형태소 분석기 사용하기

한국어의 경우 **형태소 분석기** 를 사용해서 단어를 나눌 수 있습니다.

Elastic 에서 제공하는 Nori 라는 플러그인을 사용하면 이것도 쉽게 가능합니다.

설치 방법은 [여기](https://www.elastic.co/guide/en/elasticsearch/plugins/current/analysis-nori.html)를 참고해주세요. (설치 하고 반영 안되면 재부팅 해야해요...!)

Request Body:

```json
{
  "tokenizer": "nori_tokenizer",
  "text": [
    "장송의 프리렌 2기 언제 나오냐?"
  ]
}
```

Response: 한글 단어가 잘 나눠진 걸 확인할 수 있어요.

```json
{
  "tokens": [
    {
      "token": "장송",
      "start_offset": 0,
      "end_offset": 2,
      "type": "word",
      "position": 0
    },
    {
      "token": "의",
      "start_offset": 2,
      "end_offset": 3,
      "type": "word",
      "position": 1
    },
    {
      "token": "프리",
      "start_offset": 4,
      "end_offset": 6,
      "type": "word",
      "position": 2
    },
    {
      "token": "렌",
      "start_offset": 6,
      "end_offset": 7,
      "type": "word",
      "position": 3
    },
    {
      "token": "2",
      "start_offset": 8,
      "end_offset": 9,
      "type": "word",
      "position": 4
    },
    {
      "token": "기",
      "start_offset": 9,
      "end_offset": 10,
      "type": "word",
      "position": 5
    },
    {
      "token": "언제",
      "start_offset": 11,
      "end_offset": 13,
      "type": "word",
      "position": 6
    },
    {
      "token": "나오",
      "start_offset": 14,
      "end_offset": 16,
      "type": "word",
      "position": 7
    },
    {
      "token": "냐",
      "start_offset": 16,
      "end_offset": 17,
      "type": "word",
      "position": 8
    }
  ]
}
```

### 어떻게 나누는지 확인해보기

> Postgresql의 EXPLAIN ANALYZE 처럼 explain을 true로 설정하면 더 자세한 정보를 확인할 수 있어요.

Request Body:

```json
{
  "tokenizer": "nori_tokenizer",
  "text": [
    "장송의 프리렌 2기 언제 나오냐?"
  ],
  "explain": true  // 옵션 추가
}
```

응답은 길어서 생략하겠습니다.

### 그 외 다른 토크나이저들

- Keyword Tokenizer: 입력된 텍스트를 그대로 토큰으로 사용합니다.
- Whitespace Tokenizer: 공백을 기준으로 토큰을 나눕니다.
- Ngram Tokenizer: 입력된 텍스트를 n-gram으로 나눕니다.
- Edge Ngram Tokenizer: 입력된 텍스트를 edge n-gram으로 나눕니다.

Ngram의 경우 n의 범위를 지정할 수 있습니다.

Request Body:

```json
{
  "tokenizer": {
    "type": "ngram",
    "min_gram": 2,  // 최소 길이
    "max_gram": 3  // 최대 길이
  },
  "text": [
    "장송의 프리렌"
  ]
}
```

Response: 뭔가 공백도 들어가고 이상하게 나눠진 걸 볼 수 있어요. 자동 완성같은 기능에서 사용할 수 있을 것 같아요.

```json
{
  "tokens": [
    {
      "token": "장송",
      "start_offset": 0,
      "end_offset": 2,
      "type": "word",
      "position": 0
    },
    {
      "token": "장송의",
      "start_offset": 0,
      "end_offset": 3,
      "type": "word",
      "position": 1
    },
    {
      "token": "송의",
      "start_offset": 1,
      "end_offset": 3,
      "type": "word",
      "position": 2
    },
    {
      "token": "송의 ",
      "start_offset": 1,
      "end_offset": 4,
      "type": "word",
      "position": 3
    },
    {
      "token": "의 ",
      "start_offset": 2,
      "end_offset": 4,
      "type": "word",
      "position": 4
    },
    {
      "token": "의 프",
      "start_offset": 2,
      "end_offset": 5,
      "type": "word",
      "position": 5
    },
    {
      "token": " 프",
      "start_offset": 3,
      "end_offset": 5,
      "type": "word",
      "position": 6
    },
    {
      "token": " 프리",
      "start_offset": 3,
      "end_offset": 6,
      "type": "word",
      "position": 7
    },
    {
      "token": "프리",
      "start_offset": 4,
      "end_offset": 6,
      "type": "word",
      "position": 8
    },
    {
      "token": "프리렌",
      "start_offset": 4,
      "end_offset": 7,
      "type": "word",
      "position": 9
    },
    {
      "token": "리렌",
      "start_offset": 5,
      "end_offset": 7,
      "type": "word",
      "position": 10
    }
  ]
}
```

Ngram의 대안으로 Edge Ngram이 있습니다.

Request Body:

```json
{
  "tokenizer": {
    "type": "edge_ngram",
    "min_gram": 2,
    "max_gram": 3
  },
  "text": [
    "장송의 프리렌"
  ]
}
```

Response: 시작 부분만 나누기 때문에 조금 더 효율적으로 자동 완성 기능을 구현할 수 있을 것 같아요.

```json
{
  "tokens": [
    {
      "token": "장송",
      "start_offset": 0,
      "end_offset": 2,
      "type": "word",
      "position": 0
    },
    {
      "token": "장송의",
      "start_offset": 0,
      "end_offset": 3,
      "type": "word",
      "position": 1
    }
  ]
}
```

## 3. 토큰 필터(Token Filter)

- 마지막 단계입니다!
- 생성된 토큰을 추가로 처리합니다. 예를 들어 불용어 제거, 어근 추출 등이 있고
- 영어의 경우 대소문자를 통일하는 작업도 이 단계에서 수행합니다.
- 앞에서 언급한 Tokenizer Ngram, Edge Ngram도 이 단계에서 처리할 수 있습니다.

### 1. 불용어 제거

Request Body: 영어는 따로 stopword를 지정할 필요가 없지만, 한글의 경우 불용어를 지정해줘야 합니다.

```json
{
  "tokenizer": "nori_tokenizer",
  "filter": [
    {
      "type": "stop",
      "stopwords": ["은", "는", "이", "가"]
    }
  ],
  "text": [
    "여의도는 맛집이 없어"
  ]
}
```

Response: 불용어가 제거된 걸 확인할 수 있어요.

```json
{
  "tokens": [
    {
      "token": "여의도",
      "start_offset": 0,
      "end_offset": 3,
      "type": "word",
      "position": 0
    },
    {
      "token": "맛",
      "start_offset": 5,
      "end_offset": 6,
      "type": "word",
      "position": 2
    },
    {
      "token": "집",
      "start_offset": 6,
      "end_offset": 7,
      "type": "word",
      "position": 3
    },
    {
      "token": "없",
      "start_offset": 9,
      "end_offset": 10,
      "type": "word",
      "position": 5
    },
    {
      "token": "어",
      "start_offset": 10,
      "end_offset": 11,
      "type": "word",
      "position": 6
    }
  ]
}
```

### 2. 검색 일관성 유지

Request Body: 영어의 경우 대소문자를 통일하는 작업을 수행할 수 있어요.

```bash
{
  "tokenizer": "standard",
  "filter": [
    {
      "type": "lowercase"
    }
  ],
  "text": [
    "Elasticsearch"
  ]
}
```

Response: 대문자가 소문자로 변환된 걸 확인할 수 있지요.

```json
{
  "tokens": [
    {
      "token": "elasticsearch",
      "start_offset": 0,
      "end_offset": 13,
      "type": "<ALPHANUM>",
      "position": 0
    }
  ]
}
```

### 3. 검색 범위 확장

- Stemmer를 사용하면 단어의 어근을 추출할 수 있어요. (근데 한글은 안됩니다...ㅠ)

- 예를 들어 **"running"** 을 **"run"** 으로 변환하거나, **"crawling"** 를 **"crawl"** 으로 변환할 수 있습니다.

Request Body:

```json
{
  "tokenizer": "standard",
  "filter": [
    {
      "type": "stemmer"
    }
  ],
  "text": [
    "running crawling"
  ]
}
```

Response: 어근이 추출된 걸 확인할 수 있어요.

```json
{
  "tokens": [
    {
      "token": "run",
      "start_offset": 0,
      "end_offset": 7,
      "type": "<ALPHANUM>",
      "position": 0
    },
    {
      "token": "crawl",
      "start_offset": 8,
      "end_offset": 16,
      "type": "<ALPHANUM>",
      "position": 1
    }
  ]
}
```

- 또한 Synonym Filter를 사용하면 동의어를 동일한 의미로 처리할 수도 있어요.

Request Body:

```json
{
  "tokenizer": "standard",
  "filter": [
    {
      "type": "synonym",
      "synonyms": ["민초, nice"]
    }
  ],
  "text": [
    "nice food"
  ]
}
```

Response: 이렇게 검색 조작을 할 수 있죠 ㅋㅋ

```json
{
  "tokens": [
    {
      "token": "nice",
      "start_offset": 0,
      "end_offset": 4,
      "type": "<ALPHANUM>",
      "position": 0
    },
    {
      "token": "민초",
      "start_offset": 0,
      "end_offset": 4,
      "type": "SYNONYM",
      "position": 0
    },
    {
      "token": "food",
      "start_offset": 5,
      "end_offset": 9,
      "type": "<ALPHANUM>",
      "position": 1
    }
  ]
}
```

그 외에도 다양한 Token Filter가 있습니다.

- **Ascii Folding**: ASCII 문자를 ASCII 문자로 변환합니다.
    - 입력: "Café"
    - 출력: "Cafe"
- **Ngram**: 입력된 텍스트를 n-gram으로 나눕니다.
- **Edge Ngram**: 입력된 텍스트를 edge n-gram으로 나눕니다.
- **Shingle**: 입력된 텍스트를 shingle로 나눕니다.
    - 입력: "장송의 프리렌 재밌다", 
        - min_shingle_size: 2, 
        - max_shingle_size: 3
    - 출력: ["장송의", "장송의 프리렌", "프리렌", "프리렌 재밌다", "재밌다"]
- **Truncate**: 입력된 텍스트를 잘라냅니다.
    - 입력: "엘라스틱서치", 
        - length: 3
    - 출력: ["엘라스"]
- **Unique**: 중복된 토큰을 제거합니다.
    - 입력: "장송의 프리렌 프리렌"
    - 출력: ["장송의", "프리렌"]

# 마치며

그냥 wildcard + fuzziness 써도 좋지만, 어근 추출이나 동의어 처리 등 검색 경험을 향상시키는데 도움이 되는 다양한 기능을 제공하는 것 같습니다.

또한 성능을 향상시키기 위해 Analyzer를 적절하게 사용하는 것도 중요하다고 생각했구요.

다음에 검색 프로젝트를 진행할 때는 이런 Analyzer를 적절하게 사용해보고 싶습니다.
