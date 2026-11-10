# Laboratory: Container → Kubernetes 소스 레벨 학습

최종 목표: **Kubernetes 소스 코드 수준 이해**.
전략: 아래에서 위로(bottom-up). 각 레이어는 "직접 해보기 → 해당 소스 읽기" 순서로 진행한다.

```
kubectl ─▶ kube-apiserver ─▶ (etcd)
                 │
              kubelet ──CRI(gRPC)──▶ containerd ──▶ containerd-shim ──▶ runc ──▶ Linux kernel
                                         ▲                                         (namespaces,
docker CLI ─▶ dockerd (moby) ────────────┘                                          cgroups,
                                                                                    overlayfs)
```

핵심 포인트: **K8s는 Docker(moby)를 거치지 않는다.** kubelet은 CRI로 containerd를 직접 호출한다.
그래서 moby는 "컨테이너 엔진이 어떻게 생겼나"를 보는 좋은 교재이지만, K8s로 가는 본선은
`Linux 커널 → runc → containerd → kubelet` 이다.

## 로드맵

| Step | 주제 | 직접 해보기 | 읽을 소스 |
|---|---|---|---|
| 1 | Linux primitives | `unshare`, cgroup fs, overlayfs로 손으로 컨테이너 만들기 | — |
| 2 | Go로 미니 컨테이너 | `clone` 플래그로 ~100줄짜리 런타임 작성 | — |
| 3 | OCI + runc | `runc spec` → `config.json` 수정 → `runc run` | `opencontainers/runc` `libcontainer/` |
| 4 | moby (dockerd) | `docker run` 한 번의 호출을 코드로 추적 | `moby/moby` (아래 참고) |
| 5 | containerd | `ctr`로 이미지/컨테이너/태스크 다루기 | `containerd/containerd` (`pkg/cri` 포함) |
| 6 | CRI | `crictl`로 Pod sandbox 생성 | `kubernetes/cri-api` |
| 7 | kubelet | Pod 하나가 실제 프로세스가 되기까지 추적 | `kubernetes/kubernetes` `pkg/kubelet/` |
| 8 | Control plane | apiserver / scheduler / controller 패턴 | `staging/src/k8s.io/client-go` (informer) 등 |

각 단계 노트: [`container-deep-dive/`](container-deep-dive/)

## Step 4 미리보기: moby에서 `docker run`의 경로

(moby `5dc4ce2` 기준, `git clone --depth 1 https://github.com/moby/moby.git`)

1. `daemon/server/router/container/container.go:42` — `POST /containers/create`, `/start` 라우트
2. `daemon/create.go:52` — `ContainerCreate`: 컨테이너 메타데이터/레이어 준비
3. `daemon/start.go:82` — `containerStart`
4. `daemon/start.go:167` — `createSpec` 호출 → **OCI spec (config.json) 생성**
5. `daemon/oci_linux.go:228` — `WithNamespaces`: namespace 설정이 spec에 들어가는 곳
6. `daemon/internal/libcontainerd/remote/client.go:119,149,240` — containerd에 `NewContainer` → `NewTask` → `Start`
7. 그 다음은 containerd → shim → runc (Step 3, 5에서 이어짐)

즉, dockerd가 하는 핵심 일은 "**사용자 옵션을 OCI spec으로 번역해서 containerd에 넘기는 것**"이다.
Step 1~3을 먼저 해두면 `oci_linux.go`의 코드가 전부 "아, 그 커널 기능"으로 읽힌다.
