# Step 1. 컨테이너 = 그냥 리눅스 프로세스

> "컨테이너"라는 커널 객체는 없다. **namespace(무엇을 볼 수 있나) + cgroup(얼마나 쓸 수 있나) + rootfs(어떤 파일시스템인가)** 를 조합한 평범한 프로세스일 뿐이다.

root 권한의 Linux 환경에서 실행한다 (VM 권장).

## 1-1. Namespace: 격리된 "시야"

```bash
# 현재 쉘의 namespace 목록
ls -l /proc/$$/ns
```

```bash
# PID + UTS + mount namespace를 새로 만들고 쉘 실행
sudo unshare --pid --fork --mount-proc --uts bash
```

새 쉘 안에서:

```bash
hostname my-container && hostname   # 호스트의 hostname은 바뀌지 않음 (UTS ns)
ps aux                              # 내 bash가 PID 1 (PID ns)
```

**확인할 질문**
- 다른 터미널(호스트)에서 `ps aux | grep bash` 하면 이 bash의 PID는 몇인가? → 같은 프로세스, 다른 PID 번호.
- `--fork`를 빼면 왜 이상하게 동작할까? (힌트: `unshare` 자신은 PID ns에 들어가지 않고, 그 *자식*부터 들어간다)

## 1-2. Cgroup: 자원 제한

```bash
# cgroup v2 가정
sudo mkdir /sys/fs/cgroup/lab
echo "50000 100000" | sudo tee /sys/fs/cgroup/lab/cpu.max   # CPU 50%
echo $((100*1024*1024)) | sudo tee /sys/fs/cgroup/lab/memory.max  # 100MB

# 현재 쉘을 cgroup에 넣기
echo $$ | sudo tee /sys/fs/cgroup/lab/cgroup.procs

# CPU 부하 → 다른 터미널에서 top으로 ~50%인지 확인
yes > /dev/null
```

`docker run --cpus=0.5 --memory=100m` 이 결국 이 파일들에 값을 쓰는 것이다.

## 1-3. Rootfs + OverlayFS: 이미지 레이어

```bash
mkdir -p /tmp/lab/{lower,upper,work,merged}
# lower = 이미지 레이어(읽기 전용), upper = 컨테이너의 쓰기 레이어
docker export $(docker create alpine) | tar -C /tmp/lab/lower -xf -   # docker 없으면 alpine minirootfs tar 사용

sudo mount -t overlay overlay \
  -o lowerdir=/tmp/lab/lower,upperdir=/tmp/lab/upper,workdir=/tmp/lab/work \
  /tmp/lab/merged

echo hello | sudo tee /tmp/lab/merged/new-file
ls /tmp/lab/upper      # 변경 사항은 upper에만 생긴다 (copy-on-write)
```

## 1-4. 합치기: 손으로 만든 컨테이너

```bash
sudo unshare --pid --fork --mount --uts --ipc --net \
  chroot /tmp/lab/merged /bin/sh -c 'mount -t proc proc /proc; hostname box; exec /bin/sh'
```

안에서 `ps`, `hostname`, `ip a`, `ls /` 를 확인해 보자. 이게 사실상 `docker run -it alpine sh` 의 본질이다.
(실제 런타임은 `chroot` 대신 더 안전한 `pivot_root`를 쓴다 — Step 3에서 runc 코드로 확인)

## 체크리스트 (다음 단계로 가기 전)

- [ ] namespace 7~8종(pid, mnt, uts, ipc, net, user, cgroup, time)이 각각 무엇을 격리하는지 설명할 수 있다
- [ ] cgroup이 "제한"이고 namespace가 "격리"라는 차이를 설명할 수 있다
- [ ] overlayfs의 lower/upper/merged 관계와 copy-on-write를 설명할 수 있다
- [ ] `chroot`와 `pivot_root`의 차이를 찾아봤다

다음: Step 2 — Go로 위 과정을 `syscall.SysProcAttr{Cloneflags: ...}` 로 직접 구현하기.
