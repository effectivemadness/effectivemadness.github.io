---
title:  "Kubernetes 스케줄링 정리 4 - Taints/Toleration"
date:   2025-02-15 18:29:13+0900
categories: [Cloud]
tags: [tech, kubernetes]
---
스케줄링 정리의 마지막입니다. Taint/Toleration은 CKA에서도 많이 다루는 주제이고, 실제로 많은 K8s 작동에서 사용되는 기능이라 지나가며 보셨을 것 같아요. 일단 사전에서 각 단어의 정의를 찾아보았습니다.

> Taint: 더럽히다, 오염시키다
>
> Toleration: 용인, 관용 (동사형 Tolerate: 용인하다, 참다, 견디다)

뭔가 Taint는 더럽히고, Toleration은 그걸 견딘다... 로 이해되는데요. 그냥 음차해서 문서를 봤을때는 "테인트", "톨러레이션" 이렇게만 봤는데, 의미를 보고 어렴풋이 동작을 생각해보면 이해가 확 되는 작명입니다.(이하, Taint는 묻히다, 칠한다로 표현하겠습니다.)

## 정의

Node Affinity는 파드가 어떤 노드들에 붙는지를 정의하는 속성인데요, Taint는 그 반대입니다. Taint는 노드가 어떤 파드들을 거부/밀어내도록 정의하는 속성입니다.

Toleration은 파드들에 적용해 스케줄러로 하여금 매칭된 Taint가 있으면 스케줄링하게 합니다. Toleration은 스케줄링이 가능하게 하지만, 스케줄링이 된다고 보장하지는 않습니다. 스케줄링 시 검토하는 여러 변수 중 하나일 뿐입니다.

개념만 정리하면, Node들에 Taint를 "묻히고" 그 Taint가 묻어있는 Node들에는 그 Taint를 "버틸 수 있는" Toleration이 정의된 Pod들만 스케줄링이 가능합니다. 저는 이 지점이 가장 헷갈리는 것 같습니다. 이 전 다른 정의들은 Pod에 특정 Node를 지정하거나, 스케줄링이 될 수 있는 라벨을 지정하는 등 어떤 곳에 스케줄링이 "될 수 있는" 정의지만 Taint/Toleration은 어떤 곳에 스케줄링이 "될 수 없는" 정의라는 점이 헷갈리더라구요. 직접 명령어를 보면서 확인해보겠습니다.

## 기본 구성

어떤 노드에 Taint를 칠하려면, kubectl 명령어를 사용합니다. 

`kubectl taint nodes node1 key1=value1:NoSchedule`

위 명령어를 보면, `node1` 노드에 `key1` 인 키를 가지고 `value1` 인 값, 효과가 `NoSchedule` 인 Taint를 설정하고 있습니다. 이렇게 설정된 노드에는 동일한 내용의 Toleration을 가진 Pod 들이 스케줄링 가능하구요. 만약 Taint를 제거하고싶다면 간단하게 `-` 를 붙이면 됩니다.  

`kubectl taint nodes node1 key1=value1:NoSchedule-`

이런 Taint를 Pod이 견디게 하는 Toleration은 Pod sepec에 정의할 수 있습니다.

```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
```

위 매니페스트로 정의된 Tolerations는 앞에 정의된 Taint 와 정확하게 매치되어, 해당 노드에 스케줄링이 가능합니다. 혹은, 키만 존재한다면 스케줄링할 수 있게 정의할 수도 있습니다.

```yaml
tolerations:
- key: "key1"
  operator: "Exists"
  effect: "NoSchedule"
```

위와 같이 `key1` 이 존재하면 값이 어떻든 스케줄링될 수 있도록 Pod을 설정할 수도 있습니다.

이렇게, Taint를 Key:Value 값으로 설정 가능한데요. 이 Taint 조건에 따라 어떻게 작동할지도 설정 가능합니다. 다른 스케줄링 설정 방식들과 유사하게 3가지 방식으로 설정이 가능합니다.

- NoExecute
  - Tolerate 되지 않은 Pod들은 바로 노드에서 쫒겨납니다.
  - Tolerate 되었고, `tolerationSeconds` 설정이 되어있지 않는 Pod은 그대로 실행됩니다.
  - Tolerate 되었고, `tolerationSeconds` 설정이 되어있는 Pod은 설정되어 있는 시간만큼 실행된 뒤 노드에서 쫒겨납니다.
- NoSchedule
  - Toleration이 설정되지 않은 Pod들은 해당 노드에 스케줄링되지 않습니다. 이미 실행중인 노드들은 영향 없이 계속 실행됩니다.
- PreferNoSchedule
  - `NoSchedule` 의 약한 버전입니다. 스케줄러는 최대한 Toleration 되지 않은 Pod을 실행하지 않으려 하지만, 보장되지는 않습니다.

위 예시에서 Pod Spec에서 `effect` 값을 지정하였는데요, 지정하지 않을 경우 모든 효과를 허용하는 것으로 해석되어 작동합니다. 따라서,  Node 의 Taint가 `NoSchedule` 이던 `NoExecute` 이던 아래와 같이 설정된 Pod은 실행될 수 있습니다.

```
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
```

## 여러개의 Taint/Toleration이 있다면?

노드에는 동시에 여러 Taint가 설정될 수 있고, Pod에는 그걸 버티는 여러 Toleration이 설정될 수 있습니다. 이 경우, 여러개의 Taint는 마치 체를 거르는 것 같이 Pod들을 걸러냅니다.

노드에 걸려있는 Taint 들과 비교할 Pod의 toleration 을 봤을 때, `NoSchedule` Taint가 한개라도 있다면 Pod은 그 노드에 스케줄링될 수 없습니다. 만약 `NoSchedule` Taint들은 모두 버틸 수 있지만 `PreferNoSchedule` Taint가 남아있다면 최대한 그 노드에 스케줄링하지 않으려 합니다. 마지막으로, `NoExecute` Taint를 버틸 수 없다면 노드에서 쫒겨나거나(이미 실행중인 Pod일 경우), 스케줄링되지 않습니다.(실행중인 Pod이 아닌경우) 

예를 들어, 아래와 같이 3개의 Taint가 묻어있는 노드가 있고

```bash
kubectl taint nodes node1 key1=value1:NoSchedule
kubectl taint nodes node1 key1=value1:NoExecute
kubectl taint nodes node1 key2=value2:NoSchedule
```

아래와 같이 설정된 Pod 이 생성되는 경우,

```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
```

첫번째, 두번째 taint를 버티는 Toleration은 있지만 마지막 세번째 taint를 버티는 Toleration이 없기에 해당 Pod은 스케줄링되지 않습니다. 만약, 이 Pod이 이미 생성되어 실행되고 있고, 세 Taint가 추가된다면 쫒겨나지는 않습니다. `NoSchedule` 로 Taint가 정의되었기 때문이죠.

## 용례

 이런 Taint/Toleration은 어떻게 쓸까요? 공식 문서에선 크게 세가지 용례를 들고 있습니다.

### 지정 노드 설정

어떤 노드들을 특정 유저들만 사용할 수 있도록 설정하기 위해, 해당 노드들에 Taint를 설정하고 특정 유저들이 생성하는 Pod들에 Toleration 을 설정할 수 있습니다. 이렇게 구성하면, 다른 유저들의 워크로드가 해당 노드에서 작동하지 않고 특정 유저들만 설정된 노드를 사용할 수 있습니다. admission controller 를 사용하면, 특정 유저들이 생성하는 Pod에 자동적으로 Toleration을 붙일 수도 있습니다. 추가적으로, 언급한 유저들이 정해진 노드만 사용하도록 하기 위해서는 노드에 라벨을 붙이고 언급한 유저들의 워크로드 Pod에 nodeAffinity를 설정할 수 있습니다.

### 특정 하드웨어가 구성된 노드

특정 하드웨어가 구성된 노드가 클러스터에 포함되어 있는 경우, 그 하드웨어를 사용하는 Pod 들만 스케줄링되어야 효율적으로 자원을 사용 할 수 있습니다. 예를 들어, GPU가 설치되어 있는 노드에서 GPU를 사용하지 않는 Pod이 실행될 경우 다른 GPU-bound 작업들이 실행될 수 없어 비효율적인 상황이 발생합니다. 이를 막기 위해, GPU가 설치되어 있는 노드에 Taint를 묻히고(ex. `kubectl taint nodes nodename gpu=true:NoSchedule`) GPU를 사용해야 하는 Pod에 이 Taint와 매칭되는 Toleration을 구성하면 GPU가 필요한 Pod들만 GPU가 설치된 노드에 스케줄링해 효율적으로 자원을 사용할 수 있습니다.

### Taint based Evictions

노드 컨트롤러는 특정 조건이 되면 자동으로 노드를 Taint 합니다.

- `node.kubernetes.io/not-ready`:  노드가 준비되지 않을 경우 , `Ready` 값이 "`False`".
- `node.kubernetes.io/unreachable`: 노드가 노드 컨트롤러에서 접근할 수 없는 경우  `Ready` 값이 "`Unknown`".
- `node.kubernetes.io/memory-pressure`: 노드의 메모리가 부족해 memory pressure 상태인 경우
- `node.kubernetes.io/disk-pressure`: 노드의 디스크가 부족해 disk pressure 상태인 경우
- `node.kubernetes.io/pid-pressure`: 노드의 PID가 부족해 pid pressure 상태인 경우
- `node.kubernetes.io/network-unavailable`: 노드의 네트워크를 사용할 수 없는 경우
- `node.kubernetes.io/unschedulable`: 노드에 스케줄링할 수 없는 경우
- `node.cloudprovider.kubernetes.io/uninitialized`: kubelet이 외부 CSP 에서 작동될 경우, 노드가 아직 사용할 수 없는 상태임을 표시. CSP 제공 controller-manager가 노드 초기화된 뒤 이 Taint를 제거합니다.

만약, 노드에 문제가 있어 노드를 비워야 할 경우 보통 `node.kubernetes.io/not-ready`, `node.kubernetes.io/unreachable` 값으로 `NoExecute`Taint를 추가해 노드의 스케줄링을 멈추고 작동되고 있는 Pod들을 내쫒습니다. 

만약, Taint가 걸린 상황에서 어느정도 해당 노드에서 잠시 대기해야 할 경우 `tolerationSeconds` 를 설정해 응답 없거나 잠시 문제가 발생한 노드에 붙어있게 할 수 있습니다.

```yaml
tolerations:
- key: "node.kubernetes.io/unreachable"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 6000
```

Kubernetes는 자동적으로 `node.kubernetes.io/not-ready` 와 `node.kubernetes.io/unreachable` toleration이 없는 경우  `tolerationSeconds=300` 으로 설정해 붙입니다. 따라서, 만약 노드에 문제가 발생해 `not-ready` 나 `unreachable` 상태가 된 경우 최대 5분까지 해당 노드에 붙어있게 됩니다. 그 이전에 문제가 해결된다면 해당 노드에서 계속 자동하고, 만약 문제가 해결되지 않는다면 노드에서 쫒겨나 다른 노드에 스케줄링되게 됩니다.

## 조건에 따른 Node Taint

Kubernetes Control Plane은 node contrller를 사용해 node condition에 따라 `NoSchedule` 효과로 노드를 Taint 합니다.

스케줄러는 Pod를 스케줄할 때, 노드의 상태를 직접적으로 확인하지 않고 Taint를 통해 노드 상태에 따라 스케줄 될지 말지를 확인합니다. 이렇게 구성함으로써 노드의 상태가 스케줄링에 직접 영향을 주지 않게 구성할 수 있습니다. 예를 들어,  `DiskPressure` 상태를 가지는 노드에 control plane은 `node.kubernetes.io/disk-pressure` taint를 자동으로 설정합니다. 일반 Pod 들은 이 노드에 스케줄링될 수 없지만, QoS가 높은 Pod이나 `DiskPressure` 와 연관이 없는 Pod일 경우 tolereation를 설정해 노드에 스케줄링할 수 있습니다. 

DaemonSet 컨트롤러는 아래 Toleration을 추가해 무조건 노드에 Pod이 스케줄링 되어 DaemonSet이 깨지지 않도록 유지합니다.

- `node.kubernetes.io/memory-pressure`
- `node.kubernetes.io/disk-pressure`
- `node.kubernetes.io/pid-pressure` (1.14 혹은 그 이후)
- `node.kubernetes.io/unschedulable` (1.10 혹은 그 이후)
- `node.kubernetes.io/network-unavailable` (*host network 전용*)

## 마치며

여러 포스트를 거치며 Kubernetes 에서 워크로드의 가장 기본 구성요소인 Pod이 어떻게 스케줄링 되는지, 노드 관점에서 어떻게 설정할 수 있는지, Pod 관점에서는 어떻게 설정할 수 있는지를 알아 봤습니다. CKA때도 공부했었고, Kubernetes 워크로드를 구성해보면서도 접해봤던 개념들이지만 공식 문서와 테스트를 직접 해보면서 더 확실하게 개념을 잡은 것 같습니다. 

Kubernets 개념들을 보면, 참 추상화와 모듈화가 잘 되어 구성요소들이 레고블럭과 같이 조립되는 것 같습니다. 문서를 보면 볼 수록 느껴져 더더욱 인상깊게 남는 구조입니다.