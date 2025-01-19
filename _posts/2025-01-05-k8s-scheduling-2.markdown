---
title:  " Kubernetes 스케줄링 정리 2 - Pod Topology Spread Constraints 필드 분석"
date:   2025-01-05 11:55:50+0900
categories: [Cloud]
tags: [tech, kubernetes]
---
Karpenter를 구성하면서, Pod 스케줄링에 대한 이해가 부족하다는 것을 느껴서 정리중인 Pod 스케줄링 포스팅 중 두번째인 Pod Topology Constraints입니다. 이게 실제로 사용되는건 직접 몇 번 보질 못해 공식 문서를 토대로 설정 방법만 정리해봤습니다.

## Pod Topology Constraints?

공식 문서에서는 Pod Topology Constraints가 파드 토폴로지 분배 제약 조건 으로 번역이 되어있습니다. 장애 발생 시 문제가 되는 덩어리(failure-domain)들을 고려해서 Pod들을 분배하는 제약조건을 설정한다고 이해되는데, 어느정도 맞는것 같습니다. Failure-domain은 흔히 클라우드 환경에서 볼 수 있는 리전이나 Zone(AZ), 노드나 사용자가 별도의 토폴로지를 설정할 수도 있습니다.

워크로드를 배포하고 부하에 따라 Pod 수가 감소하거나 증가하는 예를 들어 보면, 부하가 줄어들어 Pod 수가 두세개 밖에 남지 않았을 때 이 Pod들이 한 노드에 떠있다면 아주 불안한 상황에 처할 수 있습니다. 반대로, 부하가 늘어 Pod들이 많이 늘었을 때는 Zone 간 통신이 급격하게 느는것을 방지하기 위해 한 Zone에 관계있는 Pod들을 최대한 몰아서 띄워야 할 수 있습니다. 이런 경우에 Pod Topology Constraints를 사용할 수 있습니다.

## 필드 분석

 topologySpreadConstraints 필드를 보며 분석해봅니다.

```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
spec:
  # topology spread constraint 설정
  topologySpreadConstraints:
    - maxSkew: <integer>
      minDomains: <integer> # optional
      topologyKey: <string>
      whenUnsatisfiable: <string>
      labelSelector: <object>
      matchLabelKeys: <list> # optional; beta since v1.27
      nodeAffinityPolicy: [Honor|Ignore] # optional; beta since v1.26
      nodeTaintsPolicy: [Honor|Ignore] # optional; beta since v1.26
```

한개 이상의 topologySpreadConstraints 필드를 넣어서 kube-scheduler가 Pod을 스케줄링할 때 고려하게 할 수 있습니다.

- maxSkew: Pod들이 얼마나 불균형하게 분배될지 설정 가능합니다. 0보다 큰 정수로 설정 가능하고, `whenUnsatisfiable` 값에 따라 숫자의 의미가 바뀝니다.
  - `whenUnsatisfiable: DoNotSchedule` 일 경우, 특정 토폴로지에서 일치하는 Pod 수와 전역 최소값 간 최대 허용 차이를 정의합니다.
    - 전역 최소값 : 사용 가능한 도메인에서 일치하는 Pod의 최소 수 or 사용 가능한 도메인 수가 MinDomains 보다 적을 경우 0
  - `whenUnsatisfiable: ScheduleAnyway` 일 경우, 스케줄러는 불균형을 줄이는 방향으로 스케줄을 진행합니다.
- minDomains: 사용 가능한 도메인의 최소 수를 설정합니다. 사용 가능한 도메인은 node selector로 매치되는 노드들의 묶음입니다.  (Optional)
  - 설정할 경우 0보다 큰 숫자를 설정해야 합니다. `whenUnsatisfiable: DoNotSchedule` 이 설정되어 있을때만 설정 가능합니다.
  - 사용 가능한 도메인의 수가 minDomains 보다 낮을 경우, Global minimum을 0으로 설정하여 skew 계산을 진행합니다.
  - 사용 가능한 도메인의 수가 minDomains 보다 높거나 같을 경우 minDomains는 영향을 주지 않습니다.
  - minDomains 를 설정하지 않을 경우, 기본 값으로 1을 설정한 것으로 동작합니다.
- topologyKey: 노드 라벨의 키를 지정. 이 키의 라벨 값이 같은 노드들을 같은 토폴로지 안에 있는 것으로 보고 스케줄링을 진행합니다. 각 토폴로지의 인스턴스(key-value 쌍)를 "도메인"이라고 부릅니다. 스케줄러는 각 도메인에 골고루 Pod들을 스케줄링합니다. 사용 가능한 도메인을 찾을때는 Pod의 nodeAffinityPolicy와 nodeTaintsPolicy를 고려해서 찾습니다.
- labelSelector: topologySpreadConstraint에서 제약 조건을 설정할 Pod의 라벨을 설정합니다. 이 selector와 일치하는 Pod의 수를 계산해 해당 도메인에 속할 Pod의 수를 결정합니다.
- matchLabelKeys: Pod의 특정 라벨 키 값을 사용해 스프레드 계산을 수행할 대상을 결정합니다. 조회된 키-값 라벨은 위에 있는 labelSelector와 AND 계산을 해 추가적으로 들어오는 Pod의 스프레드 계산에 사용할 기존 Pod 그룹을 선택합니다.
  - matchLableKeys에 pod-template-hash를 넣으면, deployment 버전이 바뀌며 새로 추가되는 Pod들을 고려해 스프레드 할 수 있습니다.
  - 이건 다음 포스팅에 테스트를 진행해 볼 예정입니다.
- nodeAffinityPolicy: Pod의 nodeAffinity/nodeSelector 값을 어떻게 처리할지 설정합니다.
  - Honor: nodeAffinity/nodeSelector 값에 맞는 노드들만 계산에 포함합니다.
  - Ignore: nodeAffinity/nodeSelector 값을 무시하고 계산합니다.
  - 필드 값이 비어있을 경우 Honor로 설정한것으로 동작합니다.
- nodeTaintsPolicy: node taint를 어떻게 처리할지 설정합니다.
  - Honor: taint가 없는 노드는 포함, taint가 있는 노드는 toleration 이 묻어있는 pod일 경우 스케줄에 포함됩니다.
  - Ignore: taint들은 무시합니다. 모든 노드가 스케줄에 포함됩니다.

Pod가 한개 이상의  `topologySpreadConstraint` 를 설정할 경우, 모든 제약사항들은 AND 연산이 되어 계산됩니다. kube-scheduler는 설정된 모든 제약사항에 맞는 노드를 찾게 됩니다.

## 노드 라벨

Topology Spread Constraints 는 토폴로지 도메인들을 식별하기 위해 노드 라벨에 의존합니다. 잘 알려진 라벨들은 `topology.kubernetes.io/zone` , `topology.kubernetes.io/region` 등이 있고, CSP의 Managed Kubernetes 서비스를 사용할 경우 CSP가 제공하는 토폴로지도 사용이 가능합니다.

## 일관성

Pod Topology Constraint를 설정할 때, 그룹 내 모든 Pod에 설정해야 합니다. 보통 Deployment 같은 워크로드 컨트롤러를 사용할 경우, Pod Template가 알아서 설정해줘 신경쓸일이 없지만, 별도로 Kubernetes API를 건드려 그룹 내 몇몇 Pod의 Pod Topology Constraint가 달라지거나 할 경우 동작이 정상적으로 되지 않거나 트러블슈팅시 복잡도가 늘어날 가능성이 있습니다.

더불어, 토폴로지 도메인 내 모든 노드들이 일관적으로 라벨링 되어야 합니다. CSP가 제공하는 Managed Kubernetes 서비스를  사용할 경우, 노드 관리 체계가 제대로 구성되어 노드 내 라벨이 잘 붙는지 확인해둘 필요가 있습니다. 대부분의 클러스터는 자동으로 잘 알려진 라벨들(`kubernetes.io/hostname` 등)을 붙여줍니다. 

---

설정하는 문서만 봐서는 약간 복잡하지만 제대로 구성만 한다면 클러스터 안에서 Pod들을 아름답게 분배해줄 것 같은데요. 각 필드 내 버전을 보니 추가된지 얼마 되지 않은 기능인 것 같습니다. 실제로 몇몇 필드들은 베타 상태가 많구요. 다음 포스팅에서는 실제로 Pod을 띄워보면서 테스트를 해 보려 합니다!