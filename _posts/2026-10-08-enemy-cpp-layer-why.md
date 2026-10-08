---
layout: post
title: 적 캐릭터 사이에 C++ 계층을 하나 끼운 이유
date: 2026-10-08 10:00:00 +0900
categories:
  - devlog
  - architecture
  - ai
---

몬스터 계층을 이렇게 잡았다. ARLCharacter, 그 아래 적 전용 C++ 클래스인 ARLEnemyCharacter, 그 아래 블루프린트 BP_EnemyBase, 그리고 BP_Boss 같은 개별 몬스터. BP_EnemyBase가 ARLCharacter를 바로 상속해도 됐을 텐데 중간에 C++ 계층을 하나 끼운 데에는 이유가 있다.

## 출발점은 비헤이비어 트리 태스크였다

패턴 선택을 맡는 BTT_SelectPattern과 공격을 실행하는 BTT_Attack은 컨트롤하는 폰을 BP_EnemyBase로 캐스트해서 CurrentSelectedPattern이나 TrySelectPattern 같은 걸 쓰는 블루프린트 태스크다. BTT_SelectPattern에는 대응하는 C++ 클래스가 없고 에셋만 있다. 그러면 이 태스크들이 아는 몬스터 타입은 BP_EnemyBase 하나뿐이라서, 보스를 BP_EnemyBase와 무관한 별도 C++ 서브클래스로 만들면 캐스트가 실패한다. 보스도 결국 BP_EnemyBase에서 파생된 블루프린트여야 한다는 제약이 생긴다.

## 그럼 C++로 뭘 올렸나

BP_EnemyBase 위로 올리고 싶은 건 모든 적이 공통으로 필요한 규칙이었다. 보스, 엘리트, 경직 면역 플래그(bIsBoss, bIsElite, bImmuneToStagger)와 그걸 반영하는 CanBeStaggered 오버라이드, 처치 시 경험치와 골드, 아이템 드롭, 체력이 일정 비율 아래로 내려가면 격노 상태로 넘어가는 판정, 사망 후 페이드 같은 것들이다. 이런 걸 몬스터 블루프린트마다 복붙하면 규칙 하나를 바꿀 때 전부 열어서 고쳐야 한다. 한 곳에 두면 규칙이 한 곳에서 바뀐다.

연출은 훅으로 넘겼다. 예를 들어 격노 판정은 C++이 하고, OnEnraged라는 BlueprintImplementableEvent를 호출해서 실제로 어떤 이펙트를 붙일지는 블루프린트에서 정한다. 언제 일어나는지는 C++, 어떻게 보이는지는 BP라는 분담이다.

## 반대로 C++로 안 올린 것

패턴 선택은 블루프린트 태스크로 남아 있다. 쿨타임이 안 지난 패턴을 거르고, 남은 패턴 중에서 가중치로 하나를 뽑는 정도의 로직이다. 대신 패턴 데이터는 FRLMonsterPatternData 구조체와 데이터 에셋으로 뺐기 때문에, 몬스터를 늘릴 때는 데이터 행만 추가하면 된다.

## 이 구조의 한계

계층이 네 단계로 깊어진다. 그리고 BT 태스크가 BP_EnemyBase를 캐스트 대상으로 삼는 이상 보스 전용 C++ 클래스를 따로 만들 수 없어서, 보스 전용 기능도 ARLEnemyCharacter에 플래그로 얹게 된다. 이 클래스가 점점 커질 수 있다는 뜻이다. 보스 전용 기믹이 늘어나면 그때 가서 다시 나눌지 고민해야 할 것 같다.

## 정리

계층을 정한 건 취향이 아니라 BT 태스크의 캐스트 제약이었다. 공통 규칙은 C++, 개별 몬스터의 데이터와 연출은 BP로 나누는 게 지금 구조에서는 무난했다.
