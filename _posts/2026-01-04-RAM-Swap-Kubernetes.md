---
title: "RAM, Swap, 그리고 Kubernetes에서의 메모리 관리"
date: 2026-01-04
categories: [Kubernetes, 인프라]
tags: [RAM, Swap, Kubernetes, Memory, Node, Pod, Scheduler, AutoScaler]
---

## Reference

https://m.blog.naver.com/sjc02183/221998493348

## 문제 상황

- 와플 인프라 클러스터에 있는 운영체제를 ARM64 기반으로 바꾸고 swap이 가능하게 하였으나
- swap이 미적용되고 노드수가 계속 늘어나는 상황이 발생함
- 원인 파악과, 기본적인 SWAP / Ram / 쿠버네티스에서의 노드가 개념이 뭔지 이해하고자 작성한 문서

## Ram (Random Access Memory)

컴퓨터에서 CPU는 프로그램과, Data들을 제어하기 위해서는 RAM에 올라간 프로그램에만 접근이 가능

DISK 에 있는 정보를 바로 접근할수 없다.

컴퓨터는 부팅할때마다 하드드라이브 → Ram으로 핵심 운영체제 프로그램(kerenel, init, systemd)을 복제한다

![Memory Hierarchy](/assets/img/posts/2026-01-04-RAM-Swap-Kubernetes/image.png)

RAM은 Random Access Memory로 컴퓨터가 사용하는 프로그램과 데이터를 저장해두는 중간 계층이며,

L1 Cache/ L2 Cache 등등이 SRAM 이며

Main Memory 역할을 하는 RAM을 DRAM이라고 하며 일반적으로 RAM이라하면 여기를 명칭함

컴퓨터를 끄면 RAM에 있던 메모리는 모두 소멸한다. 

CPU ↔ Cache ↔ RAM ↔ DISK

### SSD / HDD

Ram은 CPU와 영구저장장치인 (ssd/hdd) 사이의 latency 차이를 완화하기 위한 메모리 계층임

#### 🔹 HDD (Hard Disk Drive)

- 자기 디스크 회전
- **느림**
- 대용량
- 저렴
- 기계식

#### 🔹 SSD (Solid State Drive)

- 플래시 메모리
- **HDD보다 훨씬 빠름**
- 비휘발성
- 여전히 RAM보다는 수백~수천 배 느림

## Swap

swap은 RAM의 공간이 부족할때, 하드디스크의 파티션을 빌려와서 가상공간을 만들어 주는 것이다.

Ram의 일부 내용을 DISK(SSD/HDD) 에 임시로 밀어내는 방법프로세스 

자주 안쓰는 메모리 페이지를 디스크로 밀어내서 (swap)

Ram의 공간을 마련해주기 위한 방법임

Swappiness 란 용어는 얼마나 Swap을 잘 활용하고 있냐를 나타내는 지표

## Node, Swap

쿠버네티스 클러스터에서 swap을 사용함에도 노드수가 늘어난다는 것은

Node의 할당가능한 메모리양 (allocatable memory) 가 부족해졌을때 발생하는데

이는 Pod Requests ≥ Node Allocatable memory 일때 스케줄링이 불가하므로 노드를 하나더 생성한다.

```yaml
resources:
  requests:
    memory: 128Mi
  limits:
    memory: 512Mi
```

### Requests.memory
- Pod가 최소한 필요하다고 선언한 메모리
- Kubernetes 스케줄러가 보는 값으로 Node에 예약되어있는 것
- 현재까지의 requests.memory의 합과, 추가적으로 필요한 양 vs node allocatable memory를 비교하여 가능하다면 노드배치를 하는 것.

### Limits.memory
- Pod가 최대 쓸수있는 메모리
- 초과하면 OOM Kill 발생하며 pod가 죽는다

OS 컨테이너는 메모리를 사용하느 만큼만 할당해주는데, 필요해질때마다 할당량을 늘려주는 방식임.

메모리 사용량 만큼 RAM을 사용하며, limit에 명시된 메모리를 초과할 경우 OOM이 발생한다

## 스케쥴러, AutoScaler - 노드의 관리자 역할

- 앞서 Node의 리소스 할당을 (request, limit) 알아봤다
- 그렇다면 노드를 늘려주고, 메모리를 할당해주는 주체인 스케줄러를 알아보자

스케줄러는 Pod 생성 요청이오면 **어느 Node에 올릴지** 결정한다

이때 확인하는 것이 `requests.memory`로 Pod가 최소한 필요한 메모리양을 기준으로 스케줄링을 예약한다

![Kubernetes Cluster](/assets/img/posts/2026-01-04-RAM-Swap-Kubernetes/image 1.png)

스케쥴러는 **Control Plane**에서 Cluster 전반을 관리하는 역할을 한다

이때, $\sum$ requests vs Node's allocatable memory를 비교했을때, requests 의 합이 훨씬커서 

파드 할당이 안된다면

노드에 할당가능한 메모리가 없다는 뜻이므로 노드 수를 늘려야한다.

이 역할을 해주는 것이 **AutoScaler**이다 

### 결론

Node에 스케쥴링 (남은 공간보면서 pod 할당하기) 가 안되고 Scale-out 해야할때면 노드수가 늘어나는것

Node 할당해주는건 스케쥴러

## 문제의 원인 : 왜 swap이 작동을 안했던걸까?

```yaml
resources:
  requests:
    memory: 128Mi
  limits:
    memory: 128Mi
```

- OOM 기준인 128Mi와 최소 사용량이 동일하여
- Swap을 사용하고자 한다면 `request` < `limits`  으로 메모리 사용량을 설정해야한다

**질문** : limit에 도달하면 OOM이 일어날텐데 이때, 노드가 늘어난는건가/ 파드가 늘어나는건가?

**답** : 아무것도 자동으로 늘어나지 않고, Pod가 OOM Kill 된다.

requeest와 limit이 동일한건 상한과 하한을 동일하게 설정하는 것이다.

즉, 실제 메모리 사용량이 limit을 넘으면 Pod 내부에서  oom kill이 발생하는 것이지 

Node나 Pod가 늘어나는건 별개의 원인이다.

- **Pod** → Controller가 결정
- **Node** → Cluster Autoscaler가 결정 (Pod가 pending 상태일떄)
