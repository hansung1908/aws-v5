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
- 인바운드 규칙은 80번 포트 (http), 22번 포트 (ssh) 오픈
- 3306번 포트 (mysql db 연결 포트)는 내 ip나 같은 시큐리티 그룹이면 접속 가능하게 설정
- 같은 시큐리티 그룹 접속 설정은 보안 그룹 생성 후 추가해야 함
- 좀 더 정확히 하면 ec2는 22, 80번 포트만 rds는 22, 3306포트만 접근 가능하게 보안 그룹을 분리해야 하지만 간단하게 하기 위해 통합
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