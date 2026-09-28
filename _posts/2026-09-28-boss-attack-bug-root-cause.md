---
layout: post
title: "보스 공격 버그, 진짜 원인은 전혀 다른 곳에 있었다"
date: 2026-09-28 12:00:00 +0900
categories: [devlog, bugfix]
---

지난 글(보스 버그는 아직 진행형, 그리고 근접 판정 설계를 하나 엎었다)에서 `URLStatComponent::BeginPlay()`에 스탯 값을 `MaxWalkSpeed`에 대입하는 코드가 아예 없다는 것까지는 확인했었다. 다만 `BP_Boss`의 실제 `Max Walk Speed` 값이 얼마인지, `Sevarog_AnimBlueprint`의 `Divide_DoubleDouble` 노드가 정확히 뭘 나누고 있는지는 그때까지 미확인 상태로 남겨뒀다.

결론부터 말하면, 그 갈래는 더 이상 파고들지 않았다. 이번에 다른 각도에서 BT 구조를 처음부터 다시 훑다가, 보스가 공격 중 멈추는 증상의 실제 원인이 이동속도 쪽과는 전혀 무관한 곳에 있다는 걸 알게 됐기 때문이다. 오늘은 그 진짜 원인 두 가지를 어떻게 찾았는지 정리한다.

## 진짜 문제: 몽타주가 스스로 끊기고 있었다

로그를 다시 자세히 들여다보니, **보스의 근접 공격 몽타주(`AM_Boss_Swing`)가 재생 시작 후 약 1초 만에 `Montage.Stop`으로 강제 종료된다**는 걸 발견했다. `Notify_Hit`가 걸린 스윙 후반부 타이밍에 도달하기도 전에 끊기고 있었다.

문제는 여기서 한 번 더 꼬였다. Behavior Tree의 `OnCompleted`가 노티파이 발동 여부와 무관하게 바로 `FinishExecute(true)`로 연결돼 있어서, BT 입장에서는 이걸 "정상 완료"로 착각하고 다음 노드(Wait)로 그냥 넘어갔다. 그래서 겉보기엔 흐름이 정상처럼 보였던 거다. PIE 로그에는 `SelectPattern`이 패턴 번호를 `3 2 1 0` 식으로 계속 재시도하는 것만 찍혔고, 처음엔 이게 그냥 가중치 기반 랜덤 선택 로직의 정상 동작인 줄 알았다.

## 원인 1: NotifyObserver가 ValueChange였다

`BT_Boss`의 구조를 다시 뜯어보니, 일반 공격 시퀀스(SelectPattern → MoveTo → BTT_Attack → Wait) 앞에 붙어있는 `BTDecorator_Blackboard`의 `NotifyObserver`가 `ValueChange`로 설정되어 있었다.

`BTS_CheckAttackRange` 서비스가 매 틱마다 `IsInAttackRange = (거리 <= TargetAttackRange)`를 계산하는데, `TargetAttackRange`는 `BTT_SelectPattern`이 고른 패턴의 사거리 값으로 매번 덮어써진다. 문제는 패턴마다 사거리가 다 다르다는 거였다 — SwingCombo 200, GroundSlam 350, SoulPull 500, Subjugation 300.

즉 패턴이 바뀔 때마다 `TargetAttackRange` 기준이 달라지고, 그러면 `IsInAttackRange` 값이 false→true로 다시 뒤집히는 순간이 생긴다. 그런데 `NotifyObserver = ValueChange`는 조건이 이미 참이어도 **값이 바뀌기만 하면 실행 중인 자기 브랜치를 처음부터 재시작**시킨다. Abort 모드가 `Lower Priority`라서 다른 형제 브랜치가 끊는 줄 알았는데, 실제로는 같은 브랜치가 자기 자신을 재시작하고 있었던 거다. 이게 "몽타주가 재생되다가 약 1초 만에 Stop되고 SelectPattern이 반복 재시도되는" 증상과 정확히 맞아떨어졌다.

수정은 간단했다. `NotifyObserver`를 `ValueChange`에서 `ResultChange`로 바꿨다 — 조건의 참/거짓 "결과"가 실제로 바뀔 때만 재평가하도록. 이렇게 고치고 PIE를 돌려보니 스윙 몽타주가 드디어 끝까지 재생됐다.

## 원인 2: Notify_Hit이 스켈레톤 노티파이였다

첫 번째 원인을 고쳤는데도 데미지가 안 들어갔다. 로그를 보니 `HIT_BRANCH_REACHED`가 한 번도 안 찍히고 있었다. `.uasset` 문자열을 검사해보니, 보스 몽타주 4개의 `Notify_Hit`이 전부 **스켈레톤 노티파이**였다. 반면 정상 동작하는 일반 몬스터 몽타주(`AM_Enemy_Attack` 등)는 `AnimNotify_PlayMontageNotify`(Montage Notify) 타입을 쓰고 있었다.

`PlayMontageAndWait`의 `OnNotifyBegin` 출력 핀은 **Montage Notify만 받는다.** 스켈레톤 노티파이는 AnimBP 이벤트 그래프로만 전달되기 때문에, `BTT_Attack`의 `OnNotifyBegin` 분기 자체에 도달할 수가 없었던 거다. 이건 사실 이전에도 이 프로젝트에서 한 번 겪었던 종류의 함정이라, 알고 나니 바로 납득이 됐다.

`AM_Boss_Swing`의 `Notify_Hit`을 Montage Notify(Notify Name = `Notify_Hit`)로 다시 만들어서 교체했다. `Notifies` 배열 자체는 MCP로는 편집이 안 돼서 에디터에서 직접 고쳤다. 이후 PIE 로그에 `HIT_BRANCH_REACHED`가 공격마다(12연타 기준 12회) 정확히 찍히는 걸 확인했다.

## 정리

같은 증상처럼 보였던 문제가 사실은 서로 다른 두 개의 원인이 겹쳐 있었다.

1. BT 데코레이터의 `NotifyObserver = ValueChange` → 사거리가 다른 패턴으로 전환될 때마다 공격 시퀀스가 자기 자신을 재시작
2. 2. 몽타주의 `Notify_Hit`이 스켈레톤 노티파이 → `PlayMontageAndWait.OnNotifyBegin`이 아예 못 받음
  
   3. 둘 다 고치고 나서야 SwingCombo 패턴이 의도대로 완전히 동작했다. 이동속도/나눗셈 쪽 가설은 결국 끝까지 확인하지 않은 채로 남겨뒀다 — 어차피 "몽타주가 1초 만에 스스로 끊긴다"는 훨씬 직접적인 원인을 로그에서 먼저 찾아냈고, 그게 실제 증상을 전부 설명했기 때문이다. 두 가설이 서로 배타적인 건 아니라서 나중에 시간 나면 그쪽도 마저 확인은 해볼 생각이다.
  
   4. 아직 남은 일도 있다. `AM_Boss_BigSwing`/`AM_Boss_SoulDrain`/`AM_Boss_Subjugation` 세 몽타주의 `Notify_Hit`도 아직 스켈레톤 노티파이라서, 같은 방식으로 교체가 필요하다. `BP_Boss`에 있는 것으로 확인된 `AttackCollision_L`/`AttackCollision_R` 기반의 별도 콜리전 데미지 시스템이 실제로 지금도 쓰이는 경로인지도 아직 정리하지 못했다.
   5. 
