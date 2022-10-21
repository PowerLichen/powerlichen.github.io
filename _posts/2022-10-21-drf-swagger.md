---
title : DRF에서 Swagger 문서 작성
date : 2022-10-21 17:06:00 +0900
categories : [Backend, Django]
tags : [drf, django]
img_path: /assets/img/2022-10-21/
---

# 개요
OpenAPI란 API 스펙을 json, yaml로 표현한 명세이다. 직접 소스코드나 문서를 보지 않고 서비스를 이해할 수 있다.  
오늘날에는 RESTful API 스펙의 사실상 표준으로 사용되고 있다.

현재 가장 사용되는 버전은 3.0으로, 구 버전인 2.0에 비해 재사용성이 향상되었다.

DRF에서도 이러한 OpenAPI 3.0을 손쉽게 만들기 위한 패키지가 있다.

# drf-spectacular
`drf-spectacular`는 DRF 환경에서 OpenAPI 3.0 구조를 손쉽게 만들어주는 패키지다.
Serializer 정보를 기반으로 자동으로 목록을 생성해주며, 여러 데코레이터를 통해 수정가능한 옵션을 제공한다.

설치는 pip를 통해 간단히 할 수 있다.
```shell
pip install drf-spectacular
```

프로젝트의 `settings.py`에는 다음과 같이 설정한다.
```python
INSTALLED_APPS = [
    ...
    'drf_spectacular',
]
...
REST_FRAMEWORK = {
    ...
    'DEFAULT_SCHEMA_CLASS': 'drf_spectacular.openapi.AutoSchema',
}
```

또한, `SPECTACULAR_SETTINGS`를 추가하여 페이지 내용을 수정할 수 있다.
몇 가지 기능에 대한 설명은 다음과 같다.
```python
SPECTACULAR_SETTINGS = {
    'TITLE': '',            # OpenAPI 3.0 페이지 타이틀,
    'DESCRIPTION': '',      # OpenAPI 3.0 페이지 설명,
    'VERSION': '1.0.0',     # 버전 정보
}
```

# 사용방법
## @extend_schema 사용법
View의 메소드에 사용하는 데코레이터.
클래스 단위의 데코레이터인 `@extend_schema_view`에서도 사용할 수 있다.

기본적인 구현은 다음과 같다.
```python
class XViewset(mixins.ListModelMixin, viewsets.GenericViewSet):
    @extend_schema(description='text')
    def list(self, request, *args, **kwargs):
        return super().list(request, *args, **kwargs)
```

또한, 이 코드는 다음과 같다.
```python
@extend_schema_view(
    list=extend_schema(description='text')
)
class XViewset(mixins.ListModelMixin, viewsets.GenericViewSet):
    ...
```

## @extend_schema 내부 데이터
`extend_schema`의 옵션으로 들어갈 수 있는 값은 여러가지가 있다.  
기본적으로 `drf-spectacular`가 자동으로 값을 찾아주지만 직접 입력을 해 줘야 하는 값도 있다.

![extend_schema](swagger-extend-schema.png)


# OpenAPI 스키마 커스터마이징
### summary
짧은 설명문을 작성할 수 있다.  
API 문서에서 URL의 오른쪽에 표시된다.

### parameters
Type: `Optional[List[Union[OpenApiParameter, _SerializerType]]]`

URL 요청에 필요한 `쿼리 패러미터`를 리스트로 정의한다.
DRF `Serializer` 또는 `OpenApiParameter`로 선언할 수 있다.
```python
@extend_schema(
    ...
    parameters=[
        OpenApiParameter(
            name="id",
            type=OpenApiTypes.NUMBER,
            required=True
        ),
        OpenApiParameter(
            name="today",
            type=OpenApiTypes.DATE,
            required=True
        ),
    ]
    ...
)
...
```
### responses
자동으로 검색된 `Serializer`를 대체한다.

사용할 수 있는 타입은 다음과 같다.
- Serializer 클래스
- Serializer 인스턴스: 리스트의 경우 Serializer(many=True)와 같이 사용
- OpenApiResponse
- 딕셔너리 형태

```python
@extend_schema(
    ...
    # 일반 시리얼라이저 할당
    responses=CustomSerializer
    # 딕셔너리 형태 <상태코드>:<시리얼라이저>
    responses={
        200: CustomSerializer,
        400: OpenApiResponse(description="Error"),
    }
    ...
)
```

### request
일반적으로 `Serializer`로 정의한다.

### **examples**
자주 사용되는 부분이다.  
`OpenApiExample`를 사용하여 예제 정보를 작성할 수 있다.

```python
examples=[
    OpenApiExample(
        name="success_example",
        value={
            "id": 1,
            "title": "test title",
            "context": "test context",
        }
    ),
    OpenApiExample(
        name="failure_example",
        value={
            "msg": "error occured"
        }
    ),
]
```
