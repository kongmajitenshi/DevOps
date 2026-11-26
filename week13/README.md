# 13주차

## GitOps
- Git + Operations, Git으로 운영한다는 뜻. Git이 곧 진실의 의미는 Git에 올려둔 버전에 맞추라는 의미. 
- 클러스터에 있어야 할 상태를 Git에 적어두고, 클러스터는 항상 Git과 같게 맞춤
- 지난주까지는 CI가 이미지까지만 올리고 배포(`kubectl set image`)는 손으로 했는데, 이걸 Argo CD가 대신 해줌

## Push vs Pull
- Push: 바깥의 CI가 클러스터에 직접 `kubectl apply` 함. CI한테 클러스터 접속 권한을 줘야 하고, 내부망에 있는 클러스터엔 접근하기 어려움
- Pull: 클러스터 안에 있는 Argo CD가 Git을 지켜보다가 바뀌면 가져와서 적용함. 안에서 밖으로 나가는 거라 바깥에 권한 줄 필요가 없음
- Git 히스토리가 곧 배포 기록이 되고, 롤백은 `git revert`로 하면 됨

## Argo CD
- Application YAML에 어느 저장소(`repoURL`), 어느 브랜치(`targetRevision`), 어느 폴더(`path`)를 어느 네임스페이스에 배포할지 적음
- `automated`: Git이 바뀌면 알아서 적용함. 없으면 Sync 버튼을 눌러야 함
- `prune`: Git에서 지운 리소스를 클러스터에서도 지움
- `selfHeal`: 누가 손으로 바꾸면 Git 상태로 되돌림
- Application YAML은 `guestbook/k8s/` 밖(저장소 루트)에 둬야 함. 안에 두면 Argo CD가 자기 설정까지 관리하게 됨
- Git 변경은 2~3분 간격으로 확인하고, 클러스터 변경은 바로 감지함

## 화면 읽기
- Sync: Synced(클러스터 = Git) / OutOfSync(다름)
- Health: Healthy / Progressing / Degraded / Missing
- Synced인데 Degraded면 Git대로 정확히 배포했는데 Git 내용이 틀린 것. 예: 없는 이미지 태그

## Drift
- 클러스터의 실제 상태가 Git의 선언에서 벗어난 것. `kubectl edit`나 `scale`로 급하게 고치고 잊어버리면 생김
- `kubectl scale --replicas=10` 해도 Git에 4라고 적혀 있으면 selfHeal이 다시 4로 돌려놓음
- Git에 없는 변경은 인정 안 됨. 바꾸고 싶으면 PR을 올려야 함
- `kubectl rollout undo`로 되돌려도 selfHeal 때문에 다시 Git 버전으로 돌아감. 되돌리기는 `git revert`로

## 이미지 배포 흐름
- 코드 PR 머지 → CI가 이미지 게시 → `deployment.yaml`의 `image:` 태그를 바꾸는 PR 머지 → Argo CD가 감지해서 롤링 업데이트
- 이미지만 GHCR에 올린다고 Git의 `image:`가 바뀌지는 않음. 그래서 태그 바꾸는 PR이 꼭 필요함
- 이 PR을 머지하는 게 사실상 배포 승인

## 배포 전략
- Recreate: 다 내리고 새로 띄움. 그 사이 서비스가 중단됨
- Rolling Update(기본값): 하나씩 교체함. 잠깐 두 버전이 섞임
- Blue/Green: 새 버전을 다 띄워두고 Service selector만 바꿔서 한 번에 전환함. 문제 생기면 바로 되돌릴 수 있지만 자원이 두 배로 듦
- Canary: 일부한테만 새 버전을 먼저 보내고 괜찮으면 늘림. 새 버전 안전한지 확인
- A/B 테스트: 배포 전략이 아니라 실험. 뭐가 더 좋은지 비교 (버튼 색 같은 거)

## 모니터링
- 배포한다고 끝X. 운영 남음. 측정X = 개선X
- 메트릭: 숫자(CPU 80%) / 로그: 텍스트(ERROR: DB 연결 실패) / 트레이스: 요청이 어디를 거쳐 갔는지
- Prometheus가 `/metrics`를 주기적으로 가져가서 수집함(Pull 방식), Grafana가 그래프로 보여줌, Alertmanager가 알림을 보냄
- `kubectl top pods`로 CPU/메모리 확인. `1m`은 CPU 1개의 0.1%, `28Mi`는 약 30MB. requests/limits와 비교해서 봄
- 알림은 사람이 뭔가 해야 할 때만 울리게 해야 함. 순간 튀는 값마다 울리면 아무도 안 봄

## 느낀 점
- 
