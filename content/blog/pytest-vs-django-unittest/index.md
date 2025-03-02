+++
title="Pytest vs Django unittest: 무엇을 선택해야 할까?"
date=2025-03-02
updated=2025-03-02
description="Django 프로젝트에서 unittest와 pytest 중 어떤 것을 선택할지 고민된다면 이 글을 읽어보셔요!"

[taxonomies]
tags=["Django", "pytest", "테스트", "Python"]

[extra]
giscus = true
quick_navigation_buttons = true
toc=true
social_media_card="preview.jpg"
+++

# 들어가며

안녕하세요! 이 글을 읽고 계신다면 Django에서 테스트를 할 때 Django 내장 프레임워크인 unittest와 파이썬 생태계에서 널리 사용되는 pytest 중 어떤 것을 선택할지 고민하고 계실 것 같은데요,

저도 업무나 개인 프로젝트를 하면서 이 둘의 차이를 크게 체감한 경험이 있어서 이 포스트를 작성하게 되었습니다. 두 프레임워크의 장단점을 비교해보고, 여러분의 프로젝트에 어떤 것이 더 적합할지 판단하는 데 도움이 되었으면 좋겠네요!

그럼 시작해보겠습니다.

# Django unittest

Django의 내장 테스트 프레임워크는 Python의 unittest 모듈을 확장한 것으로, Django 앱에 더 특화된 기능들을 제공합니다.

가장 큰 장점은 Django 프로젝트에 기본적으로 포함되어 있어서 추가 설치가 필요 없다는 겁니다!

## 자체 도구 제공

Django unittest는 `TestCase`, `Client`, `RequestFactory`와 같은 테스트를 위한 아주 편한 클래스들을 제공합니다.

### TestCase

```python
from django.test import TestCase
from myapp.models import MyModel

class MyModelTestCase(TestCase):
    def setUp(self):
        # 각 테스트 전에 실행됨 (테스트 데이터 준비하는 곳)
        MyModel.objects.create(name="Test Item", value=10)
    
    def test_my_model_creation(self):
        item = MyModel.objects.get(name="Test Item")
        self.assertEqual(item.value, 10)
        
    # 각 테스트는 독립적인 트랜잭션으로 실행되어 DB 상태가 격리됨
```

### Client

테스트 클라이언트는 HTTP 요청을 시뮬레이션해서 View를 쉽게 테스트할 수 있게 해줍니다.

```python
from django.test import TestCase, Client
from django.urls import reverse

class ViewTestCase(TestCase):
    def setUp(self):
        self.client = Client()

    def test_홈_조회(self):
        response = self.client.get(reverse('home'))
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, "홈입니다.")

    def test_포스트_생성(self):
        response = self.client.post(
            reverse('create_post'),
            {'title': '영역전개', 'content': '무량공처'},
            follow=True
        )
        self.assertEqual(response.status_code, 200)
```

### RequestFactory

`RequestFactory`는 미들웨어를 거치지 않고 직접 View 로직만 격리해서 테스트할 수 있어요.

`Client`보다 가볍고 빠르기 때문에, 단위 테스트에 집중하고 싶을 때 좋은 선택입니다.

다만, 쿠키나 세션 같은 기능을 테스트하려면 보통 미들웨어를 거쳐야 하기 때문에 그런 경우엔 `Client`를 쓰는 게 낫습니다.

```python
from django.test import TestCase, RequestFactory
from django.contrib.auth.models import User
from myapp.views import my_view

class RequestFactoryTestCase(TestCase):
    def setUp(self):
        self.factory = RequestFactory()
        self.user = User.objects.create_user(
            username='testuser', 
            password='password',
        )

    def test_my_view(self):
        request = self.factory.get('/my-url/')

        # 인증이 필요한 경우...
        request.user = self.user

        # View를 직접 호출하는 식
        response = my_view(request)
        self.assertEqual(response.status_code, 200)
```

## Fixture 지원

Django는 json, xml, yaml 등의 형식으로 픽스처를 지원해요.

(스포) 아래에서 Factory Boy와 함께 더 자세히 설명하겠지만, 개인적으로는 pytest의 Fixture가 사용성이 더 좋다고 느꼈습니다.

```json
[
    {
        "model": "myapp.product", 
        "pk": 1, 
        "fields": {
            "name": "프리렌 피규어", 
            "price": 210000
        }
    }
]
```

```python
from django.test import TestCase
from myapp.models import Product

class ProductTestCase(TestCase):
    fixtures = ['products.json']
    
    def test_fixture_loaded(self):
        product = Product.objects.get(pk=1)
        self.assertEqual(product.name, "프리렌 피규어")
```

## 데이터베이스 지원

Django unittest의 가장 강력한 기능 중 하나는 각 테스트마다 별도의 DB 트랜잭션을 생성해서 테스트 격리를 자동으로 보장해준다는 점이에요.

테스트 실행 전에 자동으로 데이터베이스를 세팅하고, 테스트 후에는 변경 사항을 롤백하기 때문에 테스트 간에 데이터가 섞이지 않아요.

```python
from django.test import TestCase
from myapp.models import Product

class ProductTestCase(TestCase):
    def setUp(self):
        # 각 테스트 메서드마다 실행됨
        self.product = Product.objects.create(
            name="프리렌 피규어", 
            price=210000,
        )
    
    def test_product_creation(self):
        # 데이터베이스 쿼리 실행
        product = Product.objects.get(name="프리렌 피규어")
        self.assertEqual(product.price, 210000)
    
    def test_product_update(self):
        # 다른 테스트에 영향을 주지 않고 DB 변경 가능
        self.product.price = 19000
        self.product.save()
        updated = Product.objects.get(id=self.product.id)
        self.assertEqual(updated.price, 19000)
```

같은 기능을 pytest로 구현하면 이렇게 됩니다.

```python
import pytest
from myapp.models import Product

@pytest.fixture
def product(db):  # db 픽스처는 데이터베이스에 접근한다는 의미
    return Product.objects.create(
        name="프리렌 피규어", 
        price=210000,
    )

def test_product_creation(db):
    product = Product.objects.create(
        name="프리렌 피규어", 
        price=210000,
    )
    assert Product.objects.filter(name="프리렌 피규어").exists()

def test_product_update(product):  # product 픽스처 자동 주입
    product.price = 79210000
    product.save()
    updated = Product.objects.get(id=product.id)
    assert updated.price == 79210000  # 수정된 값 확인
```

## 문서화가 잘 되어 있어요

Django 공식 문서에는 테스트에 관한 내용이 잘 정리되어 있어요. Django 프레임워크에 특화된 테스트 방법을 되게 상세하게 설명해줘서 처음이라면 추천합니다. (아쉽게도 한글 번역은 안댑니다..)

- [Testing in Django](https://docs.djangoproject.com/en/5.1/topics/testing/)

## 단점 (한계..?)

1. **클래스 기반 구조**: 모든 테스트가 클래스 안에 정의되어야 해서 코드가 조금 장황해질 수 있어요. 하지만 구조화가 잘 되어 있어서 체계적으로 느껴질 수도 있으니, 이건 개인 취향에 따라 장점이 될 수도 있을 것 같습니다.

2. **설정 복잡성**: 테스트 환경 설정을 위해 `setUp`과 `tearDown` 메서드를 사용해야 하는데, 복잡한 테스트에서는 `setUp` 메서드가 뚱뚱해져서 관리하기 어려울 수 있습니다...

3. **테스트 속도**: TestCase는 각 케이스마다 DB 트랜잭션을 사용하기 때문에 테스트 수가 많아지면 실행 속도가 느려질 수 있습니다.

# pytest + pytest-django

pytest는 [pytest-django](https://pytest-django.readthedocs.io/en/latest/) 플러그인을 통해 Django 테스트에 적용할 수 있습니다.

pytest는 Python 생태계에서 많이 사용되는 프레임워크로, 문법이 간결하고 기능도 훨씬 다양합니다. 물론 pytest와 pytest-django를 별도로 설치해야 합니다.

```bash
pip install pytest pytest-django
```

## Fixture 기능이 정말 좋아요

pytest의 가장 큰 장점 중 하나는 Fixture 기능입니다. fixture를 모듈화할 수 있어서 테스트 설정을 재사용하기 아주 좋읍니다... ☺️

```python
import pytest
from django.contrib.auth.models import User

@pytest.fixture
def test_user():
    user = User.objects.create_user(
        username="kwanok",
        email="kwanok@example.com",
        password="password123"
    )
    return user

@pytest.fixture
def authenticated_client(client, test_user):
    client.login(username="kwanok", password="password123")
    return client

def test_profile_access(authenticated_client):
    response = authenticated_client.get("/profile/")
    assert response.status_code == 200

def test_profile_edit(authenticated_client):
    response = authenticated_client.post("/profile/edit/", {"bio": "나야.."})
    assert response.status_code == 302
```

같은 기능을 Django unittest로 구현하면 아래처럼 될 수 있어요.

```python
from django.test import TestCase
from django.contrib.auth.models import User

class UserProfileTest(TestCase):
    def setUp(self):
        self.user = User.objects.create_user(
            username='kwanok',
            email='kwanok@example.com',
            password='password123'
        )
        self.client.login(username='kwanok', password='password123')
    
    def test_profile_access(self):
        response = self.client.get('/profile/')
        self.assertEqual(response.status_code, 200)
    
    def test_profile_edit(self):
        response = self.client.post('/profile/edit/', {'bio': '나야..'})
        self.assertEqual(response.status_code, 302)
```

여기서 보이는 것처럼 작은 예제에서는 큰 차이가 없지만, 테스트 케이스가 많아지거나 복잡해질수록 pytest의 fixture 시스템이 코드 중복을 줄이고 관리하기 편해져요.

## pytest fixture + Factory Boy

pytest와 Factory Boy, 그리고 pytest-factoryboy 플러그인을 함께 사용하면 테스트 데이터 생성이 정말 편리해집니다. 개발 커뮤니티에서도 이 조합을 많이 선호하는 것 같아요.

([Django Testing Just Got So Much Easier](https://www.reddit.com/r/django/comments/vmlxft/django_testing_just_got_so_much_easier/))

```python
# conftest.py (pytest 설정 파일)
import pytest
from pytest_factoryboy import register
from .factories import ProductFactory, CategoryFactory

# 팩토리 등록
register(CategoryFactory)
register(ProductFactory)

@pytest.fixture
def premium_product(product_factory):
    return product_factory(name="프리미엄 상품", price=5000000.00)

# factories.py (팩토리 정의)
import factory
from myapp.models import Category, Product

class CategoryFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Category
    
    name = factory.Sequence(lambda n: f'카테고리 {n}')

class ProductFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Product
    
    name = factory.Sequence(lambda n: f'상품 {n}')
    price = factory.Faker(
        'pydecimal', 
        left_digits=10, 
        right_digits=2, 
        positive=True,
    )
    description = factory.Faker('paragraph')
    category = factory.SubFactory(CategoryFactory)

# test_product.py (테스트 파일)
def test_factory_default(product):  # 자동 생성된 기본 제품
    assert product.name.startswith("상품")

def test_factory_custom(product_factory):  # 팩토리 직접 사용
    custom = product_factory(name="커스텀 상품")
    assert custom.name == "커스텀 상품"

def test_premium(premium_product):  # 직접 정의한 픽스처 사용
    assert premium_product.price == 5000000.00
```

## 풍부한 플러그인 생태계

pytest의 또 다른 큰 장점은 다양한 플러그인을 통해 기능을 확장할 수 있다는 점이에요.

- [pytest-cov](https://pytest-cov.readthedocs.io/en/latest/)
    - 코드 커버리지를 측정해주는 플러그인
    ```bash
    pytest --cov=myapp
    ```

- [pytest-mock](https://pytest-mock.readthedocs.io/en/latest/)
    - 손쉬운 모킹 기능을 제공
    ```python
    def test_payment_processed(mocker):
        mock_gateway = mocker.patch('myapp.services.PaymentGateway')
        mock_gateway.return_value.process.return_value = True
        service = PaymentService()
        result = service.process_payment(100)
        assert result is True
    ```

- [pytest-benchmark](https://pytest-benchmark.readthedocs.io/en/latest/)
    - 코드 성능을 측정할 수 있는 훌륭한 도구
    - 제가 정말 좋아하는 기능인데 django test에는 없더라고요
    ```bash
    pytest --benchmark-autosave <파일 이름>
    ```

## 매개변수화된 테스트 (Parametrized Test)

여러 테스트 케이스를 한 번에 실행하고 싶을 때 `parametrize` 데코레이터를 사용하면 코드가 훨씬 간결해집니다.

```python
import pytest
from myapp.utils import validate_username

@pytest.mark.parametrize("username,expected", [
    ('user123', True),      # 유효한 사용자명
    ('usr', False),         # 너무 짧음
    ('a' * 31, False),      # 너무 김
    ('user@123', False),    # 유효하지 않은 문자
])
def test_username_validation(username, expected):
    assert validate_username(username) == expected
```

이 기능은 Django unittest에서는 직접 구현해야 하므로, pytest를 사용하면 코드량을 크게 줄일 수 있어요.

## 병렬 실행으로 속도 향상

`pytest-xdist` 플러그인을 사용하면 테스트를 여러 프로세스에서 병렬로 실행할 수 있어요. CI/CD 파이프라인의 실행 속도를 크게 개선할 수 있기 때문에 개발 생산성 향상에 도움이 됩니다!

```bash
pip install pytest-xdist
pytest -n 4  # 4개의 CPU 코어 사용
```

# Django unittest에서 pytest로 마이그레이션하고 싶다면?

기존 Django unittest 코드를 pytest로 마이그레이션하는 것은 점진적으로 하는 것이 좋아요. pytest는 unittest와의 호환성을 제공하기 때문에, 기존 테스트는 그대로 두고 새로운 테스트만 pytest로 작성하는 전략이 가능합니다.

먼저 설정 파일을 생성해봅시다:

```ini
# pytest.ini
[pytest]
DJANGO_SETTINGS_MODULE = yourproject.settings
python_files = test_*.py *_test.py
python_classes = Test*
python_functions = test_*
```

Django unittest에서 pytest로 변환할 때 주요 변경점은 다음과 같습니다:

- `TestCase` → `@pytest.mark.django_db` 데코레이터로 변경
- 클래스 기반 → 함수 기반 테스트로 변환
- `setUp`, `tearDown` → fixture로 변환

### 변환 전 (Django unittest)
```python
from django.test import TestCase
from django.contrib.auth.models import User

class UserTests(TestCase):
    def setUp(self):
        self.user = User.objects.create(username='testuser')
        
    def test_user_creation(self):
        self.assertEqual(self.user.username, 'testuser')
```

### 변환 후 (pytest)
```python
import pytest
from django.contrib.auth.models import User

@pytest.fixture
def test_user():
    return User.objects.create(username='testuser')

@pytest.mark.django_db
def test_user_creation(test_user):
    assert test_user.username == 'testuser'
```

# 결론

Django unittest와 pytest 모두 웬만한 프로젝트를 테스트하는 데 충분한 도구입니다. 그리고 처음이라면 어떤 것을 선택할지는 프로젝트의 규모, 팀의 경험, 그리고 특정 요구사항에 따라 달라질 수 있습니다.

두 프레임워크를 모두 사용해 본 경험을 바탕으로 개인적인 견해를 표로 정리해봤어요:

| 비교 항목 | Django unittest | pytest |
|-|-|-|
| 코드 간결성 | 중간 (클래스 기반) | 높음 (함수 기반) |
| 확장성 | 중간 | 아주 높음 (플러그인 생태계) |
| 실행 속도 | 중간~느림 | 빠름 (병렬 실행, 최적화 옵션) |
| 학습 곡선 | 낮음 (Django 개발자라면) | 중간 (새로운 개념 학습 필요) |
| 기능 | 적당함 | 풍부함 |
| 픽스처 관리 | 파일 기반 | 코드 기반 (더 유연함) |
| 프로젝트 적합성 | 소~중규모 | 중~대규모 |

개인적으로는 새 프로젝트를 시작한다면 pytest를 선택하는 편이지만, 이미 Django unittest로 많은 테스트를 작성한 프로젝트라면 전환 비용도 꼭 고려해야 합니다. (pytest가 좋으니까 무족권 바꿔! → 비추천입니다.)

```python
def 개인적인_테스트_프레임워크_추천():
    if pytest에_익숙하다:
        return 'pytest'

    if 급한_프로젝트다:
        return 'django-test'
    
    if 시간_많다:
        return 'pytest'

    if 지금_django_test_사용중이다:
        if 프로젝트_시작한_지_얼마_안_됐다:
            return 'pytest'

        return 'django-test'
```

여러분의 프로젝트에 맞는 테스트 프레임워크를 선택하시길 바랍니다 😘 끗~
