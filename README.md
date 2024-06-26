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