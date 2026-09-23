---
layout: post
title: "보스 버그는 아직 진행형, 그리고 근접 판정 설계를 하나 엎었다"
date: 2026-09-23 12:00:00 +0900
categories: [devlog, bugfix, design]
---

지난 글(보스가 안 움직이는 버그, 원인은 나눗셈이었다)에서 세운 가설을 이어서 확인했다. 결론부터 말하면 아직 못 고쳤다. 대신 그 과정에서 근접 공격 판정 쪽 설계를 하나 뒤집었는데, 그 이야기도 같이 남긴다.

## 보스 버그 후속: MaxWalkSpeed 가설, 절반만 맞았다

지난 글 체크리스트 중 "이동속도를 스탯 컴포넌트(URLStatComponent)로 관리하는 구조라면, BeginPlay에서 스탯 값을 MaxWalkSpeed에 대입하는 로직이 실제로 있는지" 항목을 코드로 다시 확인했다.

```cpp
void URLStatComponent::BeginPlay()
{
    Super::BeginPlay();
    // ...
}
```

URLStatComponent::MoveSpeed(기본값 600)를 선언만 해두고, 실제로 CharacterMovement->MaxWalkSpeed에 대입하는 코드는 어디에도 없다는 걸 확인했다. 즉 "스탯 값이 이동속도에 반영되는 연결 자체가 없다"는 가설의 앞부분은 사실이었다.

다만 이것만으로 보스가 안 움직이는 증상이 100% 설명되는 건 아니라서, 아직 실제로 고치지는 못했다. BP_Boss의 Character Movement에서 Max Walk Speed 값을 직접 얼마로 세팅해뒀는지, Sevarog_AnimBlueprint의 Divide_DoubleDouble 노드가 정확히 뭘 나누고 있는지는 여전히 미확인 상태. 다음에 마저 확인할 것.

## 근접 공격 판정, NotifyState로 만들다가 엎은 이유

보스 패턴(SwingCombo / GroundSlam / SoulPull / Subjugation)에 타격 타이밍을 정확히 주려고, 몽타주 Notify 구간(NotifyBegin~NotifyEnd) 동안 무기 소켓 사이를 스윕 트레이스로 판정하는 URLAnimNotifyState_MeleeDamage 클래스를 새로 만들었다.

그런데 다시 코드를 보다가, 이미 있는 구조와 정면으로 겹친다는 걸 깨달았다. RLTypes.h의 FRLMonsterPatternData에는 ERLAttackShape(Box / Location / Projectile)가 있고, BTT_Attack이 이 값을 보고 ARLCharacter::CreateBoxHitbox / CreateLocationHitbox / SpawnProjectileAttack을 직접 호출하는 구조가 이미 있었다. 심지어 주석에 "Box: 정면 박스 판정 (근접, 예: SwingCombo, Subjugation)"이라고 못까지 박혀 있었다. 즉 보스 근접 판정 타이밍은 비헤이비어 트리 쪽에서 이미 잡고 있었던 것.

같은 문제를 두 가지 다른 방식(BT 타이밍 vs 애니메이션 Notify)으로 중복 구현하는 셈이라, URLAnimNotifyState_MeleeDamage는 커밋하지 않고 폐기하기로 했다. 몬스터 패턴은 계속 BTT_Attack + CreateBoxHitbox/CreateLocationHitbox 조합으로 가는 걸로 정리.

대신 몽타주 Notify는 BT로 타이밍을 잡기 어려운 쪽(플레이어 캐릭터 콤보처럼 입력 기반으로 진행되는 애니메이션)에 남겨두기로 했다. 그 와중에 몽타주에 Notify_Hit라는 이름 있는 Notify를 추가하려다가, 이름을 안 넣고 만들면 트랙에 "None"으로 남아서 나중에 헷갈리는 걸 겪었다. Add Notify 메뉴 검색창에 이름을 바로 타이핑해서 "New Notify: Notify_Hit"로 만드는 게 제일 깔끔하다는 것도 이번에 정리.

## 다음에 할 것

- BP_Boss Character Movement의 Max Walk Speed 실제 값 확인
- Sevarog_AnimBlueprint의 Divide_DoubleDouble 노드 직접 추적
- 플레이어 캐릭터 콤보용 몽타주에 Notify_Hit 배치 및 판정 로직 연결
