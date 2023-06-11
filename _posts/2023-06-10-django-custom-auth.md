---
title : DRF에서 커스텀 로그인 & 사용자 모델 작성하기
date : 2023-06-10 17:30:00 +0900
categories : [Backend, Django]
tags : [drf, django]
---

# 개요
회원 기능을 만드는 데는 여러 방법을 사용할 수 있다.

`Django Rest Framework`에서는 이러한 인증(Authentication) 방법으로 여러가지를 지원한다.  
이 중에는 `JWT`를 쉽게 사용할 수 있는 서드파티 패키지인 `simplejwt`도 있다.

하지만 이번에는 패키지를 그대로 쓰는 것이 아닌, 인증 시스템의 흐름에 대해 알아보자.

이를 위해서 `DRF`의 `TokenAuthentication`을 살펴보고, 직접 로그인 API를 구현해보고자 한다.

# Token Authentication
## 기본 설정
Token 기반 인증은 DB에 토큰을 저장하는 테이블을 생성하고, 이를 기반으로 현재 사용자를 구별한다.  
우선 이를 생성하기 위해 다음과 같은 설정을 추가해야한다.

```python
# settings.py
...
INSTALLED_APPS = [
    ...
    'rest_framework.authtoken'
]
...
...
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.TokenAuthentication',
    ]
}
...
```

## API 엔드포인트
토큰 인증의 경우, 기본적인 엔드포인트로 다음과 같이 구현이 되어 있다.
```python
# rest_framework.authtoken.views
class ObtainAuthToken(APIView):
    ...
    serializer_class = AuthTokenSerializer
    ...
    def post(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        user = serializer.validated_data['user']
        token, created = Token.objects.get_or_create(user=user)
        return Response({'token': token.key})
    ...
```
해당 APIView는 body data로 `username`과 `password`를 받는다.
구조를 하나씩 설명하면 다음과 같다.

- request의 data를 직렬화 한다.
- valid 체크를 진행한다.
- 직렬화 된 객체의 `user` 객체를 가져온다.
- `user` 객체에 대한 토큰을 생성한다.
- 토큰 값을 반환한다.

여기서 직렬화 후에 어떻게 데이터가 추가되는가 생각할 수 있다.  
이는 여기서 사용한 `AuthTokenSerializer`의 validate 과정을 보면 알 수 있다.

- `is_valid` 메소드는 `run_validation` 메소드를 호출한다.  
- `run_validation` 메소드는 `validate` 메소드를 호출한다.

`AuthTokenSerializer`는 `validate` 메소드를 재정의하여, `user`이름의 객체를 추가하고 있다.

```python
class AuthTokenSerializer(...):
    ...
    def validate(self, attrs):
        ...
        user = authenticate(
            request=self.context.get('request'),
            username=username,
            password=password
        )
        ...
        attrs['user'] = user
        return attrs
```

그렇다면 이를 바탕으로 커스텀 API를 만들어보자.


# 로그인(Token 인증) 커스텀 API 만들기
## views.py 정의하기
우선 로그인 기능을 만들기에 앞서, API 구성을 생각해보자.  
사용자 관련 기능으로는 `회원가입`, `로그인`, `로그아웃` 등 많은 기능이 있을 것이다.  

```python
# views.py
class UserViewSet(
    mixins.CreateModelMixin,
    GenericViewSet,
):
    serializer_class = UserSerializer
    
    def get_serializer_class(self):
        if self.action == "create":
            return UserCreateSerializer
        if self.action == "login":
            return UserLoginSerializer
        return super().get_serializer_class()
    
    
    @action(detail=False, methods=["post"], url_path="login")
    def login(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        data = {"token": serializer.validated_data["token"]}
        return Response(data, status=status.HTTP_200_OK)
```

앞서 말한 기능들을 추후 `drf`의 라우팅을 원활하게 사용하기 위해 하나의 `ViewSet`으로 작성하였다.
로그인 기능은 `ViewSet`의 `extra action`을 이용하여 구현하였다.

여기서 `DRF`에서 제공하는 기본 코드와 달라진 점은 토큰을 발급하는 코드를 직렬화 단계로 빼 놓은 부분이다.

## serializers.py 정의하기
```python
# serializers.py
from django.contrib.auth import authenticate
...

class UserLoginSerializer(serializers.Serializer):
    user_name = serializers.CharField()
    password = serializers.CharField(write_only=True)

    def validate(self, attrs):
        authenticate_kwargs = {
            "request": self.context["request"],
            "user_name": attrs["user_name"],
            "password": attrs["password"]
        }
        user = authenticate(**authenticate_kwargs)
        if user is None:
            raise AuthenticationFailed("로그인할 수 없습니다.")
        
        token, created = Token.objects.get_or_create(user=user)
        attrs["token"] = token.key
        return attrs
```
`view`의 메소드에서 토큰 부분이 빠졌기 때문에, validate에 해당 코드를 작성하였다.

`authenticate` 메소드는 로그인 실패 시 None을 반환한다. 이에 대한 예외 처리도 작성하였다.


# 커스텀 User 모델 작성하기
## 작성 주의점
`django`에서는 유저 모델을 커스텀 할 때, 최소 `AbstractBaseUser`을 상속하여 작성하는 것을 추천한다.

하지만 꼭 따로 모델을 만들 필요가 있다면 넣어야 할 속성들이 있다.

```python
    @property
    def is_anonymous(self):
        return False

    @property
    def is_authenticated(self):
        return True
```
첫번째로는 읽기 전용인 익명 사용자에 대한 보안 때문에 사용되는 `property`다.

`is_anonymous`는 익명 사용자와 일반 사용자를 구별한다.  
`is_authenticated`는 사용자의 인증 여부를 알려준다.  

```python
    @property
    def is_active(self):
        return True
```
is_active는 사용자의 활성화 여부를 나타낸다.  
`django`의 기본 User 모델은 이를 테이블의 attribute로 사용하는 것을 권장하고 있다.

이번 프로젝트에서는 active 여부를 구분하지 않기 때문에 `property`로 작성하여 항상 `True`를 반환하도록 하였다.

해당 속성은 `settings.py`에 `DEFAULT_AUTHENTICATION_CLASSES`로 작성한 인증 설정에서 사용되므로 꼭 작성이 필요하다.

```python
    def check_password(self, raw_password):
        return self.password == raw_password
```
마지막으로 `check_password` 메소드이다.

해당 메소드는 입력된 password가 DB의 데이터와 일치하는지 사용되며, `django`에서 기본적으로 적용되는 비밀번호 암호화 때문에 `auth backend`에서 호출되는 메소드다.

마찬가지로 이번 프로젝트에서는 password 암호화를 적용하지 않았으므로, 간단하게 비교하는 함수로 작성하였다.

## 전체 코드
```python
from django.contrib.auth.base_user import BaseUserManager
from django.db import models


class UserManager(BaseUserManager):
    def create_user(self, user_name, password):
        user = self.model(
            user_name=user_name,
            password=password
        )
        user.save(using=self._db)
        return user

class User(models.Model):
    user_name = models.CharField(
        max_length=150,
        unique=True,
    )
    password = models.CharField(max_length=128)
    created_at = models.DateTimeField(auto_now_add=True)

    USERNAME_FIELD = "user_name"
    REQUIRED_FIELDS = ["password"]

    objects = UserManager()

    class Meta:
        db_table = "user"
        
    @property
    def is_active(self):
        return True

    @property
    def is_anonymous(self):
        return False

    @property
    def is_authenticated(self):
        return True

    def check_password(self, raw_password):
        return self.password == raw_password

```