# gitops-sample-manifest

GitOps CI/CD 파이프라인 실습 프로젝트의 **배포 매니페스트 레포**입니다.
Helm 차트로 애플리케이션 배포를 정의하고, ArgoCD가 이 레포를 감시하며
kubeadm 클러스터에 자동 배포(GitOps)합니다.

애플리케이션 소스는 [gitops-sample-app](https://github.com/gyu2001/gitops-sample-app) 참고.

---

## 전체 아키텍처

```
[gitops-sample-app] --(GitHub Actions)--> GHCR 이미지 push
                                               |
[이 레포] Helm values의 이미지 태그 업데이트 ---+
       |
       | (ArgoCD가 이 레포를 감시)
       v
  ArgoCD --- 자동 동기화(auto-sync) ---> kubeadm 클러스터(EC2)
```

## GitOps 흐름

1. 앱 레포에서 이미지가 빌드되어 GHCR에 새 태그로 올라감
2. 이 레포의 `sample-app/values.yaml`에서 이미지 태그를 갱신
3. ArgoCD가 이 레포의 변경을 감지
4. auto-sync 정책에 따라 클러스터에 새 버전을 자동 배포
5. 실제 서비스에 반영 (버전 변경 확인)

## 인프라 환경

| 항목 | 값 |
|---|---|
| 클러스터 | kubeadm 기반 멀티 노드 (Control-plane 1, Worker 1) |
| 실행 환경 | AWS EC2 (m7i-flex.large) |
| CNI | Calico |
| 배포 도구 | ArgoCD (auto-sync, self-heal) |
| 패키징 | Helm |

## 사용 기술

`Kubernetes` `ArgoCD` `Helm` `kubeadm` `Calico` `AWS EC2`

---

## 트러블슈팅 — ArgoCD 배포 실패 (DNS 조회 타임아웃)

ArgoCD가 애플리케이션을 배포하려 할 때 아래 오류가 반복 발생했습니다.

```
ComparisonError: Failed to load target state: ... rpc error: code = Unavailable
desc = dns: A record lookup error: lookup argocd-repo-server on 10.96.0.10:53:
dial udp 10.96.0.10:53: i/o timeout
```

**원인 추적 과정**
1. ArgoCD 파드, CoreDNS, Calico 모두 `Running` 상태로 인프라 자체는 정상
2. 클러스터 내부에서 DNS 조회를 직접 테스트 → 서비스 이름 변환 실패(`can't resolve`)
3. 오류 메시지의 `dial udp ... i/o timeout`에 주목 → **UDP 통신**이 막힌 정황
4. EC2 보안그룹 인바운드 규칙 확인 → 노드 간 통신 규칙이 **"모든 TCP"로만** 열려 있었음

**원인**
Calico는 노드 간 파드 네트워크에서 TCP뿐 아니라 **UDP(DNS 질의 53번 포트)와
IP-in-IP 프로토콜**을 사용하는데, 보안그룹이 TCP만 허용하고 있어
노드를 넘어가는 DNS 질의(UDP)가 차단되어 있었습니다.

**조치**
보안그룹의 노드 간 통신 규칙을 "모든 TCP" → **"모든 트래픽(All traffic)"**으로
변경하여 UDP·IP-in-IP를 포함한 노드 간 통신을 허용했습니다.

**결과**
DNS 조회가 정상 동작(`argocd-repo-server` → ClusterIP 정상 변환)하고,
ArgoCD의 ComparisonError가 해소되어 애플리케이션이 정상 배포(Synced/Healthy)되었습니다.

> 배운 점: 파드/서비스가 모두 Running이어도, 노드 간 네트워크 계층(L3/L4)이
> 막히면 클러스터 DNS가 실패할 수 있다. "Running = 정상"이 아니라 실제 통신을
> 검증해야 한다는 것을 확인했다.
