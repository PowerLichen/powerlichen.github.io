---
title : DRF에서 테스트 코드 작성하기(3) - factory boy
date : 2023-03-19 19:23:00 +0900
categories : [Backend, Django]
tags : [drf, django]
img_path: /assets/img/2023-03-19/
---

# 개요
[factory boy](https://factoryboy.readthedocs.io/en/stable/)는 테스트 시 사용해야 할 중복된 코드들을 관리하기 위한 패키지다.

앞서 사용한 테스트 코드에서는 다음과 같은 코드가 있었다.

여기서 유저 데이터나 로그인 정보와 같은 데이터를 선언하는데 임의의 값을 일일히 지정해야 하는 문제점이 있다.

```python
...
    @classmethod
    def setUpTestData(cls):
        cls.client = APIClient()
        user_data = {
            "username": "hello",
            "password": "12345678"
        }
        user = User.objects.create_user(**user_data)
        cls.client.login(**user_data)

        cls.notice = Notice.objects.create(
            user=user,
            title="test title",
            context="test context"
        )
        cls.url = reverse('notice-detail', kwargs={"pk": cls.notice.id})
...
```

이를 간단한 선언문으로 관리하고, 자동으로 임의의 값을 할당하는 걸 지원하는 패키지가 바로 factory_boy다.

# factory_boy
## factory 정의
팩토리를 정의할때는 먼저 파이썬 객체를 인스턴스화하기 위해 사용되는 변수를 선언한다.

여기서 어떤 객체를 팩토리화 할 것인지 `class Meta:`의 `model`로 객체를 정의한다.

예시에서는 파이썬의 기본 유저 모델과 필수 데이터인 username, password를 정의하였다.

```python
import factory
from django.contrib.auth import get_user_model

UserModel = get_user_model()


class UserFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = UserModel
        django_get_or_create = ("username", )

    username = factory.Faker("user_name")
    password = factory.Faker("password")

```
`django_get_or_create`는 `DjangoModelFactory`에서 지원하는 옵션이다.  
생성 작업 시 django의 `create`가 아닌 `get_or_create`를 사용하여 중복된 값 생성시 생길 수 있는 문제를 예방할 수 있다.

여기서 factory.Faker는 무작위 값을 할당하는 기능을 한다.

## factory.Faker
factory_boy에서는 가짜 데이터 생성하는 패키지인 [Faker](https://faker.readthedocs.io/en/latest/)를 지원한다.

`factory.Faker('ean', length=10)` 코드는
`faker.Faker.ean(length=10)`을 호출한다.

Faker의 첫번째 패러미터인 provider로 선언가능한 값은 Faker의 [Standard Providers](https://faker.readthedocs.io/en/latest/providers.html) 및 공식문서를 통해 확인할 수 있다.

예제에서 사용한 provider은 다음과 같다.
- user_name: 사용자 이름 provider.
- password: 비밀번호 provider.
`special_chars`, `digits`, `upper_case`, `lower_case` 옵션으로 생성시 포함될 값을 설정.
- text: 문자열 provider. 글자수를 제한하여 `title` 생성에 사용.
`max_nb_chars` 옵션에 따라 `words, sentence, paragraphs` provider를 사용하여 생성
- sentence: 문장 provider.

## factory 사용
객체를 선언하였다면 이제 test 코드에서 사용할 수 있다.

```python
cls.user = UserFactory()
cls.notice = NoticeFactory(user=cls.user)
```

여기서 패러미터로 주어지지 않은 값은 factory 객체에서 선언했던 `faker`를 통해 자동 할당된다.

위 코드의 NoticeFactory는 방금 생성한 user가 작성한 글임을 나타내기 위해 user를 명시적으로 지정해주었다.


# 예제 코드
```python
# factories.py
import factory
from django.contrib.auth import get_user_model

from api.testcode.models import Notice

UserModel = get_user_model()


class UserFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = UserModel
        django_get_or_create = ("username", )

    username = factory.Faker("user_name")
    password = factory.Faker("password")


class NoticeFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Notice
    
    user = factory.SubFactory(UserFactory)
    title = factory.Faker("sentence")
    context = factory.Faker("text", max_nb_chars=20)
```
User와 Notice에 대해 팩토리를 정의하였다.
```python
# test_notice_update.py
class NoticeUpdateTestCase(APITestCase):
    @classmethod
    def setUpTestData(cls):
        cls.client = APIClient()

        # Old setup data code
        # user_data = {
        #     "username": "hello",
        #     "password": "12345678"
        # }
        # user = User.objects.create_user(**user_data)
        # cls.client.login(**user_data)
        #
        # cls.notice = Notice.objects.create(
        #     user=user,
        #     title="test title",
        #     context="test context"
        # )
        #
        # cls.url = f"/api/testcode/notice/{cls.notice.id}/"

        user = UserFactory()
        cls.client.login(username=user.username, password=user.password)
        cls.notice = NoticeFactory(user=user)
        
        cls.url = reverse("notice-detail", kwargs={"pk": cls.notice.id})
```
팩토리를 사용함으로서 코드가 훨씬 간결해진 것을 볼 수 있다.  
추후 테스트에서 username, password, title, context와 같은 내용을 검증해야 할 경우에는 팩토리로 선언했던 값을 사용하면 된다.

예를 들어 `cls.notice = NoticeFactory(user=user)`를 통해서 자동으로 가짜 값이 생성된 title, context는 다음과 같이 사용할 수 있다.

```python
self.notice.title
# Three image son
self.notice.context
# Serious inside else memory if six field live on traditional.
```
