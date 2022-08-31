---
title : Django에서 다중 업데이트 구현
date : 2022-08-31 23:05:00 +0900
categories : [Backend, Django]
tags : [drf, django]
img_path: /assets/img/2022-08-31/
---

# 개요
기본적으로 DRF에서는 ViewSet을 사용한다.  
다른 프레임워크의 `Recources` 또는 `Controllers`가 해당 기능과 유사하다고 할 수 있다.

기존 Django의 View에서는 get, post와 같은 function으로 HTTP 메소드 요청을 처리한다.  
하지만 DRF의 ViewSet은 HTTP 메소드에 대한 핸들러를 제공하는 것이 아니라 클래스 기반의 list, create와 같은 형태로 해당 메소드를 처리한다.

해당 기능은 ViewSet의 `.as_view()` 옵션을 통해 바인딩 된다.
```python
user_list = UserViewSet.as_view({'get': 'list'})
user_detail = UserViewSet.as_view({'get': 'retrieve'})
```
하지만 일반적으로는 이렇게 하지 않고, ViewSet을 라우터에 등록하여 urlconf가 자동으로 생성되도록 하여 사용된다.

```python
# urls.py
from myapp.views import UserViewSet
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.register(r'users', UserViewSet, basename='user')
urlpatterns = router.urls

# views.py
class UserViewSet(viewsets.ModelViewSet):
    serializer_class = UserSerializer
    queryset = User.objects.all()
```

기존의 View를 사용하지 않고 ViewSet을 사용할 경우 장점은 다음과 같다.
- queryset과 같은 중복되는 작업을 클래스의 변수로서 사용할 수 있다.  
동일 DB에 대한 GET, POST, PUT, PATCH 등은 어떤 HTTP 메소드가 오더라도 하나의 쿼리셋만 사용하므로 중복을 줄일 수 있다.
- 라우터를 사용하여 path 작성이 간편해진다.

# ModelViewSet
일반적으로 가장 자주 사용하는 ViewSet이다.

`ModelViewSet`을 사용하면 기본적인 list, create, retrieve, update, partial_update, destroy가 전부 자동생성된다.

이러한 자동화는 모두 ModelViewSet의  다음과 같은 구조로 때문이다.

![ModelViewSet](viewset-modelviewset.png)

ModelViewSet은 여러 ModelMixin을 전부 상속하고 있다.
## DRF의 generics
rest_framework.generics에 선언된 APIView 클래스들 중 일부는 다음과 같다.

|기능|HTTP Method|APIView|조합|
|---|-|-|-|
Create|POST|CreateAPIView|(GenericAPIView + CreateModelMixin)
Read(Many)|GET|ListAPIView|(GenericAPIView + ListModelMixin)
Read(One)|GET|RetrieveAPIView|(GenericAPIView + RetrieveModelMixin)
Update|UPDATE,PATCH|UpdateAPIView|(GenericAPIView + UpdateModelMixin)
Delete|DELETE|DestroyAPIView|(GenericAPIView + DestroyModelMixin)

이렇게 각각 필요에 맞게 사용하라고 만들어 둔 APIView가 있고, 위 다섯개의 APIVIew 기능들이 전부 합쳐진 것이 ModelViewSet이다.  
하지만 기능을 구현하다 보면 특정 기능만 필요할 때도 있다. DB 데이터에 대해 Read-only일 수도 있고, 조회가 되지 않아야 할 수도 있다.

그러므로 개발초기에 ModelViewSet을 사용하여 데이터들을 확인하며 개발 후, 필요한 부분만 남긴 APIView로 커스텀하는 것이 바람직하다.

## 커스텀 ViewSet
ModelViewSet의 구조와 비슷하게 필요한 기능만 상속할 수 있다.  
예를 들어, create, list, retrieve만 필요하다고 하면 다음과 같이 사용할 수 있다.

```python
from rest_framework import mixins

class CreateListRetrieveViewSet(
    mixins.CreateModelMixin,
    mixins.ListModelMixin,
    mixins.RetrieveModelMixin,
    viewsets.GenericViewSet
):
	pass
```

# ViewSet의 action 활용
ViewSet의 메소드에서는 현재 동작의 속성을 확인할 수 있다.

그 중 `action`은 현재 동작하는 action의 이름을 가지고 있다. 이를 통하여 동작을 제어할 수 있다.

예를 들어, 각 `action`에 대해 다른 시리얼라이저를 사용할 수 있다.
```python
def get_serializer_class(self):
    if self.action == "create":
        return CreateSerializer
    if self.action == "list":
        return ListSerializer
    
    return self.serializer_class
```