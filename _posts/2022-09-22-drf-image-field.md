---
title : DRF에서 ImageField 변환하여 저장하기
date : 2022-09-22 17:23:00 +0900
categories : [Backend, Django]
tags : [drf, django]
img_path: /assets/img/2022-09-22/
---

# 개요
Django에서 ImageField를 사용하면 이미지 데이터를 url로 관리할 수 있다.  
하지만 저장을 할 때 이미지 파일의 크기를 변경하거나 회전 처리를 하는 등 추가 작업을 진행해야 할 수도 있다.

이럴때는 어느 부분을 건드려야 할까?

# 구현
## 첫번째 방법, 저장 후 불러오기
첫번째로 생각해볼 수 있는 방법은 다음과 같다.

1. 원본 파일을 그대로 저장
2. 저장한 파일을 불러와 후처리를 진행
3. 기존 데이터 삭제 후 저장

해당 방법의 문제점은 데이터의 저장이 두번 호출된다는 점이다.  
로컬 저장소를 사용한다면 문제가 없겠지만, S3와 같은 클라우드 저장소를 사용한다면 비용이 두배로 나오는 문제점이 있다.

## 두번째 방법, 저장되기 전에 처리
저장없이 이미지 처리를 하기위해 CREATE의 구조를 살펴보았다.  
DRF에서 CREATE는 다음과 같다.

```python
    def create(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        self.perform_create(serializer)
        headers = self.get_success_headers(serializer.data)
        return Response(serializer.data, status=status.HTTP_201_CREATED, headers=headers)

    def perform_create(self, serializer):
        serializer.save()
```

1. body로 받은 데이터를 직렬화한다.
2. Validation 검사를 진행한다.
3. preform_create를 호출한다 -> **serializer.save() 수행**
4. 헤더를 설정하고 반환

여기서 serailizer.save() 메소드는 다시 self.create()를 호출한다. 그렇다면 시리얼라이저의 create 메소드를 수정하면 원하는 방법을 찾을 수 있다!

```python
class ImgProcSerializer(serializers.ModelSerializer):
    class Meta:
        model = ImageProcess
        fields = ['id', 'img']

    def create(self, validated_data):
        return super().create(validated_data)
```

create 메소드의 `validated_data`에는 직렬화된 데이터가 있으며, 여기서 ImageField로 선언했었던 img의 경우 `InMemoryUploadedFile` 자료형으로 들어가 있다.

해당 파일은 python의 Pillow 패키지로 충분히 열 수 있다.
예를 들어 1000x1000으로 이미지를 변환해 저장하는 코드를 짜 보았다.

```python
from django.core.files.uploadedfile import InMemoryUploadedFile
from io import BytesIO
from PIL import Image
...
class ImgProcSerializer(serializers.ModelSerializer):
	...
    def img_resize(img: InMemoryUploadedFile) -> InMemoryUploadedFile:
        pil_img = Image.open(img).convert('RGBA')
        pil_img = pil_img.resize((1000,1000))

        new_img_io = BytesIO()
        pil_img.save(new_img_io, format='PNG')
        result = InMemoryUploadedFile(
            new_img_io, 'ImageField', img.name, 'image/png', new_img_io.getbuffer().nbytes, img.charset
        )
        return result


    def create(self, validated_data):
        result = self.img_resize(validated_data['img'])
        validated_data['img'] = result
        return super().create(validated_data)
```

`img_resize` 메소드에서 리사이즈 작업을 진행해 주었다.
자료형을 통일해야 하기 때문에, 마지막에 InMemoryUploadedFile를 새로 만들었다.

Django의 InMemoryUploadedFile 생성자에 필요한 데이터는 다음과 같다.
(`file`, `field_name`, `name`, `content_type`, `size`, `charset`)

여기서 변경이 필요한 부분은 첫번째인 `file` 부분이므로, 나머지는 기존값을 최대한 활용하도록 작성하였다.


# 전체 코드
```python
from io import BytesIO

from django.core.files.uploadedfile import InMemoryUploadedFile
from PIL import Image
from rest_framework import serializers

from api.imageprocess.models import ImageProcess

...
class ImgProcSerializer(serializers.ModelSerializer):
    class Meta:
        model = ImageProcess
        fields = ['id', 'img']

    def img_resize(self, img: InMemoryUploadedFile) -> InMemoryUploadedFile:
        pil_img = Image.open(img).convert('RGBA')
        pil_img = pil_img.resize((1000,1000))

        new_img_io = BytesIO()
        pil_img.save(new_img_io, format='PNG')
        result = InMemoryUploadedFile(
            new_img_io,
            'ImageField',
            img.name,
            'image/png',
            new_img_io.getbuffer().nbytes,
            img.charset
        )

        return result


    def create(self, validated_data):
        result = self.img_resize(validated_data['img'])
        validated_data['img'] = result
        return super().create(validated_data)
...
```