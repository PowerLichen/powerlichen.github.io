---
title : DRF에서 JWT 기반 인증 사용하기
date : 2022-09-27 21:47:00 +0900
categories : [Backend, Django]
tags : [drf, django]
img_path: /assets/img/2022-09-27/
---

# 개요
JWT는 **토큰**을 기반으로 사용자를 식별하는 인증이다.

기존의 세션 방식의 인증은 서버의 메모리에 정보가 저장된다.  
HTTP 프로토콜은 상태 정보를 기억하지 않기 때문에 여러번의 세션 검증이 이루어지는데, 이는 곧 서버의 부하로 이어진다.  

JWT는 토큰 자체에 사용자의 정보들이 들어있으며, 서버에서의 서명을 통해서 간단하게 사용자를 특정할 수 있다.

# Simple JWT
[공식 문서](https://django-rest-framework-simplejwt.readthedocs.io/en/latest/)

`Simple JWT`는 DRF의 인증에서 JWT를 쓸 수 있는 서드파티 패키지다.  
pip를 통해 간단히 설치할 수 있다.

```shell
pip install djangorestframework-simplejwt
```

## 로그인과 리프레시
다음으로는 settings.py에 simplejwt 관련 내용을 등록한다.

```python
INSTALLED_APPS = [
    ...
    'rest_framework_simplejwt'
    ...
]
...
REST_FRAMEWORK = {
    ...
    'DEFAULT_AUTHENTICATION_CLASSES': (
        ...
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    )
    ...
}
...
```

등록이 완료되었으면 urls.py에 simplejwt의 view를 등록한다.

```python
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
)

urlpatterns = [
    ...
    path('api/auth/login/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('api/auth/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
    ...
]
```

리프레시의 경우, 리퀘스트 body로 `{"refresh": <토큰>}`를 포함하여야 한다.

## 로그아웃 기능
jwt에서 로그아웃을 처리하는 방식으로는 두 가지가 있다.
1. 클라이언트에 저장된 JWT를 삭제하기
2. 블랙리스트 생성

여기서는 `Simple JWT`에서 제공하는 블랙리스트 기능에 대해 알아보자.

우선 settings.py에 다음 앱을 등록해야 한다.
```python
INSTALLED_APPS = (
    ...
    'rest_framework_simplejwt.token_blacklist',
    ...
)
```

그리고 로그아웃 처리에 대한 url을 등록한다.
```python
from rest_framework_simplejwt.views import TokenBlacklistView

urlpatterns = [
  ...
  path('api/auth/logout/', TokenBlacklistView.as_view(), name='token_blacklist'),
  ...
]
```

해당 경로로 POST 요청을 통해 로그아웃을 진행할 수 있다.
리퀘스트 body로 `{"refresh": <토큰>}`를 포함하여야 한다.
