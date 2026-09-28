---
layout: post
title: "보스가 안 움직이는 버그, 원인은 나눗셈이었다"
date: 2026-09-20 12:00:00 +0900
categories: [devlog, bugfix]
---

오늘은 보스 몬스터(BP_Boss)가 스폰 직후 전혀 움직이지 않고, 공격 모션도 제대로 재생되지 않는 문제를 추적했다. 결과물보다 "어떻게 원인을 좁혀나갔는지" 과정 자체가 기록해둘 가치가 있어서 devlog에 남긴다.

## 증상

- 보스가 스폰된 뒤 플레이어를 인식해도 제자리에서 움직이지 않음
- - 공격 패턴(Attack Montage)이 실행되긴 하는데, 눈에 띄는 애니메이션 변화가 거의 없음
 
  - ## 1차 확인: Behavior Tree는 정상이었다
 
  - PIE 로그를 보면 `0 1 2 3` / `0 1 2` / `0 1` 식으로 매번 다른 개수의 값이 찍히는데, 처음엔 이게 버그처럼 보였다. 확인해보니 이건 `SelectionWeight`(가중치) 기반으로 4개의 공격 패턴 중 하나를 랜덤하게 고르는 로직이라, 몇 번 재시도 끝에 걸리기도 하고 바로 걸리기도 하는 정상 동작이었다. 라이브로 BT를 직접 열어서 실행 흐름을 지켜본 결과도 동일해서, **BT 자체는 원인이 아님**을 확인했다.
 
  - ## 2차 확인: 몽타주 스켈레톤 문제도 아니었다
 
  - 몽타주가 다른 스켈레톤용이라 재생이 씹히는 경우 보통 다음과 같은 경고가 뜬다.
 
  - ```
    Playing a montage for the wrong skeleton
    ```

    로그 전체를 검색했지만 이 경고는 한 번도 찍히지 않았다. 그래서 이 가능성도 배제.

    ## 3차 확인: 로그에 반복적으로 찍히는 경고 발견

    대신 PIE를 시작하자마자부터, 공격 여부와 무관하게 매 틱마다 아래 경고가 계속 출력되고 있었다.

    ```
    LogScript: Warning: Script Msg: Divide by zero: Divide_DoubleDouble
    LogScript: Warning: Script Msg called by: Sevarog_AnimBlueprint_C .../BP_Boss_C_X.CharacterMesh0.Sevarog_AnimBlueprint_C_0
    ```

    즉 보스가 쓰는 `Sevarog_AnimBlueprint`의 AnimGraph 안에서 어떤 `Divide` 노드가 매 프레임 분모를 0으로 나누고 있다는 뜻이다.

    ## 가설: MaxWalkSpeed가 0이다

    이런 종류의 AnimBP는 보통 로코모션 블렌드스페이스에서 `현재 속도 / 최대 이동 속도(MaxWalkSpeed)`를 계산해서 블렌드 값으로 쓴다. 만약 `CharacterMovement`의 `MaxWalkSpeed`(또는 스탯 컴포넌트에서 이동속도를 세팅하는 구조라면 그 스탯 값)가 0이라면 세 가지가 동시에 설명된다.

    1. AI가 `Move To`를 실행해도 실제 이동 속도가 항상 0이라 안 움직이는 것처럼 보인다.
    2. 2. 동시에 AnimGraph의 `속도 / 최대속도` 나눗셈이 매 틱 0으로 나뉘면서 지금 로그에 찍히는 경고가 뜬다.
       3. 3. 이 나눗셈 결과가 NaN이 되면 블렌드/슬롯 처리가 꼬여서, Attack 몽타주가 재생되어도 시각적으로 어색하게 나오거나 거의 안 보이게 될 수 있다.
         
          4. "안 움직임"과 "공격해도 티가 안 남", 서로 무관해 보이던 두 증상이 **이동속도 값 하나(0)** 로 같이 설명될 가능성이 높다는 게 오늘까지의 결론이다.
         
          5. ## 다음에 확인할 것
         
          6. - `BP_Boss`의 Character Movement: Walking 카테고리에서 `Max Walk Speed` 값 확인 (0이거나 비정상적으로 낮은지)
             - - 이동속도를 스탯 컴포넌트(`URLStatComponent`)로 관리하는 구조라면, 보스 전용 스탯 데이터(DataTable/DataAsset)에 MoveSpeed가 세팅되어 있는지, `BeginPlay`에서 스탯 값을 `MaxWalkSpeed`에 대입하는 로직이 실제로 있는지
               - - `Sevarog_AnimBlueprint`의 AnimGraph에서 `Divide_DoubleDouble` 노드를 직접 찾아 정확히 뭘 나누고 있는지 확인
                
                 - 다음 글에서 실제로 원인이 맞았는지, 어떻게 고쳤는지 이어서 정리할 예정이다.
                 - 
