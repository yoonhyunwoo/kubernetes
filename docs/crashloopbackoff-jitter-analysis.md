# 쿠버네티스 파드 백오프에 지터 도입하기

쿠버네티스에서 파드가 실패할 때 재시도 간격을 조절하는 백오프(backoff)机制에 지터(jitter, 무작위 분산)를 추가하면 동시에 실패한 파드들이 한꺼번에 재시도하며 시스템을 압박하는 문제를 완화할 수 있다. 이 문서는 백오프가 동작하는 세 가지 레이어, 지터가 필요한 이유, 현재 논의 상태를 다룬다.

<br>

## 백오프가 동작하는 세 가지 레이어

쿠버네티스에서 "파드 실패 후 재시도"는 목적이 서로 다른 세 개의 독립적인 레이어에서 각각 동작한다. 같은 "백오프"라는 단어를 쓰지만, 계산 주체도, 지수 증가 방식도, 상한값도 다르다.

<br>

**1. Kubelet - 컨테이너 크래시 재시작 (CrashLoopBackOff)**

컨테이너가 비정상 종료하면 kubelet이 지수 백오프 간격으로 재시작한다. 초기값 10초에서 시작해 매 실패마다 2배씩 증가하며, 최대 5분(300초)에서 멈춘다. 예를 들어 10초, 20초, 40초, 80초, 160초, 300초 순서다.

이미지 풀 실패 시에도 같은 구조가 동작한다. ImagePullBackOff 상태가 되면, 초기값 10초에서 최대 5분까지 지수 백오프로 재시도한다.

실제 백오프 계산은 [flowcontrol.Backoff](https://github.com/kubernetes/kubernetes/blob/8c078bc32e90f0ea72f57169846b533e20b1ca71/staging/src/k8s.io/client-go/util/flowcontrol/backoff.go#L96-L109) 구조체가 담당한다. Next 메서드가 매 실패마다 backoff 값을 2배로 늘리고, maxDuration(5분)에서 캡한다. 핵심 코드는 다음과 같다.

```
delay := entry.backoff * 2
entry.backoff = min(delay, p.maxDuration)
```

[kubelet.go](https://github.com/kubernetes/kubernetes/blob/8c078bc32e90f0ea72f57169846b533e20b1ca71/pkg/kubelet/kubelet.go#L165-L166)에서 초기값과 상한을 정의한다. newCrashLoopBackOff 함수([kubelet.go:353-369](https://github.com/kubernetes/kubernetes/blob/8c078bc32e90f0ea72f57169846b533e20b1ca71/pkg/kubelet/kubelet.go#L353-L369))가 feature gate에 따라 이 값을 교체한다.

<br>

**2. 스케줄러 - 스케줄링 실패 재시도 (backoffQ)**

파드가 노드 부족이나 affinity 불일치로 스케줄링에 실패하면, 스케줄러는 파드를 activeQ에서 backoffQ로 옮긴다. 초기값 1초, 최대 10초로 지수 백오프가 동작한다. 예를 들어 1초, 2초, 4초, 8초, 10초 순서다.

[calculateBackoffDuration](https://github.com/kubernetes/kubernetes/blob/8c078bc32e90f0ea72f57169846b533e20b1ca71/pkg/scheduler/backend/queue/backoff_queue.go#L249-L266)이 비트 시프트로 백오프를 계산한다. 핵심은 initialBackoff를 count-1만큼 왼쪽으로 시프트하는 것이다.

<br>

**3. Job 컨트롤러 - Job 파드 실패 재시도 (backoffLimit)**

Job의 파드가 실패하면 교체 파드를 생성하기 전에 대기한다. 초기값 10초, 최대 60초로 지수 백오프가 동작한다. backoffLimit(기본 6회)을 초과하면 Job 전체를 Failed로 표시한다.

[getRemainingTimeForFailuresCount](https://github.com/kubernetes/kubernetes/blob/8c078bc32e90f0ea72f57169846b533e20b1ca71/pkg/controller/job/backoff_utils.go#L255-L276)가 매 실패마다 backoffDuration을 2배로 늘린다.

<br>

## 지터가 필요한 이유

백오프에 지터가 없으면, 동시에 실패한 파드들이 동일한 백오프 스케줄을 가진다. 파드 100개가 같은 시각에 크래시하면, 100개 모두 10초 후에 동시에 재시도하고, 실패하면 20초 후에 다시 100개가 동시에 재시도한다. 이 "동기화된 재시도(synchronized retry)"가 다음과 같은 연쇄 문제를 일으킨다.

<br>

**API 서버 부하 스파이크**

재시도 시점에 파드 100개가 동시에 API 서버에 요청을 보낸다. API 서버가 이를 감당하지 못하면 2차 실패가 발생하고, 백오프가 더 길어지는 악순환이 시작된다.

<br>

**이미지 레지스트리 폭주**

ConfigMap이나 Secret 업데이트로 파드가 전체 재시작할 때, 모든 파드가 같은 시각에 이미지 풀을 재시도한다. 레지스트리가 재폭주하고 다시 실패한다.

<br>

**복구 지연**

백오프 윈도우가 5분일 때, 모든 파드가 5분마다 한 번에 재시도한다. 백오프 윈도우 안에서 원인이 해결돼도, 다음 윈도우까지 최대 5분을 기다려야 한다. 지터가 있으면 일부 파드는 더 일찍 재시도하므로 복구가 점진적으로 진행된다.

<br>

## 현재 코드에서 지터 상태

**Kubelet (CrashLoopBackOff, ImagePullBackOff)** - 지터 인프라는 이미 구축돼 있지만 활성화되지 않았다.

[Backoff 구조체](https://github.com/kubernetes/kubernetes/blob/8c078bc32e90f0ea72f57169846b533e20b1ca71/staging/src/k8s.io/client-go/util/flowcontrol/backoff.go#L45-L48)에 maxJitterFactor 필드가 있고, [jitter 메서드](https://github.com/kubernetes/kubernetes/blob/8c078bc32e90f0ea72f57169846b533e20b1ca71/staging/src/k8s.io/client-go/util/flowcontrol/backoff.go#L173-L179)가 구현돼 있다. Next 메서드에서 백오프 증가 시 이 값을 더한다.

하지만 kubelet이 Backoff를 생성할 때 [NewBackOff](https://github.com/kubernetes/kubernetes/blob/8c078bc32e90f0ea72f57169846b533e20b1ca71/staging/src/k8s.io/client-go/util/flowcontrol/backoff.go#L55-L57)(initial, max)를 호출한다. 이 생성자는 내부적으로 maxJitterFactor를 0.0으로 설정한다. 지터 인프라가 준비돼 있지만 스위치가 꺼져 있는 상태다.

지터를 켜려면 [NewBackOffWithJitter](https://github.com/kubernetes/kubernetes/blob/8c078bc32e90f0ea72f57169846b533e20b1ca71/staging/src/k8s.io/client-go/util/flowcontrol/backoff.go#L63-L66)(initial, max, jitterFactor)를 호출하면 된다. 예를 들어 jitterFactor가 0.5면, 10초 백오프에 0~5초의 무작위 분산이 추가된다.

<br>

**스케줄러 backoffQ** - 지터가 없다. calculateBackoffDuration이 순수 비트 시프트만 사용한다.

**Job backoff** - 지터가 없다. getRemainingTimeForFailuresCount가 순수 곱셈만 사용한다.

<br>

## 업스트림 논의 상태

이 문제는 2020년부터 인지됐지만 아직 해결되지 않았다. 세 개의 핵심 이슈와 PR이 타임라인을 이룬다.

<br>

**이슈 #87915 - "Backoffs should have jitter" (2020년 2월, 자동 종료)**

[이슈 #87915](https://github.com/kubernetes/kubernetes/issues/87915)에서 vllry가 최초로 문제를 제기했다. "CrashLoopBackOff 파드들이 lock-step으로 재시도하며 API를 때린다." lavalamp는 API Priority and Fairness(KEP-1040)로 우회 해결을 시도했지만, 근본 해결이 아니었다. 이슈는 활동 부족으로 rotten 처리되어 자동 종료됐다.

이 논의의 결과로 2021년 11월, [커밋 ec93e854](https://github.com/kubernetes/kubernetes/commit/ec93e854ca0)에서 flowcontrol.Backoff에 지터 인프라가 추가됐다. 하지만 kubelet에서 이를 활성화하는 작업은 이뤄지지 않았다.

<br>

**이슈 #123201 - "Add jitter to CrashLoopBackOff" (2024년 2월, 열림)**

[이슈 #123201](https://github.com/kubernetes/kubernetes/issues/123201)이 현재 활성 이슈다. kind/feature, sig/node, help wanted 라벨이 붙어 있다. "대규모 synchronized restart loop가 공통 자원에 압력을 가한다"는 것이 핵심 동기다.

thockin(Tim Hockin, 쿠버네티스 창립 멤버)이 "이거 생각만큼 간단한가?"라고 물었고, tallclair(Tim Allclair)가 "그래, 간단할 거야"라고 답했다.

2025년 9월 16일, toVersus가 구체적인 diff를 제시했다. 다음 한 줄만 바꾸면 된다.

```
- klet.crashLoopBackOff = flowcontrol.NewBackOff(base, boMax)
+ klet.crashLoopBackOff = flowcontrol.NewBackOffWithJitter(base, boMax, boMaxJitterFactor)
```

<br>

**PR #133066 - "Propagate backoff duration for crashloop backoff" (2025년 10월, 병합)**

[PR #133066](https://github.com/kubernetes/kubernetes/pull/133066)은 지터와는 별개지만 밀접하게 관련된 버그 수정이다. CrashLoopBackOff에서 계산된 백오프 시간이 pod worker에 제대로 전파되지 않던 문제를 해결했다. BackoffError 타입을 도입해 컨테이너 시작 로직이 정확한 백오프 시간을 반환하고, pod sync 루프가 그 값을 사용하도록 수정했다.

이것이 [kuberuntime_manager.go:2104](https://github.com/kubernetes/kubernetes/blob/8c078bc32e90f0ea72f57169846b533e20b1ca71/pkg/kubelet/kuberuntime/kuberuntime_manager.go#L2104)의 NewBackoffError 호출과 sync_result.go의 MinBackoffExpiration 구현체다.

<br>

**KEP-4603 - "Tune CrashLoopBackOff" (v1.32 알파)**

[KEP-4603](https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/4603-tune-crashloopbackoff/README.md)은 백오프 곡선 자체를 완화하는 작업이다. ReduceDefaultCrashLoopBackOffDecay feature gate를 통해 기본 백오프를 10초에서 5분에서 1초에서 60초로 줄인다. KubeletCrashLoopBackOffMax(별도 KEP-5593)는 노드별 maxContainerRestartPeriod를 설정할 수 있게 한다.

지터 도입과 이 KEP 사이에는 긴장이 있다. toVersus가 지적한 대로, 지터를 넣으면 재시작 시간이 현재보다 길어질 수 있는데, 이는 KEP-4603의 방향(빠른 복구)과 반대다.

<br>

## 남은 논의 거리

toVersus가 이슈 #123201에 올린 미해결 질문 세 가지가 현재 논의의 핵심이다.

<br>

**지터 계수 기본값**

jitterFactor가 0.1일 때, 1초 시작 백오프에서 최대 지터는 0.1초다. 이 분산량이 동기화 해소에 충분한지 의문이다. 반면 너무 크면(0.5 이상) 백오프 예측성이 떨어져 디버깅이 어려워진다.

thockin은 대칭 지터를 제안했다. 지터를 양방향(plus/minus)으로 분산시켜 평균 백오프를 기존과 동일하게 유지하는 방식이다. 현재 flowcontrol.Backoff.jitter 메서드는 단방향(plus만) 지터만 지원하므로, 이 제안을 구현하려면 메서드 자체를 수정해야 한다.

<br>

**설정 노출 여부**

kubelet 설정 파일에서 maxContainerRestartJitterFactor를 노출해야 하는지에 대한 논의다. thockin은 "설정 추가보다 적은 게 낫다"며 고정값을 선호했다.

<br>

**KEP 필요성**

지터 도입은 행동 변화이므로 KEP가 필요한지에 대한 질문이다. tallclair와 thockin 모두 KEP가 적절하다고 동의했다.

toVersus의 결론은 다음과 같다. "ReduceDefaultCrashLoopBackOffDecay가 아직 알파에 머물러 있으니, 최종 decay 곡선이 확정된 후에 지터를 다루는 게 순서가 맞겠다."

<br>

## 기여를 고려하는 사람에게

이슈 #123201은 help wanted 상태다. 구체적 diff까지 제시된 상태에서 ReduceDefaultCrashLoopBackOffDecay의 최종 값 확정만 기다리고 있다.

필요한 작업은 다음과 같다. 첫째, KEP-4603의 최종 decay 곡선 확정을 확인한다. 둘째, 지터 계수 기본값을 정한다. 셋째, 단방향 지터를 그대로 쓸지 대칭 지터로 확장할지 결정한다. 넷째, KEP를 작성한다. 다섯째, flowcontrol.Backoff 생성부를 NewBackOff에서 NewBackOffWithJitter로 변경하고 테스트를 수정한다.

기존 테스트 중 TestDoBackOff(kuberuntime_manager_test.go:6153)와 TestMinBackoffExpiration이 결정론적 백오프를 가정하므로, 지터 도입 시 가짜 시계(fake clock)와 고정 시드(fixed seed)로 테스트를 수정해야 한다.
