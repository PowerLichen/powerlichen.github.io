---
title : DRF의 페이지네이션(Pagination)
date : 2022-09-15 23:16:00 +0900
categories : [Backend, Django]
tags : [drf, django]
img_path: /assets/img/2022-09-15/
---

# 개요
API 호출 중에는 테이블 내 전체 데이터를 불러오는 작업이 있다.  
하지만 테이블 내 데이터가 수백개, 수천개가 된다면 한번의 호출로 너무 많은 데이터가 전송 될 것이다.

# DRF 페이지네이션
[공식 문서](https://www.django-rest-framework.org/api-guide/pagination/)

DRF에서는 이를 페이지 단위로 해결하기 위한 페이지네이션을 제공한다.

기본적으로 제공되는 페이지네이션 클래스는 다음과 같다.
## API 레퍼런스
### PageNumberPagination
쿼리 패러미터인 `page`를 통해 페이지네이션을 수행한다.

### LimitOffsetPagination
쿼리 패러미터인 `offset`과 `limit`을 사용한다.
- offset: 몇 번째 레코드 부터 출력할지 설정. 기본값 0
- limit: 몇 개의 레코드를 보여줄 지 설정.

### CursorPagination
다음 페이지로 넘어갔을 때 중복되지 않도록 출력하는 페이지네이션.

마지막으로 노출된 객체를 기억하여 그 객체로부터 `PAGE_SIZE`만큼 출력한다.


# DRF 페이지네이션 적용하기
## DEFAULT_PAGINATION_CLASS
Django의 settings.py에 `DEFAULT_PAGINATION_CLASS` 설정을 통해 페이지네이션을 전역적으로 관리할 수 있다.

```python
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.LimitOffsetPagination',
    'PAGE_SIZE': 100
}
```

## pagination_class
View에서 pagination_class를 정의하여 지역적으로 설정할 수 있다.

```python
class LargeResultsSetPagination(PageNumberPagination):
    page_size = 1000
    page_size_query_param = 'page_size'
    max_page_size = 10000

class BillingRecordsView(generics.ListAPIView):
    queryset = Billing.objects.all()
    serializer_class = BillingRecordsSerializer
    pagination_class = LargeResultsSetPagination
```

# 커스텀 페이지네이션
각 기능마다 다른 형태의 페이지네이션이 필요하다면 직접 커스텀하는 것이 필요하다.

특히, 페이지네이션의 Response를 바꿔야 하는 경우가 종종 있다.  
DRF에서 기본적으로 제공하는 페이지네이션은 다음과 같은 구조로 반환된다.

```json
HTTP 200 OK
{
    "count": 1023,
    "next": "https://api.example.org/accounts/?page=5",
    "previous": "https://api.example.org/accounts/?page=3",
    "results": [
       …
    ]
}
```

이는 `get_paginated_response` 메소드를 오버라이딩하는 것을 통해 해결할 수 있다.

예시에서는 다음과 같은 값을 포함하도록 하였다.
- results: 결과 데이터
- curPage: 현재 페이지
- maxPage: 전체 페이지 개수

```python
class CustomPagination(pagination.PageNumberPagination):
    def get_paginated_response(self, data):
        return Response({
            'results': data,
            'curPage': self.page.number,
            'maxPage': self.page.paginator.num_pages
        })
```



