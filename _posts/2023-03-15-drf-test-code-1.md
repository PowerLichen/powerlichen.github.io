---
title : DRF에서 테스트 코드 작성하기(1) - DRF의 테스트 코드
date : 2023-03-15 13:45:00 +0900
categories : [Backend, Django]
tags : [drf, django]
img_path: /assets/img/2023-03-15/
---

# 개요
테스트 코드.
테스트에 대한 중요성은 굳이 말 하지 않아도 잘 알수 있다.
내가 개발한 코드가 의도된 대로 작성되었는지 검증하는 것은 무엇보다 중요하다. 
테스트 주도 개발(TDD) 라는 개발 방법론까지 있는 데다가, 기업의 과제 테스트나 면접 등에서 테스트에 관한 이야기는 빼놓을 수 없을 정도다.

Django에서도 이런 테스트 코드를 잘 작성할 수 있게 지원해주고 있다.

# Django에서의 테스트 방법
[공식 문서](https://docs.djangoproject.com/en/4.1/topics/testing/overview/)를 참고하면 테스트 코드를 실행하는 과정은 다음과 같다.

## 테스트 코드 작성
startapp으로 app을 만들었다면 tests.py 라는 파일이 존재할 것이다.
해당 파일을 사용해도 좋고, 다른 파일을 만들어도 좋다.
테스트 실행 시 모든 `test*.py` 를 검색하여 실행하기 때문에, 해당 양식에 맞추어 파일을 작성하기만 하면 된다.

공식문서에 나온 예시는 다음과 같다.
```python
# ./animals/tests.py
from django.test import TestCase
from myapp.models import Animal

class AnimalTestCase(TestCase):
    def setUp(self):
        Animal.objects.create(name="lion", sound="roar")
        Animal.objects.create(name="cat", sound="meow")

    def test_animals_can_speak(self):
        """Animals that can speak are correctly identified"""
        lion = Animal.objects.get(name="lion")
        cat = Animal.objects.get(name="cat")
        self.assertEqual(lion.speak(), 'The lion says "roar"')
        self.assertEqual(cat.speak(), 'The cat says "meow"')
```

## 테스트 실행 

테스트 코드 실행 방법은 다음과 같다.

```shell
# 모든 테스트를 찾아 실행하기
python manage.py test

# `animals` 디렉토리 내의 모든 테스트 실행
python manage.py test animals

# 하나의 테스트 케이스(클래스)만 실행
python manage.py test animals.tests.AnimalTestCase

# 하나의 테스트 메소드만 실행
python manage.py test animals.tests.AnimalTestCase.test_animals_can_speak

```

# DRF에서의 테스트 방법
DRF에서 테스트 코드를 작성하고 실행하는 과정은 동일하다.
물론, DRF에는 그에 맞는 테스트 코드를 작성하는 방법이 있으니 그 방법을 사용하는 것이 더 바람직하다.

DRF에서는 Django test case 클래스인 `TestCase`를 상속한 `APITestCase`가 있다.

`APITestCase`는 `client_class`를 DRF의 `APIClient`로 재정의 하고 있으며, `login`이나 `credentials`와 같은 추가 메소드를 제공한다.

위 사항들을 고려하여, 공지사항을 등록하는 `notice`라는 app의 테스트를 만들어보자.

## 디렉토리 구조
```
├── project
│   ├── ...
│   └── settings.py
├── notice
│   ├── tests
│   │   ├── __init__.py
│   │   ├── test_notice_create.py
│   │   ├── test_notice_update.py
│   │   └── ...
│   ├── __init__.py
│   └── ...
...
```
디렉토리 구조는 다음과 같이 설정하였다.
`notice` 앱의 경우, CRUD가 전부 포함되어 있다.
tests.py 파일 하나만 사용할 경우 파일 길이가 너무 길어지고 한눈에 찾아보기 어렵다.

## setUpTestData와 setUp
디렉토리를 구성했다면 다음으로는 테스트에 사용될 데이터를 선언하는 단계이다.
여기서 두 가지의 데이터 선언 방법이 있다.

```python
# test_notice_update.py
from rest_framework.test import APITestCase


class NoticeUpdateTestCase(APITestCase):
    @classmethod
    def setUpTestData(cls):
        # 전체 테스트 케이스에서 사용할 데이터 설정
        ...

    def setUp(self):
        # 테스트 케이스가 실행될 때 마다 전처리할 데이터 설정
        ...
```

setUpTestData()
- 클래스 수준에서 실행.
- 클래스 단위로 처음 한번만 실행된다.

setUp()
- 테스트 메소드가 실행될 때 마다 실행.

두 메소드에는 위와 같은 차이가 있다.
그렇다면 여기서 시작할 때 한번만 초기화를 진행해도 될 경우에는 `setUpTestData`를, 여러번 초기화가 필요한 데이터는 `setUp`을 사용하면 될 것이라 예상할 수 있다.

기본적으로 테스트에서 필요로 하는 데이터는 다음과 같다.
- `client`: API method를 통해 테스트를 할 클라이언트
- `url`: 테스트 할 url
- `미리 선언할 데이터`: update나 delete 테스트를 위해 선언해 둘 데이터
- `json 테스트 데이터`: create, update 등에서 사용될 데이터

여기서 `client`, `url`, `미리 선언할 데이터`는 초기에 한번만 실행해도 되므로 `setUpTestData()`가 적합하다고 할 수 있다.

하지만 `json 테스트 데이터`는 메소드 마다 초기화하여 데이터를 `pop`하는 방식으로 성공, 실패 테스트를 해볼 수 있으니 `setUp()`에 적합하다고 할 수 있다.

### setUpTestData(cls)
`setUpTestData`는 클래스 메소드로 self 대신 cls를 사용한다.
해당 메소드에서는 다음과 같은 데이터를 선언하였다.
- 클라이언트 및 클라이언트에 로그인 처리
- 미리 선언할 데이터
- url

```python
from rest_framework.test import APIClient

cls.client = APIClient()
```
우선 사용할 클라이언트를 선언한다.

```python
from django.contrib.auth.models import User
...
user = User.objects.create_user(
    username="hello",
    password="12345678"
)

# SessionAuthentication을 사용할 경우
cls.client.login(
    username="hello",
    password="12345678"
)

# rest_framework_simplejwt를 사용할 경우
from rest_framework_simplejwt.tokens import RefreshToken

refresh = RefreshToken.for_user(user=user)
access = refresh.access_token
cls.client.credentials(
    HTTP_AUTHORIZATION=f"Bearer {access}"
)
```
사용자를 생성하고, 이를 기반으로 클라이언트에 로그인해 주었다.
세션 로그인, 토큰 로그인에 따라 다른 방법을 사용한다.

```python
cls.notice = Notice.objects.create(
    user=user
    title="test title",
    context="test context"
)
```
추후 테스트에 사용될 notice를 미리 선언한다.
작성자는 위에서 생성한 user를 사용하였다.

```python
cls.url = f"/notice/{cls.notice.id}/"
```
마지막으로 테스트에서 사용할 url를 선언하였다.

### setUp(self)
해당 부분에서는 메소드를 실행할 때 전달할 데이터를 정의한다.
예시에서는 update를 예시로 들었기 때문에, 그에 대한 데이터를 선언하였다.
```python
self.data = {
    "title": "Modified title",
    "context": "Modified context"
}
```


### 전체 코드
simple jwt를 사용하였다고 가정.
```python
# test_notice_update.py
from django.contrib.auth.models import User
from rest_framework.test import APIClient
from rest_framework.test import APITestCase
from rest_framework_simplejwt.tokens import RefreshToken


class NoticeUpdateTestCase(APITestCase):
    @classmethod
    def setUpTestData(cls):
        cls.client = APIClient()
        user = User.objects.create_user(
            username="hello",
            password="12345678"
        )
        refresh = RefreshToken.for_user(user=user)
        access = refresh.access_token
        cls.client.credentials(
            HTTP_AUTHORIZATION=f"Bearer {access}"
        )
        
        # 유저 생성 및 클라이언트에 auth 설정
        cls.notice = Notice.objects.create(
            user=user
            title="test title",
            context="test context"
        )
        cls.url = f"/notice/{cls.notice.id}/"

    def setUp(self):
        self.data = {
            "title": "Modified title",
            "context": "Modified context"
        }
```

## 테스트 메소드 작성
데이터를 선언해두었으니 이제 테스트 케이스를 작성하는 단계만 남았다.

간단하게 테스트 케이스를 고려하면 다음과 같은 것을 생각해 볼 수 있다.
- 전체 update
- 일부 update

물론 이 외에도 update 권한 체크나, url의 id 범위 벗어남, 잘못된 패러미터 등 많은 케이스가 존재한다.
하지만 여기서는 위의 두 케이스만 고려하였다.

테스트에서는 APIClient의 HTTP method를 호출하고, response를 통해 데이터를 확인할 수 있다.
일반적으로 `self.assertEqual()`를 자주 사용한다.
이외의 메소드는 [unittest 문서](https://docs.python.org/ko/3/library/unittest.html#assert-methods)와 django test 문서의 [assertions 파트](https://docs.djangoproject.com/en/4.1/topics/testing/tools/#assertions)를 통해 확인할 수 있다.

### 전체 update
전체 테스트의 경우에는 미리 데이터를 선언해 두었으므로 간단하다.
```python
    def test_notice_update_success(self):
        response  = self.client.patch(
            self.url,
            data=self.data
        )
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(response.data["title"], self.data["title"])
        self.assertEqual(response.data["context"], self.data["context"])
```

### 일부 update
일부 수정 테스트에서는 `setUp`에서 선언된 데이터를 수정하여 사용한다.
```python
    def test_notice_update_with_missing_title(self):
        self.data.pop("title")
        response  = self.client.patch(
            self.url,
            data=self.data
        )
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(response.data["title"], self.notice.title)
        self.assertEqual(response.data["context"], self.data["context"])
        
    def test_notice_update_with_missing_context(self):
        self.data.pop("context")
        response  = self.client.patch(
            self.url,
            data=self.data
        )
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(response.data["title"], self.data["title"])
        self.assertEqual(response.data["context"], self.notice.context)
```