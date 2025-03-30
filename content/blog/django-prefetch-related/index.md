+++
title="Django의 prefetch_related: N+1을 잡자"
date=2025-03-30 23:30:00
updated=2025-03-30
description="Django ORM 최적화와 N+1 문제 해결을 위한 prefetch_related의 심층 분석"

[taxonomies]
tags=["Django", "Database"]

[extra]
giscus = true
quick_navigation_buttons = true
toc = true
social_media_card="preview.jpg"
+++

Django를 사용해서 개발할 때 N+1을 해결하기 위해 보통 select_related 또는 prefetch_related를 사용합니다.

prefetch_related는 **역방향 ForeignKey (1:N)** 그리고 **ManyToMany (N:M)** 의 경우 쓰면 좋은 솔루션입니다.

# Prefetch Related 란?

1. 메인 쿼리를 먼저 실행하여 메인 객체(리스트)를 가져옵니다.
2. 그리고 그 객체들을 별도의 쿼리로 가져옵니다.
3. **파이썬 메모리에**서 두 결과를 조인합니다

저는 3번을 강조하고 싶은데 `select_related`랑 다른 점이기 때문입니다.

`select_related`는 SQL 조인을 통해서 가져옵니다.

반면, `prefetch_related`는 별도의 쿼리를 실행해서 애플리케이션에서 결과를 조합합니다.

# 모델 예시

다음과 같이 모델이 정의돼있다고 가정해보겠습니다.

(제가 작명 센스가 없습니다... 이렇게 모델링 하면 리뷰가 많이 달릴수도...)

모델은 총 4개 입니다. `작가`, `태그`, `포스트`, `댓글` 입니다.

- 작가는 여러개의 포스트를 가질 수 있고,
- 포스트는 여러개의 댓글을 가져올 수 있습니다.
- 포스트와 댓글은 각각 태그들을 가질 수 있다고 가정했습니다.


```python
from django.db import models


class Author(models.Model):
    name = models.CharField(max_length=100)
    
    class Meta:
        db_table='authors'


class Tag(models.Model):
    name = models.CharField(max_length=100)
    
    class Meta:
        db_table='tags'


class Post(models.Model):
    author = models.ForeignKey(Author, on_delete=models.CASCADE, related_name='posts')
    title = models.CharField(max_length=100)
    content = models.TextField()
    tags = models.ManyToManyField(Tag)
    
    class Meta:
        db_table='posts'


class Comment(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name='comments')
    content = models.TextField()
    tags = models.ManyToManyField(Tag)

    class Meta:
        db_table='comments'
```

# 그냥 조회하기

만약, 포스트를 가져올 때 댓글, 태그, 작성자를 가져온다고 하면 다음과 같이 가져올 수 있습니다.

```python
posts = Post.objects.all()[:3]
for post in posts:
    tags = []
    for tag in post.tags.all():
        tags.append(tag)
    
    comments = []
    for comment in post.comments.all():
        comments.append(comment)

    print(f'게시물: {post}')
    print(f'태그: {tags}')
    print(f'댓글: {post}')
    print(f'작성자: {post.author}')
```

데이터를 미리 세팅해두고, 쿼리를 실행해보면 총 10개의 쿼리가 실행된 걸 알 수 있습니다.

1번은 게시물을 가져오고, 나머지 9번은 태그, 포스트, 작성자를 각각 3번씩 가져오기 때문이죠.

익숙한 N+1 문제입니다.

```
=== 쿼리 성능 분석 결과 ===

실행 시간: 0.0050초
쿼리 수: 10
평균 쿼리 시간: 0.0000초

=======================

Query 1:
SQL: SELECT "posts"."id", "posts"."author_id", "posts"."title", "posts"."content" FROM "posts" LIMIT 3

Query 2:
SQL: SELECT "tags"."id", "tags"."name" FROM "tags" INNER JOIN "posts_tags" ON ("tags"."id" = "posts_tags"."tag_id") WHERE "posts_tags"."post_id" = 1

Query 3:
SQL: SELECT "comments"."id", "comments"."post_id", "comments"."content" FROM "comments" WHERE "comments"."post_id" = 1

Query 4:
SQL: SELECT "authors"."id", "authors"."name" FROM "authors" WHERE "authors"."id" = 1 LIMIT 21

Query 5:
SQL: SELECT "tags"."id", "tags"."name" FROM "tags" INNER JOIN "posts_tags" ON ("tags"."id" = "posts_tags"."tag_id") WHERE "posts_tags"."post_id" = 2

Query 6:
SQL: SELECT "comments"."id", "comments"."post_id", "comments"."content" FROM "comments" WHERE "comments"."post_id" = 2

Query 7:
SQL: SELECT "authors"."id", "authors"."name" FROM "authors" WHERE "authors"."id" = 1 LIMIT 21

Query 8:
SQL: SELECT "tags"."id", "tags"."name" FROM "tags" INNER JOIN "posts_tags" ON ("tags"."id" = "posts_tags"."tag_id") WHERE "posts_tags"."post_id" = 3

Query 9:
SQL: SELECT "comments"."id", "comments"."post_id", "comments"."content" FROM "comments" WHERE "comments"."post_id" = 3

Query 10:
SQL: SELECT "authors"."id", "authors"."name" FROM "authors" WHERE "authors"."id" = 2 LIMIT 21

```

# Prefetch Related 로 조회하기

그리고 이렇게 여러번 호출되는 미리 불러오기 위해 prefetch_related를 사용하면, 10번 실행되는 것을 3번으로 줄일 수 있습니다.

```python
posts = (
    Post.objects
    .select_related('author')
    .prefetch_related('tags', 'comments')
    .all()
)[:3]

for post in posts:
    위와 동일 ...
```

```
=== 쿼리 성능 분석 결과 ===

실행 시간: 0.0028초
쿼리 수: 3
평균 쿼리 시간: 0.0003초

=======================

Query 1:
SQL: SELECT "posts"."id", "posts"."author_id", "posts"."title", "posts"."content", "authors"."id", "authors"."name" FROM "posts" INNER JOIN "authors" ON ("posts"."author_id" = "authors"."id") LIMIT 3

Query 2:
SQL: SELECT ("posts_tags"."post_id") AS "_prefetch_related_val_post_id", "tags"."id", "tags"."name" FROM "tags" INNER JOIN "posts_tags" ON ("tags"."id" = "posts_tags"."tag_id") WHERE "posts_tags"."post_id" IN (1, 2, 3)

Query 3:
SQL: SELECT "comments"."id", "comments"."post_id", "comments"."content" FROM "comments" WHERE "comments"."post_id" IN (1, 2, 3)
```

# 만약 2-Depth의 모델이라면?

예를 들어 작성자를 기준으로 작성자가 쓴 포스트에 달린 댓글을 모두 가져오고 싶은 경우가 있을 수 있죠.

그럴 땐 `__` 로 구분해서 관계를 명시해주면 됩니다.

이렇게 가져온 값은 `RelatedManager`로 가져오게 되고, Prefetch된 객체는 일반 쿼리셋처럼 다시 쿼리를 호출할 수 있습니다.

```python
authors = (
    Author
    .objects
    .prefetch_related("posts__comments")  # 여기서 comment도 같이 prefetch 됩니다.
    .all()
)[:3]

comment_count = 0

for author in authors:
    for post in author.posts.all():
        for comment in post.comments.all():
            comment_count += 1
            
print(comment_count)
```

```
=== 쿼리 성능 분석 결과 ===

실행 시간: 0.0037초
쿼리 수: 3
평균 쿼리 시간: 0.0007초

=======================

Query 1:
SQL: SELECT "authors"."id", "authors"."name" FROM "authors" LIMIT 3

Query 2:
SQL: SELECT "posts"."id", "posts"."author_id", "posts"."title", "posts"."content" FROM "posts" WHERE "posts"."author_id" IN (1, 2, 3)

Query 3:
SQL: SELECT "comments"."id", "comments"."post_id", "comments"."content" FROM "comments" WHERE "comments"."post_id" IN (1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12)
```

# 만약 Prefetch에 조건을 걸고 싶다면요?

예를 들어 특정 태그의 게시물과 댓글을 가져오고 싶다면 이렇게 할 수 있습니다.

먼저 `Prefetch` 를 가져옵니다.

```python
from django.db.models import Prefetch
```

그리고, lookup에는 관계를 나타내고, queryset에 적절한 조건을 넣어주면 작성자와 연결된 객체들은 해당 조건에 걸리는 것들만 조회됩니다.

```python
tag_id = 1

authors = Author.objects.prefetch_related(
    Prefetch(
        lookup="posts",
        queryset=Post.objects.filter(tags__id=tag_id).all(),
    ),
    Prefetch(
        lookup="comments",
        queryset=Comment.objects.filter(tags__id=tag_id).all(),
    )
).all()

나머지 위와 같음...
```

# RelatedManager로 가져오는 건 너무 불안해요. 실수로 N+1이 또 터지면 어떡하죠?

이는 `to_attr` 파라메터로 `RelatedManager`가 아닌 List에 바인딩해서 가져올 수 있습니다.

성향에 따라 성능이 중요하면 `to_attr`를 쓰면 좋을 것 같고, 생산성이 중요하면 `to_attr`은 안 쓰고 개발하는 게 더 좋은 것 같습니다.

`to_attr` 쓰자!파

- N+1 쿼리 방지가 강제됩니다. 실수로 .filter() 같은 걸 못하죠.
- 명시적인 의도를 표현할 수 있어요.

`to_attr` 쓰지말자!파

- 조직의 코드베이스 컨벤션이 아닌 경우...?
- 추가 쿼리가 필요한 경우 원래 relation에 접근해서 가져와야 합니다.

```python
tag_id = 1

authors = Author.objects.prefetch_related(
    Prefetch(
        lookup="posts",
        queryset=Post.objects.filter(tags__id=tag_id).all(),
        to_attr='prefetched_posts',  # 여기
    ),
    Prefetch(
        lookup="prefetched_posts__comments",
        queryset=Comment.objects.filter(tags__id=tag_id).all(),
        to_attr='prefetched_comments',  # 여기
    )
).all()

comment_count = 0

for author in authors:
    for post in author.prefetched_posts:  # RelatedManager 가 아니기 때문에 리스트 처럼 사용합니다. 위 예시 비교하면 .all() 이 없어요.
        for _ in post.prefetched_comments:
            comment_count += 1
            
print(comment_count)
```

# 여기에 캐시까지 더한다면?

Django 캐시까지 더한다면 DB 조회 없이도, 빠르게 응답이 가능합니다.

물론 캐시는 값이 변경되지 않을 수 있으니까 적절히 flush 하는 로직이 들어가야 합니다.

```python
from django.core.cache import cache

def get_featured_posts_with_cache():
    cache_key = 'featured_authors_with_relations'
    cached_authors = cache.get(cache_key)
    
    if cached_authors is None:
        authors = Author.objects.prefetch_related("posts__comments").all()[:20]
    
        comment_count = 0
        
        for author in authors:
            for post in author.posts.all():
                for comment in post.comments.all():
                    comment_count += 1

        cache.set(cache_key, authors, 60*60)
        return authors
    
    return cached_authors
```

# 마지막 속도 비교

N+1을 해결하면 많은 시간을 아낄 수 있습니다.

sqlite 환경이지만 속도 비교만 올리고 마무리하겠습니다.

```python
# 객체별 숫자

{
    'authors': 10000, 
    'tags': 100, 
    'posts': 29920, 
    'comments': 149843,
}
```

```
=== 쿼리 성능 분석 결과 ===

--- original ---
실행 시간: 1.8857초
쿼리 수: 9000
평균 쿼리 시간: 0.0000초

--- prefetched ---
실행 시간: 0.0708초
쿼리 수: 3
평균 쿼리 시간: 0.0007초

======================
```

# 출처

- https://docs.djangoproject.com/en/5.1/ref/models/querysets/
