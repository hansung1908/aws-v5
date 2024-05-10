# aws-v5

- 시작 전 aws 리전 설정을 서울로 꼭 설정

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