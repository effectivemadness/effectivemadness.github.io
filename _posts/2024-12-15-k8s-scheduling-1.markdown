---
title:  " Kubernetes 스케줄링 정리 1 - Node 관점"
date:   2024-12-15 12:20:50+0900
categories: [Cloud]
tags: [tech, kubernetes]
---
최근 Karpenter를 테스트 하고 있는데요. 테스트 과정 중 Karpenter가 분명히 노드 용량이 부족하지 않음에도 계속 새 노드를 띄우려 하더라구요. 카펜터 설정이 잘못된건가, 뭘 잘못 설치했나 하고 알아보던 중 원래 있던 노드에는 스케줄링이 될 수 없어 계속 그런 동작을 시도하더라구요. 그 뒤로, 카펜터 Pod을 특정 노드에서만 띄우도록 설정하는 과정에서 NodeAffinity를 설정했는데 이 과정을 지나며 Kubernetes에서 Pod의 스케줄링을 어떻게 하는지 잘 모르는것 같아 정리해보려 합니다.

## Kubernetes Scheduler

> In Kubernetes, *scheduling* refers to making sure that [Pods](https://kubernetes.io/docs/concepts/workloads/pods/) are matched to [Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/) so that [Kubelet](https://kubernetes.io/docs/reference/generated/kubelet) can run them. - Kubernetes Docs

Kubernetes에서 보통 스케줄링이라 하면, 작업의 최소 단위인 Pod을 어떤 노드에 올릴지 정하는 과정을 말하는데요, 기본적으로 Kubernetes에서는 kube-scheduler가 default scheduler로 존재합니다. kube-scheuder는 각 Pod의 제약사항을 확인해 적절한 노드에서 돌아갈 수 있게 스케줄링을 진행합니다. Pod이 "실행 가능한"노드를 먼저 찾고, 그 "실행 가능한"노드들을 점수를 매긴 후 가장 높은 점수를 가진 노드를 고릅니다.

kube-scheduler가 진행하는 이 과정은 크게 두 가지로 나뉘어집니다.

1. Filtering
2. Scoreing

Filtering 과정에서는 Pod이 "실행 가능한"노드인지 확인합니다. 예를 들어, `PodFitsResource` 필터는 노드에 Pod이 소모 가능한 자원이 남아있는지 확인합니다. Filtering 과정이 지난 뒤에는 Pod이 "실행 가능한" 노드만 남게 됩니다. 

Scoreing 과정에서는 Filtering 과정을 거친 노드들을 Pod 실행에 가장 적절한 순서대로 점수를 매기게 됩니다. 이후, kube-scheduler는 점수가 높은 순서대로 Pod을 노드에 할당합니다. 만약 점수가 같은 노드들이 있다면 랜덤으로 할당합니다.

scheduler의 Filtering/Scoreing 과정을 설정하기 위해서 두가지 방법을 지원합니다.

1. Scheduling Policy : Filtering 과정에서 사용하는 기준(*Predicates*) 과 Scoring 과정에서 사용하는 우선순위(*Priorities*)를 설정합니다.
2. Scheduling Profiles : 각 스케줄링 단계를 구현하는 플러그인들을 설정합니다.

## Pod에 노드 할당하기

Pod을 특정 노드에 반드시 할당해야 하거나, 되도록 특정 노드에서 실행하도록 해야 하는 경우가 있습니다. 이 경우, 4가지 방법을 사용할 수 있습니다.

- 노드의 라벨을 사용한 nodeSelector
- 어피니티, 안티어피니티(Affinity, Anti-affinity)
- nodeName 필드
- Pod Topology spread constraints

이 네가지 방법 중, 마지막 Pod Topology spread constraints를 제외한 방법을 확인해봅니다.

### 노드 라벨을 사용하는 nodeSelector

다른 Kubernetes 자원과 비슷하게, 노드들도 라벨을 가지고 있습니다. 직접 노드에 라벨을 붙일 수도 있고, Kubernetes가 자동으로 붙인 라벨들도 있습니다. 이 노드 라벨들을 사용해 직접 Pod이 실행될 노드를 선택할 수 있습니다.

Pod spec 내 `nodeSelector` 필드를 노드의 라벨과 함께 지정하면, Kubernetes 스케줄러는 지정된 노드에만 Pod을 할당합니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    disktype: ssd #disktype라벨 값이 ssd인 노드에만 스케줄링
```

### 어피니티, 안티어피니티(Affinity, Anti-affinity)

affinity는 "친밀감, 관련성"으로 해석되는 단어인데요. `nodeSelector`와 비슷하게 라벨을 기반으로 노드를 선택할 수 있습니다. 하지만, affinity의 경우 조금 더 유연하게 선택 가능한 연산자를 제공합니다. 추가적으로, 이 규칙을 꼭 지키지 않아도 된다고 설정을 할 수 있고, 다른 Pod이 있는지/없는지에 대해서도 스케줄링이 가능합니다.

#### **Node affinity**

노드 어피니티의 경우, nodeSelector와 비슷합니다. Pod이 실행될 특정 노드를 선택할 수 있는데요. 두가지 종류로 설정할 수 있습니다.

- `requiredDuringSchedulingIgnoredDuringExecution` : 스케줄링때만 조건을 확인하고, 한번 실행된 후에는 조건을 확인하지 않습니다. - nodeSelector에 문법이 추가된 느낌입니다.
- `preferredDuringSchedulingIgnoredDuringExecution` : 스케줄링때 조건을 확인하지만, 조건에 맞는 노드가 없다면 그냥 무시하고 스케줄링합니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution: # 아래 조건은 무조건 맞아야 스케줄링 합니다
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - antarctica-east1
            - antarctica-west1
      preferredDuringSchedulingIgnoredDuringExecution: # 아래 조건은 맞으면 좋지만, 맞는 노드가 없으면 그냥 스케줄링합니다.
      - weight: 1 # 각 규칙의 Weight는 합산하여 스케줄링 시 우선순위를 산정하는 score가 됩니다.
        preference:
          matchExpressions:
          - key: label-1
            operator: In
            values:
            - key-1
      - weight: 50
        preference:
          matchExpressions:
          - key: label-2
            operator: In
            values:
            - key-2
  containers:
  - name: with-node-affinity
    image: registry.k8s.io/pause:2.0
```

어피니티를 설정할 때, 연산자는 `In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt` , `Lt` 를 사용할 수 있습니다.

#### **Pod간 affinity, anti-affinity**

어피니티를 설정할 때, 특정 Pod이 실행되는지에 따라서도 조건을 넣을 수 있습니다. 예를 들어, 노드에 데이터를 공유하는 Pod 끼리 같이 실행되어야 한다면 이 조건을 설정해 한 노드에서 실행할 수 있도록 설정할 수 있습니다. inter-pod affinity, anti-affinity 규칙은 대개, "이 Pod은 어떤 조건 Y에 맞는 한개 이상의 Pod이 실행되(고 있지 않`anti-affinity의 경우`)는 X에서 실행되어야 함"으로 표현 가능합니다. 조건 Y는 라벨(+네임스페이스)로 지정 가능합니다. X는 토폴로지 도메인 - 노드, 랙, CSP 내 존/리전이 될 수 있습니다. X는 `topologyKey`를 사용해 선택하게 됩니다.

노드 어피니티와 비슷하게, 두가지 종류로 설정할 수 있습니다.

- `requiredDuringSchedulingIgnoredDuringExecution`
- `preferredDuringSchedulingIgnoredDuringExecution`

작동 방식은 비슷하지만, 두 개의 용례를 살펴보면, `requiredDuringSchedulingIgnoredDuringExecution` 조건의 affinity를 활용해 위에 언급한 한 노드에서 실행되어야 하는 여러개의 Pod들을 묶어 실행할 수 있습니다. `preferredDuringSchedulingIgnoredDuringExecution` 조건의 anti-affinity를 활용하면 Pod을 클러스터 내 노드들에 최대한 골고루 퍼뜨려 실행시킬수도 있습니다. 예시를 보겠습니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-affinity
spec:
  affinity:
    podAffinity: # Pod 어피니티 확인
      requiredDuringSchedulingIgnoredDuringExecution: # 스케줄링 시 무조건 확인
      - labelSelector:
          matchExpressions: # security:S1 인 조건
          - key: security
            operator: In
            values:
            - S1
        topologyKey: topology.kubernetes.io/zone # security:S1 인 Pod이 한개라도 떠있는 Zone 내 노드에 스케줄링
    podAntiAffinity: # Pod 안티어피니티 확인
      preferredDuringSchedulingIgnoredDuringExecution: # 스케줄링시 고려하지만, 안맞아도 스케줄링
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions: # security:S2 인 조건
            - key: security
              operator: In
              values:
              - S2
          topologyKey: topology.kubernetes.io/zone # security:S2 인 Pod이 떠있는 Zone 내 노드에는 스케줄링 피함
  containers:
  - name: with-pod-affinity
    image: registry.k8s.io/pause:2.0
```

예시에서는 Affinity로 security:S1인 팟을 스케줄링 시 무조건 확인하는 조건으로 넣었고, topologyKey는 `topology.kubernetes.io/zone` 을 지정했습니다. 이 경우, security:S1 라벨이 붙은 Pod이 한개라도 떠 있는 Zone 내 노드를 선택해 스케줄링 할 수 있습니다.

Anti-affinity의 경우에는, security:S2인 팟을 스케줄링시 고려하는 조건으로 넣었고, topologyKey는 동일하게 `topology.kubernetes.io/zone` 을 지정했습니다. 이 경우에는 security:S2 인 Pod이 떠있는 Zone 내 노드에는 스케줄링을 피하게 됩니다. 이 조건을 만족하지 못하더라도, 다른 방법이 없을 경우 스케줄링이 진행됩니다.

추가적으로, affinity와 anti-affinity의 경우 모두 `topologyKey` 가 필요합니다. 특별히, anti-affinity 를 `requiredDuringSchedulingIgnoredDuringExecution` 로 설정한 경우, `topologyKey` 는 `kubernetes.io/hostname` 만 사용 가능합니다. affinity나 anti-affinity를 설정할 때, `labelSelector` 대신 `namespaceSelector`도 사용가능합니다.

### NodeName

`nodeName` 은 어피니티나 `nodeSelector` 보다 더 명시적으로 노드를 선택하는 방법입니다. 해당 필드가 비어있지 않다면, 스케줄러는 Pod을 무시하고 그 노드에 바로 배치하려 합니다. `nodeName` 은 `nodeSelector` 나 Affinity를 무시하고 작동합니다. 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  nodeName: kube-01
```

이 방법은 아주 명확하지만 다양한 한계를 가지고 있습니다.

- 그 이름의 노드가 존재하지 않다면 Pod은 실행되지 않습니다. 특정 조건 아래에서는 삭제될 수 있습니다.
- 그 이름의 노드가 해당 Pod을 배치하기에 적은 자원을 가지고 있다면 Pod 실행이 실패하고 그 이유(OutOfmemroy, OutOfcpu)를 확인할 수 있습니다.
- 클라우드 환경에서의 노드 이름은 예측가능하지 않을 수 있습니다.

---

노드 관점에서의 Pod 스케줄링의 일부를 정리해봤습니다. Deployment Spec에 있는 템플릿으로만 보던 개념들인데, 텍스트로 정리해보니 이해가 잘 되는 것 같습니다. 며칠동안 Pod Topology spread constraints 과 Taint/Toletration 까지 마저 보고 정리하는게 목표입니다. 자원/라벨을 활용해서 선택하고 제어하는 구조를 꾸준히 보다보니, Kubernetes는 확장 가능하게 잘 설계가 되었다는 생각이 드네요.