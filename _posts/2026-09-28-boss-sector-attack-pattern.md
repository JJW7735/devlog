---
layout: post
title: "보스 공격 패턴 확장: 전방위 부채꼴 판정 (진행중)"
date: 2026-09-28 14:00:00 +0900
categories: [devlog, combat, wip]
---

> 이 글은 진행중인 작업 기록이다. 아래 C++ 구현은 끝났지만 아직 라이브로 한 번도 켜보지 못했고, 블루프린트 배선과 전용 머티리얼도 남아있다. 완료되면 후속 글에서 실제로 의도대로 동작했는지 정리할 예정이다.
>
> 직전 글에서 만든 근접 박스 공격 예고 시스템을 SwingCombo에 붙이고 나니, 다음 패턴인 BigSwing도 손봐야 했다. `DA_Enemy_Boss`를 뜯어보니 이때까지 내가 "GroundSlam"이라고 불렀던 패턴의 실제 몽타주가 `AM_Boss_BigSwing`이었다 — 이름이 예상과 달라서 한 번 더 확인이 필요했다.
>
> ## 요구사항
>
> BigSwing 몽타주는 사전동작(손을 뻗고 잠깐 정지)한 뒤에 크게 휘두르는 구조다. 원하는 동작은 이렇다.
>
> - 사전동작이 시작되는 순간 큰 데칼이 뜬다
> - - 판정은 보스를 중심으로 한 부채꼴 형태: 정면 기준 바로 뒤쪽 90도만 안전지대, 나머지 270도는 전부 위험지대
>   - - 실제 스윙이 나가는 순간, 그 270도 범위 안에 있는 대상 전부에게 한 번에 데미지
>    
>     - 기존에 있던 "특정 지점 중심 원형 판정"(`ERLAttackShape::Location`, `CreateLocationHitbox`)과는 다르다. 이건 지정된 한 지점 주변을 때리는 거고, 이번에 필요한 건 "보스 자기 자신"을 중심으로 한 도넛 모양(정확히는 뒤쪽 쐐기를 뺀 원)이다.
>    
>     - ## 설계: 새 시스템을 만들지 않고 기존 걸 확장
>    
>     - Box 텔레그래프 시스템(`StartBoxAttackTelegraph`)을 만들 때 이미 "노티파이 위치 스캔 → 표시 구간 계산 → Tick에서 판단"이라는 틀이 잘 잡혀 있었기 때문에, 부채꼴 전용으로 별도 시스템을 새로 만들지 않고 이 틀을 그대로 확장하기로 했다. 구체적으로는:
>
> - 노티파이 스캔 로직을 `ComputeTelegraphWindows`라는 공용 헬퍼로 분리해서 Box/Sector 둘 다 재사용
> - - `ETelegraphShape { Box, Sector }` 내부 enum으로 현재 텔레그래프가 어느 모양인지 구분
>   - - `UpdateAttackTelegraph`가 이 값으로 분기해서 `ShowBoxAttackIndicator` 또는 `ShowSectorAttackIndicator` 중 알맞은 쪽을 호출
>    
>     - ## 데이터: Sector 판정 추가
>    
>     - ```cpp
>       UENUM(BlueprintType)
>       enum class ERLAttackShape : uint8
>       {
>           Box,
>           Location,
>           Projectile,
>           Sector  // 보스 자신 중심의 부채꼴(전방위) 판정
>       };
>       ```
>
> `FRLMonsterPatternData`에는 `SectorSafeZoneAngle`(기본 90도)을 추가했다. `AttackShape == Sector`일 때만 편집 가능하게 `EditCondition`을 걸어뒀다.
>
> ## 판정: 원형 스윕 + 각도 필터링
>
> 언리얼에 부채꼴 전용 트레이스 함수가 따로 있는 건 아니라서, `CreateLocationHitbox`와 똑같이 `SphereTraceMultiForObjects`로 반지름 전체를 스윕한 다음, 결과에서 각도로 안전지대만 걸러내는 방식을 썼다.
>
> ```cpp
> const float AngleFromForward = FMath::RadiansToDegrees(
>     FMath::Acos(FVector::DotProduct(ForwardDir, ToTarget2D)));
>
> if (AngleFromForward >= (180.0f - SafeHalfAngle))
> {
>     continue; // 뒤쪽 안전지대 쐐기 안 → 데미지 제외
> }
> ```
>
> 정면 벡터와 대상 방향 벡터의 내적을 각도로 바꾼 다음, 그 각도가 "180도 - 안전지대 절반 폭" 이상이면(=거의 정후방이면) 안전지대로 보고 건너뛴다. 나머지는 전부 맞는다.
>
> ## 데칼: 모양은 머티리얼에서, 파라미터는 C++에서
>
> `ARLDecalIndicator`에 원래 선언만 있고 쓰이지 않던 `EIndicatorType::Cone`을 실제로 활용해서 `SetupSectorIndicator`를 추가했다. 데칼 자체는 원형 풋프린트로 깔아두고, "뒤쪽만 잘라내는" 모양은 머티리얼의 UV 각도 계산으로 처리하도록 설계했다 — 데칼 컴포넌트 자체는 사각형/원형 볼륨만 표현할 수 있고, 임의 각도로 잘린 모양은 셰이더의 픽셀 단위 마스킹으로만 가능하기 때문이다.
>
> 안전지대 폭 값은 머티리얼 애셋을 직접 수정하지 않고 `UMaterialInstanceDynamic`을 만들어서 `SafeZoneAngleDeg`라는 이름의 Scalar Parameter로 매 호출마다 밀어넣는 방식으로 처리했다. 원본 머티리얼에 그 이름의 파라미터가 없으면 그냥 조용히 무시되도록 해서, 아직 전용 머티리얼이 없어도 컴파일/실행에는 문제가 없다.
>
> ## 남은 일
>
> 지금 이 시점까지 한 건 C++ 쪽 배관 작업뿐이다. 아직 안 된 것들:
>
> - **선행 확인**: `AM_Boss_BigSwing`의 `Notify_Hit`이 스켈레톤 노티파이인지 Montage Notify인지 확인 필요. 스켈레톤이면 지난 글의 버그와 똑같은 이유로 이 시스템 전체가 안 먹는다.
> - - `DA_Enemy_Boss`에서 BigSwing 패턴의 `AttackShape`를 `Location`에서 `Sector`로 변경
>   - - `BTT_Attack`에 Sector 케이스 배선: 사전동작 시작 시 `StartSectorAttackTelegraph` 호출, `Notify_Hit`에서 `CreateSectorHitbox` 호출
>     - - 부채꼴 전용 머티리얼 신규 제작 (기존 원형/사각형 머티리얼 재사용 불가, UV 각도 기반 마스킹 필요)
>       - - 실제 PIE로 각도 판정 방향과 데칼 크기가 의도대로 나오는지 검증
>        
>         - 다음 글은 이걸 실제로 켜본 다음, 어디가 맞았고 어디가 틀렸는지 정리하는 내용이 될 것 같다.
>         - 
