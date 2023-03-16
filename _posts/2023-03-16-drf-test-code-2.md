---
title : DRF에서 테스트 코드 작성하기(2) - reverse
date : 2023-03-16 14:04:00 +0900
categories : [Backend, Django]
tags : [drf, django]
---

# 개요
앞서 테스트 코드에서 url을 문자열로 선언을 해 두었다.

하지만 선언했던 url에 오탈자를 수정하였거나, url을 아예 고친 경우를 고려해보자.  
이런 상황에서는 urls.py에 수정한 대로 테스트 코드의 url도 고쳐야하는 문제점이 있다.

이런 상황을 간단히 해결할 수 있는 것이 바로 `reverse`이다.

# reverse
## 작동 원리
reverse 함수는 `reverse(viewname, *args, **kwargs)`와 같이 viewname과 args, kwargs를 사용한다.

여기서 viewname은 urls.py에 선언해둔 `name`을 의미한다.

```python
# urls.py
...
router = routers.DefaultRouter()
router.register(r"", views.NoticeViewSet, basename="notice")

urlpatterns = [
    path("", include(router.urls)),
    path("login", TokenObtainPairView.as_view(), name="login")
]
...
```

### router의 경우
![router example](/assets/img/router-simple-router-path.png)

router로 등록한 path의 URL name은 위와 같은 표를 따른다.

여기서 retrieve, update, delete와 같은 메소드는 reverse의 args, kwargs를 통해 정확한 대상을 지정할 필요가 있다.

## 적용 방법
notice를 예로 들면 다음과 같다.

여기서 args 또는 kwargs에서의 값은 상수나 변수를 사용할 수 있다.
```python
reverse("notice-list")                        # /notice/
reverse("notice-detail", args=[1])            # /notice/1/
reverse("notice-detail", kwargs={"pk": 2})    # /notice/2/
```

# 예제 코드
```python
# urls.py
...
router = routers.DefaultRouter()
router.register(r"", views.NoticeViewSet, basename="notice")

urlpatterns = [
    path("", include(router.urls)),
]
...

# test_notice_update.py
...
class NoticeUpdateTestCase(APITestCase):
    @classmethod
    def setUpTestData(cls):
        ...
        cls.notice = Notice.objects.create(
            user=user
            title="test title",
            context="test context"
        )
        cls.url = reverse("notice-detail", kwargs={"pk": notice.id})
        ...

```