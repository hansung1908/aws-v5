# aws-v5
- 시작 전 aws 리전 설정을 서울로 꼭 설정

### Springboot 2.6.6, JDK 11
- devtools
- springweb
- lombok
- jpa
- mysql

### 전체구성
- 기존엔 ./gradlew build를 통해 프로젝트를 실행파일로 만들고 ec2 + 엘라스틱 빈스톡을 통해 배포
- 이번 프로젝트에선 깃허브에 코드 배포 후, ci 서버를 통해 테스트와 빌드를 진행
- 이때 aws와 ci서버의 환경이 동일해야 함
- ci서버에서 aws로 배포시 cd를 통해 자동 배포

### ci/cd
- 지속적 통합 (continuous integration) / 지속적 배포 (continuous deployment)
- 개발 시, 여러 사람이 개발한 프로젝트를 통합
- 빌드 전, 테스트를 통해 검증
---
- travis 방식
  - 깃허브에 해당 프로젝트를 push하는지 서버에서 간헐적으로 request 요청 (폴링)
  - 서버는 해당 코드를 다운, 단위 + 통합 테스트, 빌드를 거쳐 검증
  - 해당 프로젝트에 변경점이 발견되면 서버는 aws에 push
---
- jenkins 방식
  - 깃허브에 webhook을 설정, 새로운 push 발생시 서버에 hook 이벤트 전달
  - 서버는 해당 코드를 다운, 단위 + 통합 테스트, 빌드를 거쳐 검증
  - 해당 프로젝트에 변경점이 발견되면 서버는 aws에 push
---
- travis는 계속 물어봐야 되서 자원 낭비가 심함
- push 이벤트 발생할 때만 반응하는 jenkins가 더 좋음
- 하지만 github action으로 github내에서 처리 가능하여 해당 기능 사용

### iam
- identify access manager
- aws 사용자, 정책, 그룹, 역할을 이해해야 함
- 공장으로 예시를 들면 root는 사장으로 모든 공정을 관리
- 사용자는 직원으로 일부 공정일만 부여받아 수행
- 그룹은 직원이 많은 경우 각 부서별로 분리하여 관리하는데 비유
- 정책은 각 그룹은 맡은 공정일을 수행하기 위해 맡은 공정에 대한 권한의 모음
- 그룹으로 관리하면 사용자의 그룹만 변경하면 다른 역할을 부여할 수 있어 편리
- 역할 또한 권한의 모음이지만 사용자에게 부여하는 것이 아닌 서비스(운반책)에 부여

### rds 생성
- 보안 그룹을 생성, 이름은 security-group-aws-v5
- 22번 포트 (ssh), 3306번 포트 (mysql db 연결 포트)는 내 ip나 ec2 보안 그룹이면 접속 가능하게 설정
- 엘라스틱 빈스톡을 만들면 로드밸런서와 ec2에 대한 보안 그룹이 자동으로 만들어짐
- 로드 밸런서 보안 그룹은 80번 포트 (http)만 접근 가능
- ec2 보안 그룹은 22번 포트랑 로드 밸런서 보안 그룹만 접근 가능
- 순서상 rds 보안 그룹을 먼저 생성하므로 ec2 보안 그룹은 나중에 추가
---

- 이후 rds를 생성, 이름은 aws-v5-mysql
- db는 mysql, 템플릿은 프리티어로 설정
- db 인스턴스 식별자는 이름과 동일하게, 마스터 사용자 이름과 비번은 임의로 지정
- 보안 그룹은 앞서 만든 그룹으로만 설정
- 퍼블릭 엑세스를 허용하여 외부에서도 접근 가능하도록 지정
---

- 생성한 rds를 로컬 컴퓨터 (mysql workbench)에서 접속, 접속 이름은 aws-v5-mysql
- hostname은 rds의 엔드포인트, username과 password는 rds의 마스터 사용자 이름과 비번으로 설정
- 이후 데이터베이스랑 테이블 생성 및 설정
```text
-- 데이터베이스 생성
create database awsdb;

-- 데이터베이스 사용
use awsdb;

-- Book 테이블 생성
create table Book(
	id bigint auto_increment primary key,
    title varchar(255),
    content varchar(255),
    author varchar(255)
);

-- 인코딩 설정 확인
show variables like 'c%';

-- 데이터베이스 인코딩을 utf8mb4로 변경
alter database awsdb character set = 'utf8mb4' collate = 'utf8mb4_general_ci';

-- 테이블 확인
select * from Book;

-- 시간대 설정 확인
select @time_zone, now();
```
- 시간대를 수정하기 위해선 권한이 필요
- 이를 위한 파라미터 그룹 생성, 이름은 aws-v5-mysql-group
- 엔진 유형은 mysql community, 파라미터 그룹 패밀리는 mysql 8.0으로 설정
- db 파라미터 그룹 유형은 db parameter group으로 설정
- 생성 후 편집을 통해 time_zone 파라미터 값을 Asia/Seoul로 변경
- rds에 연결하기 위해 수정 -> 추가 사항 -> db 파라미터 그룹 -> 만들어논 커스텀 파라미터 그룹 설정
- 해당 과정에서 문제가 없다면 rds 재부팅 후 시간대를 다시 확인했을 때 서울 시간대로 성공적으로 변경 완료

### 엘라스틱 빈스톡 생성
- aws에서 엘라스틱 빈스톡 어플리케이션을 생성
- 환경 속성에서 RDS_HOSTNAME, RDS_PORT, RDS_DB_NAME, RDS_USERNAME, RDS_PASSWORD 속성 추가
- 각각 rds 엔드포인트, db 포트번호, db 이름, db 사용자, db 비밀번호를 값으로 입력
---

##### 로드 밸런서
- 서버 트래픽을 여러 서버에 분산
- 로드 밸런서 유형은 alb (application load balancer)로 설정
- 로드 밸런서 리스너와 프로세스를 통해 라우팅 설정, 여기선 디폴트로 진행
---

##### 오토 스케일링
- 서버 트래픽이 많을 경우 자동으로 ec2 서버 증설, 반대로 트래픽이 적을 때는 자동으로 서버 축소
- 오토 스케일링 환경 유형은 밸런싱된 로드로 설정
- 오토 스케일링 최소 최대 서버 수는 2 ~ 4대로 설정
---

- 롤링 업데이트 배포 정책은 변경 불가로 설정
- iam 역할 생성, 이름은 aws-elasticbeanstalk-ec2-role
- 아래의 권한 추가
```text
AWSElasticBeanstalkWebTier
AWSElasticBeanstalkWorkerTier
AWSElasticBeanstalkMulticontainerDocker
```
- elasticbeanstalk에 사용한 iam 역할 하나 더 생성, 이름은 aws-elasticbeanstalk-service-role
- 구성은 사용 사례에서 elasticbeanstalk을 선택하면 나오는 그대로 진행
- ec2 키페어도 생성
- 생성한 역할과 키페어는 서비스 엑세스에서 각자 맞는 위치에 선택

### 롤링
- 배포 전략으로 무중단 배포를 가능하게 함
- 세가지 전략이 있음
---

##### 한번에 모두
- ec2 서버에 새 프로젝트 배포시 가동하는 서버 모두 동시 중단
- 이후 모든 서버를 새 프로젝트로 업데이트하고 재가동
---

##### 추가 배치
- ec2 서버를 하나 새로 증설하여 새 프로젝트 배포
- 정해진 서버 수를 맞추기 위해 구 버전 배포 서버는 삭제
- 이 과정을 반복해 모든 서버에 새 프로젝트 배포
- 배포 중 에러 발생시 롤백을 진행해야 하는데 이때 모든 과정을 되돌림
- 다중의 서버를 보유하고 있다면 롤백시 시간도 많이 걸리고 사용 가능한 서버도 제한적이게 됨
- 자원을 절약할 수 있다는 장점도 있음
---

##### 블루 / 그린 -> 변경 불가
- 새 프로젝트를 업데이트할 ec2 서버를 증설
- 정상적으로 업데이트되면 기존 서버는 삭제하고 업데이트한 서버로 연결
- 이때 기존 서버를 블루, 새로 만든 서버를 그린이라 하여 블루 / 그린이라고도 함
- 새로 증설한 서버에서만 롤백하면 되므로 롤백이 쉬움
- 단, 업데이트 과정에서 증설된 서버를 유지하는 만큼 그 과정에서 자원 소모가 많음
- 반대로 말하면 업데이트 과정에서만 증설된 서버를 유지하면 되므로 전체적인 관점에선 자원 소모가 크지는 않음

### 전체 구성
- alb를 통해 http, https, 포트, 패킷 (http 헤더) 등의 요청을 받을 수 있음
- 또한 alb는 aws에서 ip를 뺀 dns 주소만 표시, 주소는 aws ec2의 로드밸런서에서 확인 가능
- ip 주소를 표시하지 않는 이유는 주기적으로 ip주소를 자동으로 할당받아 항상 달라지기에 굳이 표시하지 않음
---

- 요청을 받은 alb는 운영중인 2개의 ec2 서버로 요청을 전달
- ec2 서버는 80번 포트에 nginx로 구성된 리버스 프록시 서버로 요청을 받음
- 받은 요청은 jdk를 통해 설치된 스프링 서버로 보냄, 이때 5000번 포트로 통신
- 두 서버의 구성은 동일
---

- rds는 3306 포트로 열려 있고, 시큐리티 그룹으로 묶인 ec2 서버와 로컬 컴퓨터 ip만 접근 가능
---

- ip 주소가 주기적으로 바뀐다는 건 도메인 이름을 지정할 수 없다는 뜻이고 실제로 alb로는 서비스하기 어려움
- 그래서 추후 https나 패킷 처리는 못하지만 고정 ip는 받을 수 있는 nlb (network load balancer)를 만들어야 함
- 그럼 클라이언트로부터 맨 처음 요청을 받은 뒤 alb로 넘어가는 방식으로 바꿔 고정 ip로 도메인을 할당받을 수 있음

### github action
- 깃허브를 통한 배포
- 먼저 로컬 컴퓨터의 spring 프로젝트를 깃허브에 연결
```text
1. 코드를 변경해서 기능을 추가
2. git push 명령어를 통해 git에 프로젝트 업로드
3. git에 변경된 코드 반영
4. ci 서버에 우분투 설치
5. jdk 설치
6. git으로부터 코드 다운로드
7. 코드 테스트
8. build (실행 파일 생성)
```
- 3번에서 4번으로 넘어갈 때 트리거는 .github/workflows/*.yml
- 해당 yml 파일 내용을 바탕으로 ci 서버에서 4 ~ 8번 과정이 진행
- 처음 깃허브 배포시 깃허브 프로젝트의 상단바에 actions를 들어가 활성화해야 함

### elastic beanstalk 배포
- 엘라스틱 빈스톡에 *.jar 배포
- /var/app/current 경로에 /application.jar로 이름 변경후 저장
- 추가로 해당 폴더 경로에 /procfile도 자동 생성 및 실행
- procfile에는 해당 스크립트로 구성
```text
# 해당 jar 파일을 실행
java -jar application.jar
```
---

- 엘라스틱 빈스톡에 deploy.zip 배포 (이름은 임의로 지정)
- application.jar, procfile, .ebextensions/00-makeFiles.config으로 구성
- 00-makeFiles.config에서 /sbin 경로에 /appstart 파일을 생성
- 리눅스에서 bin이나 sbin에 있는 파일은 모든 경로에서 실행 가능
- 스크립트에서 mode는 리눅스의 접근 권한 설정과 동일
- 내용은 java -jar application.jar과 동일
- procfile 파일에는 config를 통해 만든 appstart 실행
- procfile은 jar 파일 배포와 같이 자동으로 실행
---

- deploy 파일에 AWS_ACCESS_KEY, AWS_SECRET_KEY는 환경 변수로 설정
- 환경 변수는 깃허브 저장소에서 setting -> secrets and variables -> actions -> repository secrets에서 생성

### 한글 입력 오류 해결
- db 인코딩 설정 문제로 한글 입력시 오류가 발생할 수 있음
- aws rds -> 파라미터 그룹에서 생성한 그룹에서 파라미터 편집
- character_set으로 시작하는 파라미터 값 전부 utf8mb4로 변경
- 변경 완료후 db 재부팅
- 하지만 테이블은 인코딩 설정 전에 만들어 진거라 설정이 안먹히므로 drop 후 새로 생성

### 로드밸런서 고정 ip 설정
- 앞서 말했든 고정 ip 사용을 위해선 nlb (network load balancer)를 사용해야 함
- nlb는 네트워크 계층에서 로드 밸런싱을 하여 요청을 처리
---

- 먼저 ec2에서 탄력적 ip를 생성하여 ip를 할당 받음
- 이후 로드 밸런서에서 생성 후 nlb 선택
- 이름은 aws-v5-nlb, 매핑은 하나만 선택해서 앞서 만든 탄력적 ip 사용
- 보안 그룹은 여태 만든 그룹 다 선택
- 리스너 및 라우터 설정에서 80번 포트의 그룹을 선택
- 하지만 alb에 대한 그룹을 만들지 않아 대상 그룹 생성을 클릭하여 이름이 aws-v5-alb의 기본 구성이 alb인 그룹 생성
- 만든 그룹을 80번 포트 그룹으로 설정 후 생성
---

- 만약 도메인 설정을 하고 싶으면 가비아 같은 도메인 등록 사이트에서 도메인을 구매
- 도메인 사이트에서 해당 도메인 dns 관리를 통해 탄력적 ip를 dns로 설정