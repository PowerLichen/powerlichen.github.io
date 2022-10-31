---
title : Django Settings.py 분리하기
date : 2022-10-31 22:24:00 +0900
categories : [Backend, Django]
tags : [drf, django]
img_path: /assets/img/2022-10-31/
---

# 개요
프로젝트에 도커 적용하면서 개발환경과 배포환경 분리의 필요성을 느꼈다.

배포 환경에서는 `debug`를 `False`로 변경하고, `ALLOWED_HOSTS`를 추가하여야 했다.

하지만 배포 환경 `settings.py`로는 로컬에서 테스트가 되지 않기 때문에 일일히 개발 환경 상태로 값을 고치고 작업하였다.

이런 번거로움을 덜기 위한 작업이 바로 setting의 분리다.

# settings.py 분리하기
환경을 분리하기 위해 똑같은 `settings.py`를 여러개로 분리하는 것이 답이 아니다.

우선, 모든 환경에서 공통적으로 사용할 수 있는 환경설정은 `base.py`로 선언한다.  
그리고 `local`용 설정과 `production`용 설정으로 파일을 나누었다.

```
├── project
│   ├── settings
│   │   ├── base.py
│   │   ├── local.py
│   │   └── prod.py
│   ├── __init__.py
│   ├── asgi.py
│   ├── urls.py
│   └── wsgi.py
...
```
## 환경설정 파일
### base.py
기존의 `settings.py` 내용을 그대로 사용하였다.  
여기서 환경에 따라 달라질 부분인 `DEBUG`, `ALLOWED_HOSTS`, `DATABASES` 등을 제외하였다.

또한, BASE_DIR를 재정의해야한다.
```python
...
BASE_DIR = Path(__file__).resolve().parent.parent.parent
...
```
기존 `settings.py` 위치는 `/<프로젝트명>/project/` 였다.  
하지만 이제는 `base.py`의 위치가 `/<프로젝트명>/project/settings/`로 한 단계 깊어졌다.  
이를 반영하기 위에 뒤에 `.parent`를 추가하였다.

### local.py
`base.py`를 import를 통해 불러온 뒤, 필요한 부분을 정의해준다.
```python
from .base import *
DEBUG = True
ALLOWED_HOSTS = ['*']
...
DATABASES = {
    'default': {
        ...
        'HOST': "localhost",
        ...
    }
}
...
```

### prod.py
`base.py`를 import를 통해 불러온 뒤, 필요한 부분을 정의해준다.

정의한 데이터는 배포 환경에 맞도록 구성하였다.

# settings.py를 지정 실행하기
기존의 서버 실행 명령으로 Django 서버를 실행하면 오류가 나는 것을 볼 수 있다.  
이는 `settings.py`가 사라지면서 읽지 못하기 때문이다.

그러므로 실행할 때는 다음과 같은 명령어를 실행하여야 한다.
```shell
# 로컬 설정 기반 실행
python manage.py runserver --settings=project.settings.local

# 배포 설정 기반 실행
python manage.py runserver --settings=project.settings.local
```

환경 변수를 사용한 방법도 있다.  
`DJANGO_SETTINGS_MODULE` 환경 변수에 경로를 설정해두면, Django가 자동으로 해당 설정 파일을 읽는다.