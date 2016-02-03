---
layout:     post
title:      모씨 서비스 구성에 대해
summary:    모씨 아키텍쳐에 대해서 이야기 합니다.
categories:
---
2016년 새해가 벌써 한 달이 지났습니다. 2014년 11월 1일에 오픈
한 모씨 서비스도 하루가 다르게 바뀌고, 성장하고 있습니다. 오늘
은 그 동안 미루어 두었던 모씨 서비스를 구성하는 다양한 구성에 
대해 이야기를 해 볼까 합니다. 실제로 많은 분들께서 모씨 시스템
구성에 대해 궁금해 하시고, 몇몇 분들은 직접 사무실에 오셔서 
문의를 주신 적도 있었습니다. 지난 1년 3개월동안 발전해 온 구성에 
대해 알아보도록 하겠습니다.

### Hosting
모씨는 AWS EC2 를 사용합니다. 저희가 상주하는 곳은 도쿄 리전이며,
여러 사정으로 인하여 서울 리전으로 이주할 계획은 현재 없습니다.
AWS 에서는 굉장히 많은 종류의 서비스를 제공하는데요, 가급적 AWS 에
지나치게 의존성이 발생하는 것을 최소화 하고자 서버 호스팅은 EC2
외에는 아주 일부분에 한하여 AWS 전용 서비스를 이용하고 있습니다.

서비스 초기에 AWS 에 대해 자세히 알지 못하는 상태로 시스템을 구성 
하였다가 [EC2 의 CPU credit](http://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/t2-instances.html#t2-instances-cpu-credits) 
과 [EBS 의 IOPS limit](http://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/EBSVolumeTypes.html) 
에 부딪혀 낭패를 겪은 적이 몇 번 있었습니다. 특히 메인 
데이터베이스의 EBS  IOPS limit 문제가 터졌을 때는 정말 멘붕(?) 
이었었는데요, 상용 서비스를 구축할 때는 반드시 관련 문서들을 참고
하여 저희와 같은 삽을 뜨지 않기를 바랍니다.

그 외에 저희가 실질적으로 보유하고 있는 하드웨어 장비는 사내에 있는
맥 미니 서버 한 대가 있습니다. iOS 클라이언트의 자동 빌드를 위해
보유하고 있습니다.

### OS
맥 미니 서버에 설치된 OSX 을 제외하고는 모든 서버에서 Ubuntu 
14.04 LTS  를 이용하고 있습니다. 모씨 팀에서 활동 중인 
개발자는 4명이며, 상대적으로 설정과 관리가 편한 Ubuntu 
는 그야말로 소규모 개발팀에게 축복과도 같은 운영체제라고 할 
만합니다. 1 년을 넘는 시간 동안 250만명 이상의 회원의 트래픽을
수 십대의 Ubuntu 14.04 로 운영하며 한 번의 문제도 일으킨
적이 없었습니다.

### Database
모씨는 다양한 데이터베이스가 얽혀서 동작하고 있습니다. 
하나씩 살펴보도록 하겠습니다.

#### PostgreSQL
모씨 서비스는 메인 데이터베이스로 PostgreSQL 9.4 를 이용하고
있습니다. 이곳에 가장 중요한 데이터들이 저장되고 있습니다.
PostgreSQL 은 그 특성 상 다른 데이터베이스에 비해 부하가
있을 경우에 튜닝 포인트가 좀 여러군데입니다. 아마 데몬이
멀티 스레드가 아닌 멀티 프로세스 형태로 동작하기 때문이
아닐까 합니다. 모씨 서비스의 경우 원하는 성능을 맞추기
위해 오토 배큠, 커넥션 풀링, 커널 리소스등등 다양한 부분에
대한 설정 및 성능 튜닝이 이루어졌습니다. 모씨 팀의 경우
PostgreSQL 공식 문서 및 데이터베이스 사랑넷 및 페이스북
그룹등에서 여러 도움을 받을 수 있었습니다.

PostgreSQL 의 경우 멀티프로세스 형태로 동작하는 만큼 커넥션
풀링은 굉장히 중요한 요소입니다. 어플리케이션 수준에서
커넥션 풀링을 지원하기도 합니다만, 모씨의 경우 PgBouncer 를
통한 커넥션 풀링을 구현하고 있습니다.

어떤 파일이 운영체제 캐시에 올라와 있는지를 확인해 주는
vmtouch 라는 툴이 있습니다. 필요에 따라 PostgreSQL 의
데이터 파일이 어느정도 touching 되어 있는지를 확인하고
필요에 따라 touching 해줄 경우 데이터베이스 warm-up 에
큰 도움이 됩니다.

#### MariaDB
모씨에 등록된 수 억장의 카드에 포함되어 있는 해시 태그들은
중복 제거되어 별도 저장된 것만 600만개에 이릅니다. 이
데이터들을 메인 데이터베이스에서 like 검색을 시도 할 경우
상황에 따라 심각한 퍼포먼스 저하를 일으키곤 하였습니다.
검색엔진부터 시작해서 다양한 데이터베이스들을 테스트
하면서 MySQL 을 포크한 MariaDB 를 선택 하였습니다. 그리고
MariaDB 는 우리가 필요로 하는 성능을 충분히 내 주었습니다.

다만 MariaDB(MySQL 포함) 의 경우 몇 가지 단점을 가지고
있는데요, 그중 가장 큰 문제는 utf-8 문제입니다. utf-8 설정이
아닌 utf-8 mb4 설정을 하지 않을 경우 4바이트 기반의 utf-8
문자열을 저장하지 못하는 문제가 발생하며(스마트폰의 emoji
아이콘들이 대표적인 예입니다.) 대소문자를 구분하는 검색을
하기 위해서는 별도의 collate 문을 추가해야 합니다. 또한
varchar 타입의 경우 767 byte 까지만 허용하기 때문에 utf-8 mb4
기준으로는 약 190자만 저장 가능하게 됩니다. 이래저래 특이한
제약조건이 많은 데이터베이스이긴 합니다만, 제한된 상황과
통제된 모델링을 통해서 충분히 극복할 수 있는 부분이라
생각합니다.

다만 MariaDB 및 MySQL 을 처음 접하시는 분들이 이런 MariaDB
만의 특성을 RDBMS 의 전체적인 특성으로 오해하는 일은 없으면
좋을 듯 합니다.

#### MongoDB
NoSQL 계에서 가장 유명한 MongoDB 를 여러군데에서 사용하고
있습니다. 주로 사용하는 곳은 BSon 형태로 데이터를 쉽게 쓰고,
읽을 수 있는 곳에서 사용합니다. 예를 들어 어떤 카드의 순
조회수를 기록하고자 할 때, 순 조회수 자체는 RDBMS 에 기록
됩니다만, 해당 카드를 조회한 사용자들은 MongoDB 에 기록되는
형식입니다. MongoDB 는 3.0 때 도입된 WiredTiger 스토리지
엔진으로 데이터 저장 효율이 비약적으로 향상 되었습니다.

MongoDB 를 사용할 때 유의해야 할 점이 있습니다. aggregation
쿼리나 index 를 사용할 때인데요, TPS 가 급증할 경우 MongoDB
를 이용할 때 모씨 팀은 다음과 같은 문제를 겪은 경험이 있습니다.

* 메인 데이터베이스로 사용하기 위해 기본적인 스키마를 설정한다.
* 모델링에 맞추어 데이터를 저장한다.
* 서비스가 발달하며 쿼리가 복잡해지거나, 데이터가 늘어나기 시작한다.
* 쿼리의 성능이 떨어지는 것을 확인한다.(특히 find)
* 인덱스를 생성한다.
* 인덱싱 작업 때문에 insert/update 가 더 느려진다.
* 다 느려진다.

실제로 저희가 겪었던 문제이기도 한데요, MongoDB 자체는
Schemaless 한 데이터베이스인만큼 RDBMS 만큼의 인덱스 성능이
나오지는 않습니다. 또한 마구잡이로 막(?) 때려넣고 조회
하는 자유로움을 스스로 포기하는 부분이라고도 생각해 볼
수 있을 것 같습니다. 아무쪼록 주의깊게 사용할 필요가
있습니다.

#### DynamoDB
AWS 에서 제공하는 NoSQL 류의 데이터베이스입니다. 쉽게 구축
이 가능하고, Read/Write Capacity 에 따라 가격이 책정되는 구조라
손쉽게 Scale In/Out 을 할 수 있습니다. 다만 인덱스가 제한적이고,
쿼리 역시 매우 제한적입니다. 별도의 서버 구축 없이, 간단한
데이터를 쌓는 용도에 추천할 만한 서비스입니다.

모씨 서비스의 경우 초기 극히 일부분에 한하여 DynamoDB 가 
사용되었으며, 대부분의 로그성 데이터는 MongoDB 에 저장되도록
코드가 수정 되었습니다.

### Storage
이용자들이 생성하는 수억장의 카드 이미지는 AWS S3 에 저장됩니다.
실제로 이런 단순 데이터 스토리지는 직접 구축하는 것 보다
S3 를 이용하는 것이 어떤 식으로든 이득입니다. AWS 의 수 많은
서비스들 중 개인적으로 가장 빛나는 서비스가 아닌가 생각합니다.

이곳에 업로드 된 수 테라 바이트의 이미지는 CloudFront CDN 으로
배포 됩니다. CloudFront 는 외부 CDN 에 비해 특별히 뛰어난 점이
있는 편도 아니며, 자세히 요금을 따져 보면 S3 에서 직접 서빙하는
것 보다 비쌀 수도 있습니다. 그럼에도 불구하고 CloudFront 에는
두 가지 장점이 있는데요, 

* S3 에서 CloudFront 로의 서빙 비용이 무료입니다.
* 예약 요금을 사용하게 될 경우 기간/트래픽 등의 조건을 통해
추가 가격할인이 가능 하다는 점입니다.

### Cache
주력 캐시 서버는 모두 redis 로 동작합니다. Ubuntu ppa 에 등록된
chris lea 의 redis 패키지는 jemalloc 과 함께 빌드 되어 있어
추가적인 성능 향상을 기대해 볼 수 있습니다. 모씨 서버 어플리케이션
내부에 자체적인 샤딩 루틴이 포함되어 있어, 여러 대의 redis
서버 인스턴스에 다양한 데이터가 분산 저장되며, 히트가 되지
않을 경우 자동으로 데이터베이스로부터 긁어오도록 되어 있습니다.

또한 수 많은 주요 데이터들은 RDBMS 에 바로 기록되지 않도록 
설계하였습니다. 직접 RDBMS 에 데이터를 업데이트 할 경우 
예상할 수 있는 스트레스 때문인데요, 따라서 redis 서버에 먼저
데이터를 set 하고, 일정한 딜레이 후에 캐시 서버에서 데이터베이스
로 데이터를 저장하도록 하는 루틴들이 포함되어 있습니다.

이때 redis 캐시 서버에 여러 커넥션이 동시에 set 을 하여
데이터 정합성 문제를 일으키지 않기 위해 여러 큐 형태의
파이프 라인 코드가 동작하고 있습니다.

### Search
검색 엔진으로는 AWS CloudSearch 와 ElasticSearch 를 사용하고 있으며,
현재 AWS CloudSearch 를 모두 덜어내고 ElasticSearch 로 마이그레이션을
지속적으로 수행하고 있습니다. 지리 정보 관련한 데이터 질의 역시
초기엔 MongoDB 의 Geolocation 쿼리를 이용하였으나, 현재는
ElasticSearch 기반으로 변경한 상태입니다.

### Application Server
서버 상에서 동작하는 모든 어플리케이션은 Python 으로 작성
되었습니다. 메인 어플리케이션은 Flask 기반으로 구현 되었습니다.
PostgreSQL 데이터베이스와는 SQLAlchemy 와 Alembic 을 통해
구성 되어 있습니다. 저의 경우 Django 를 사용해 오다가 이번에
처음으로 SQLAlchemy 를 사용하였는데, 처음엔 ORM 코드를 작성하는게
정말 익숙치 않았는데, 1년 정도 꾸준히 쓰면서 이제 적응이
되어 가는 것 같습니다.

관리 시스템 서버의 경우 Django 로 구현되어 있습니다.
어떻게 보면 어플리케이션 서버의 경우 단순한 프레임워크 + 
라이브러리의 조합으로 구성되어 있고, 관리 시스템의 경우
최대한의 생산성을 보장해 줄 수 있는 풀 스택 프레임워크를
도입했다고 생각할 수 있겠습니다.

WSGI 구현체로는 uWSGI 를 이용합니다. 설정이 복잡하긴 하지만,
Gunicorn 도 사용해 본 결과 성능 상으로 uWSGI 가 더 앞서는
것 같습니다. 모씨 서비스의 특수성인지는 잘 모르겠는데 uWSGI
설정 시 thread 를 비활성화 시키고 process 를 늘리는 것이 성능
향상에 큰 도움이 되었습니다. 다만 process 별 수행 성능 편차를
최대한 막기 위해서 gevent loop engine 설정을 적용하였습니다.

다양한 비동기 작업 및 지연 작업, 캐시-데이터베이스 간 동기화
작업을 위해 celery 가 많은 곳에서 사용되고 있습니다. celery
역시 concurrency pool 로 gevent 설정을 적용된 상태입니다.

모든 uWSGI 는 Nginx 로 reverse proxying 되어 최종적으로 서비스
됩니다.

### RabbitMQ
Celery 의 Broker 로 최초에 redis 를 이용하였으나, 트래픽 급증과
함께 안정성 문제가 몇 번 발생하여 RabbitMQ 클러스터를 구성 하였습니다.
RabbitMQ 는 초당 수백 메가바이트의 트래픽을 안정적으로 소화
하고 있습니다.

### LoadBalancing/AutoScale
어플리케이션 서버는 AWS ELB 로 트래픽을 분산 처리 합니다. 
ELB 의 큰 장점 중 하나는 ELB 에 물려있는 서버들의 부하를 체크하여
AutoScale 옵션을 붙일 수 있다는 점입니다. 이런 이슈 덕분에
어플리케이션 서버 인스턴스 수를 항상 부하에 맞게 조절할 수
있습니다.

다만 ELB 의 경우 Idle Timeout 을 3600 초까지만 설정할 수 있다는
단점이 있습니다. 다른 곳에는 문제가 되지 않는데, RabbitMQ 클러스터에 
접속하는 Celery worker 들의 경우 문제가 될 수 있습니다. 특히
.pidbox 큐들의 경우 대부분의 시간동안 idle 상태인 관계로 celery
프로세스들에 문제를 일으킬 수 있습니다. 이에 RabbitMQ 클러스터는
HaProxy 를 이용하여 로드 밸런싱을 하도록 설정해 두었습니다.
HaProxy 의 경우 사실상 무제한의 idle timeout 을 설정할 수 있기
때문에 이 경우 유용하게 사용될 수 있습니다.

### Monitor
여러 모니터링 툴을 사용하고 있습니다. RabbitMQ, HaProxy,
ElasticSearch 등에서 자체적으로 제공하는 모니터링 툴 이외에도
Newrelic 을 이용한 서버, 어플리케이션 성능 분석을 수행하고 있으며,
Google Analytics 를 이용한 데이터 분석 역시 진행하고 있습니다.

### And...
지난 1년이 넘는 시간동안 모씨 서비스를 안정적으로 운영하기
위해 많은 시간과 노력을 투자했습니다. 4명의 개발팀원 중 2명만이
서버 개발을 진행했기 때문에 힘에 부치는 일도 많았고, 
장애 역시 숱하게 겪었습니다. 외부 팀의 컨설팅을 받은 적도
있고 밤새도록 공부하며 돌파한 적도 한 두번이 아니었던 것
같네요. 과거에도 그랬고, 지금도 마찬가지이며, 앞으로도 안정적인
서비스 제공은 모씨 개발팀의최우선 순위라는 사실을 알려드리고 싶습니다.

모씨 시스템에 대해 흥미가 있거나, 모씨 시스템에 해주고 싶으신
이야기가 있다면 언제든지 이야기 해 주세요. 경청하겠습니다.

긴 글 읽어주셔서 감사합니다.