---
layout: post
title: "공격 예고, 디버그용 박스에서 진짜 데칼로"
date: 2026-09-28 13:00:00 +0900
categories: [devlog, combat, vfx]
---

보스 근접 공격 버그를 고치고 나니, 그동안 가려져 있던 문제 하나가 눈에 들어왔다. 스윙 공격 때 보이던 빨간 사각형이 사실 실제 이펙트가 아니라 `CreateBoxHitbox(..., bDrawDebug=true)`가 그려주는 `BoxTraceMulti` 디버그 드로우였다는 거다. 에디터 PIE에서만 보이고, 그것도 `Notify_Hit` 판정이 실행되는 그 순간에만 잠깐 나타났다 사라진다. 실제 빌드에서는 아예 안 보인다.

이번엔 이걸 진짜 게임에서도 보이는 예고(텔레그래프) 데칼로 바꾸는 작업을 했다. 조건은 두 가지였다: SwingCombo뿐 아니라 `ERLAttackShape::Box`를 쓰는 다른 패턴(Subjugation 등)에도 그대로 재사용 가능할 것, 그리고 공격이 나가기 0.5~1초 전에 미리 뜨고 실제 타격 순간에 사라질 것.

## 노티파이 위치만으로 표시 구간을 계산하기

제일 처음 떠올린 방법은 "표시 시작" 노티파이를 몽타주에 하나 더 추가하는 거였다. 근데 그러면 몽타주마다 수동으로 노티파이를 얹어야 하고, 콤보처럼 히트가 여러 번인 패턴에서는 매번 짝을 맞춰줘야 해서 번거롭다. 그래서 방향을 바꿨다 — 이미 존재하는 `Notify_Hit`의 위치만 스캔해서, 표시 구간을 자동으로 역산하는 방식으로.

```cpp
void ARLCharacter::StartBoxAttackTelegraph(UAnimMontage* Montage, FName HitNotifyName, float LeadTime,
    FVector BoxExtent, float ForwardOffset, UMaterialInterface* DecalMaterial,
    AActor* FaceTarget, float TurnSpeed)
```

이 함수를 몽타주 재생 시작 직후 한 번만 호출하면, 내부에서 `Montage->Notifies`를 순회하며 이름이 일치하는 `Notify_Hit` 이벤트의 `GetTriggerTime()`을 전부 모으고, 각 히트마다 `[히트 - LeadTime, 히트)` 구간을 계산해서 저장해둔다. 콤보처럼 히트 간격이 `LeadTime`보다 짧으면 직전 히트 시각부터 바로 이어서 표시하도록 구간을 잘라서, 데칼이 겹쳐서 뭉치지 않게 했다.

실제 표시/숨김은 `Tick`에서 매 프레임 판단한다.

```cpp
const float Position = AnimInstance->Montage_GetPosition(TelegraphMontage);
```

몽타주의 "재생 위치"를 직접 기준으로 삼았기 때문에, 재생 속도가 달라지거나 히트스톱이 걸려도 타이밍이 어긋나지 않는다. 몽타주가 끝나거나 중간에 끊기면 `StopAttackTelegraph`가 자동으로 상태를 정리한다. Montage Notify든 스켈레톤 노티파이든 둘 다 인식하도록 `Event.Notify`가 있으면 `GetNotifyName()`을, 없으면 `Event.NotifyName`을 확인하는 식으로 짰다 — 지난 버그 수정 경험 때문에 이 부분은 특히 신경 써서 처리했다.

## 데칼 자체는 기존 인디케이터 액터를 재사용

`ARLDecalIndicator`에 원형/화살표용으로 이미 있던 구조를 그대로 확장해서 `Box` 타입을 추가했다.

```cpp
void ARLDecalIndicator::SetupBoxIndicator(const FVector& BoxExtent, UMaterialInterface* Material)
```

여기서 좀 헷갈렸던 부분이 데칼의 로컬 축 방향이다. `DecalComponent`는 생성자에서 `RelativeRotation = (Pitch -90, Yaw 0)`로 바닥을 내려다보는 상태인데, 여기에 소유자의 Yaw를 추가로 곱하면 로컬 Y축이 소유자의 RightVector, 로컬 Z축이 ForwardVector 방향으로 정렬된다(FRotator 합성 규칙을 손으로 유도한 결과라 처음엔 100% 확신은 못 했는데, PIE로 켜보니 맞았다). 그래서 `CreateBoxHitbox`의 `BoxExtent.Y`(좌우 폭)는 `DecalSize.Y`에, `BoxExtent.X`(전방 길이)는 `DecalSize.Z`에 매핑했다.

## 머티리얼: 원형에서 사각형으로, 그리고 스타일 맞추기

기존에 만들어둔 원형 데칼 머티리얼은 `RadialGradientExponential` 노드로 중심에서의 거리를 계산하는 구조였다. 사각형으로 바꾸려면 "원형 거리" 대신 "사각형 거리"가 필요한데, 이건 Chebyshev 거리(각 축 방향 거리 중 최댓값)로 구현했다.

```
TextureCoordinate → Subtract(0.5, 0.5) → Abs → (R, G 각각) → 둘 중 Max
```

처음엔 이걸로 부드러운 그라디언트 형태로 만들었는데, 실제로 PIE에서 보니 기존 원형 데칼의 느낌과 안 맞았다. 원형 데칼은 "선명한 테두리 + 안쪽 반투명 채움" 스타일인데, 사각형 버전은 중심이 진하고 가장자리로 옅어지는 피라미드 그라디언트였던 거다. 그래서 원형 데칼과 똑같은 구조로 다시 짰다 — 바깥쪽 임계값과 안쪽 임계값 두 개를 `If` 노드로 비교해서 그 사이 영역만 테두리로 만들고, 안쪽 전체는 낮은 고정 Opacity로 채우는 방식. Opacity는 (테두리 + 채움)이고 Emissive는 그 Opacity에 색을 곱한 값으로 처리했다.

## 결과

`BTT_Attack`에서 `Switch on AttackShape`의 Box 분기가 몽타주 재생 직후 `StartBoxAttackTelegraph`를 호출하도록 배선했다. 이제 스윙 공격은 실제로 휘두르기 0.5~1초 전에 데칼이 뜨고, 타격 순간 사라진다. 3연타 콤보는 세 번 뜨고 세 번 사라진다. PIE에서 방향/크기 확인도 끝났고, SwingCombo 패턴은 이걸로 완료됐다.

다음은 이 시스템을 그대로 확장해서 보스의 전방위(부채꼴) 공격 패턴까지 지원하도록 만드는 작업인데, 그건 다음 글에서.
