---
title : DRF에서 Authentication과 Permission
date : 2022-10-05 23:16:00 +0900
categories : [Backend, Django]
tags : [drf, django]
img_path: /assets/img/2022-10-05/
---

# 개요
DRF에는 `인증`과 `권한` 개념이 있다.

서버로 들어오는 요청을 기반으로 자격 증명을 진행하는 것이 `인증`  
그리고 현재 접근에 대한 통과 여부를 결정하는 것이 `권한`

DRF에서는 이러한 `인증`과 `권한`이 어떻게 구성되어 있는지 알아보자.

그리고 알아낸 정보를 바탕으로 `커스텀 권한`을 만드는 방법을 알아보자

# 인증(Authentication)
DRF에서 인증을 지정하는 방법은 두 가지가 있다.

첫 번째는 `settings.py`에 DEFAULT_AUTHENTICATION_CLASSES를 지정하는 방법이다.
```python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.BasicAuthentication',
        'rest_framework.authentication.SessionAuthentication',
    ]
}
```

두 번째는 각 View에 대해서 인증을 지정하는 방법이다.
```python
...
class ExampleView(APIView):
    authentication_classes = [SessionAuthentication, BasicAuthentication]
    ...
```

Authentication은 들어오는 요청의 내용에 따라 인증을 진행한다.
인증을 통해 사용자를 파악한 경우, `request.user`에는 Django의 사용자 인스턴스가 할당된다.
`queryset` 등에서 이 정보를 활용하여 필터링을 진행할 수 있다.

# 권한(Permissions)
권한은 현재 사용하려는 API에 대한 권한을 확인하는 단계이다.  
기본적으로 DRF에는 다음과 같이 설정되어 있다.

```python
REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.AllowAny',
    ]
}
```

해당 항목을 수정하여 전역적 `권한`을 지정할 수 있다.

이 때, 각 `View`는 `permission_classes`를 지정하는 것으로 전역 권한을 무시할 수 있다.

```python
class ExampleView(APIView):
    permission_classes = [IsAuthenticated]
```

# 커스텀 Permissions
위와 같이 DRF에서 기본적으로 제공하는 권한을 사용할 수 있다면 쉽게 개발을 할 수 있다.

하지만 프로젝트를 진행하면서 인증 서버를 별도로 사용하는 경우가 생겼다.  
이럴 때는 어떻게 하는 것이 좋을까?

DRF 공식 문서에는 권한 커스터마이징에 대한 내용이 작성되어 있다.

[Custom permissions](https://www.django-rest-framework.org/api-guide/permissions/#custom-permissions)

```python
...
class UserAccessPermission(permissions.BasePermission):
    def has_permission(self, request, view):
        ...
    def has_object_permission(self, request, view, obj):
        ...
```

각 메소드는 통과에 해당하면 `True`를, 거부라면 `False`를 반환해야 한다.

여기서 두 메소드의 역할은 다음과 같다.

> has_permission  
> - 모든 HTTP 요청에 대해 실행된다.

> has_object_permission  
> - DRF의 메소드인 get_object에 대해서 실행된다.  
> - 즉, POST 요청에 대해서는 수행되지 않는다.

위 사항을 기반으로 외부 인증 서버를 사용한 Permission을 만들어보았다.

```python
class UserAccessPermission(permissions.BasePermission):
    def has_permission(self, request, view):
        token = request.META.get('HTTP_AUTHORIZATION')
        url = 'http://auth-server:8080/api/auth/token/'
        try:
            res = requests.get(url, headers={'Authorization': token})
        except requests.exceptions.ConnectionError as err:
            raise AuthServerConnectionError
        
        if res.status_code == requests.codes.ok:
            return True

        return False

    def has_object_permission(self, request, view, obj):
        token = request.META.get('HTTP_AUTHORIZATION')
        return obj.user == decodeJWTPayload(token):

```

`has_permission` 메소드를 통하여 외부 인증서버를 확인한다.
`has_object_permission` 메소드에서는 현재 사용하려는 오브젝트와 token의 사용자 id가 같은지 확인한다.

실제로는 이외에도 추가로 구현할 사항이 많지만, 간략하게 이 정도로만 정리하였다.