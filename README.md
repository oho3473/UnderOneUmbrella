# UnderOneUmbrella



### 프로젝트 개요


"한우산아래"는 우산을 활용해 주변 환경과 상호작을 통해 퍼즐을 해결하는 3D 퍼즐 게임으로, 현재 2026 코리아인디게임데브캠프 4차 심사 및 개발중인 프로젝트입니다.
"한우산아래" 프로젝트에서 기여한 작업 일부를 정리한 리포지토리입니다.


| 항목 | 내용 |
| --- | --- |
| **프로젝트명** | 한우산아래 |
| **플랫폼** | PC,Moblie |
| **개발 언어** | C++ |
| **사용 기술** | Unreal 5.7, Codex |
| **담당 작업** | 퍼즐 진행 관계를 분석하고 자동 실행하는 개발 도구 구현, 플레이어·NPC·VFX·퍼즐 상태를 확인하기 위한 통합 디버그 기능 구현, 수집 연출의 기능 구현, 물에 반응하는 퍼즐 시스템 설계 및 구현|



### 1. 퍼즐 관계 파악 및 재현 기능

<p align="center">
<img width="426" height="240" alt="비교" src="https://github.com/user-attachments/assets/86bd3c61-48a5-4fb1-abe5-6a16ff1faf90" />
<sub><b><br> 좌 - 개발 툴 사용을 통한 검증(약 18초) / 우 - 직접 플레이하여 검증(약 31초) </b></sub>


| 필요 정보 | 내용 |
| --- | --- |
| - 현재 조건이 만족됐는가 <br />- 어떤 입력 조건을 사용하는가 <br />- 디버깅 입력을 어떻게 처리하는가 | 공통 정보를 바탕으로 서로 다른 퍼즐들을 동일하게 처리할 수 있도록 PuzzleConditionSource 컴포넌트로 구성 | 

| 공개 코드 | 역할 |
|---|---|
| [PuzzleConditionSourceComponent.h](https://github.com/oho3473/UnderOneUmbrella/blob/main/DebugCheatTool/UOUPuzzleConditionSourceComponent.h) · [구현](https://github.com/oho3473/UnderOneUmbrella/blob/main/DebugCheatTool/UOUPuzzleConditionSourceComponent.cpp) | 조건 상태, 입력 관계, 치트 처리의 공통 진입점 |
| [PuzzleCheatGraphTypes.h](https://github.com/oho3473/UnderOneUmbrella/blob/main/DebugCheatTool/UOUDevelopmentPuzzleCheatGraphTypes.h) | 노드·간선·외부 입력·Wave와 실행 상태 |
| [PuzzleResultCompletionState.h](https://github.com/oho3473/UnderOneUmbrella/blob/main/DebugCheatTool/UOUPuzzleResultCompletionState.h) | 결과 동작의 완료 여부를 확인하는 인터페이스 |



World 전체의 퍼즐 관계는 World Subsystem에서 관리

<br>
<img width="545" height="466" alt="image" src="https://github.com/user-attachments/assets/a1997a40-e0d8-4955-8aa9-4bff53539960" />

현재 World에 존재하는 PuzzleConditionSource를 가진 퍼즐을 일괄 수집하고, 입력·결과 참조를 기반으로 관계 그래프를 생성 관리했습니다.
TSet을 사용하여 중복 방문을 제거하고, 수집된 노드를 실행 깊이에 따라 Wave로 구성해 실제 진행 순서를 유지합니다.

```cpp
//중복 방지
TSet<int32> RequiredNodeIndices;

//탐색 대상 관리
TArray<int32> NodesToVisit;
NodesToVisit.Add(TargetNodeIndex);

while (!NodesToVisit.IsEmpty())
{
    const int32 NodeIndex = NodesToVisit.Pop(EAllowShrinking::No);

    if (RequiredNodeIndices.Contains(NodeIndex))
    {
        continue;
    }

    RequiredNodeIndices.Add(NodeIndex);

    NodesToVisit.Append(PuzzleGraphNodes[NodeIndex].PrerequisiteNodeIndices);
}
```

수집한 노드는 실행 깊이로 정렬하고, 이미 만족한 그룹을 제외해 Wave로 구성합니다.
각 Wave는 기존 ConditionGroup의 치트 진입점을 통해 활성화합니다.

결과 동작의 지연 시간과 완료 여부를 확인한 뒤 다음 Wave로 진행합니다.
```cpp
// CheckCurrentGraphWaveCompletion 내부 발췌
// World·노드·PuzzleGroup 유효성 검사를 통과한 이후의 처리
const double ElapsedSeconds = World->GetTimeSeconds() - ActiveNode.StartTimeSeconds;
if (ElapsedSeconds < ActiveNode.MinimumWaitSeconds
    || !ArePuzzleGroupResultsCompleted(PuzzleGroup))
{
    ScheduleGraphCompletionCheck();
    return;
}
```

### 2. 상태를 관측하는 디버그 기능
<img width="1338" height="734" alt="image" src="https://github.com/user-attachments/assets/9fc6f333-05ec-4799-ba92-42ec0886b7f2" />


각 객체는 인터페이스를 통해 상태 요약, 표시 위치, 연결 관계를 제공합니다.

| 공개 코드 | 역할 |
|---|---|
| [DebugProvider.h](https://github.com/oho3473/UnderOneUmbrella/blob/main/DebugCheatTool/UOUDebugProvider.h) · [구현](https://github.com/oho3473/UnderOneUmbrella/blob/main/DebugCheatTool/UOUDebugProvider.cpp) | 디버그 정보 제공 계약과 기본 동작 |

### Provider 수집과 표시

`DebugDrawSubsystem`은 World의 Actor와 Component를 탐색해 같은 등록 함수로 전달합니다.
등록 시 인터페이스와 Puzzle 카테고리를 확인하고, 객체 수명을 소유하지 않는 약한 참조로 보관합니다.

```cpp
// UOUDevelopmentDebugDrawSubsystem.cpp 발췌
void UUOUDevelopmentDebugDrawSubsystem::TryAddPuzzleDebugProvider(
    UObject* ProviderObject)
{
    if (!IsValid(ProviderObject)
        || !ProviderObject->GetClass()->ImplementsInterface(
            UUOUDebugProvider::StaticClass())
        || IUOUDebugProvider::Execute_GetDebugCategory(ProviderObject)
            != EUOUDebugCategory::Puzzle)
    {
        return;
    }

    PuzzleDebugProviders.AddUnique(TWeakObjectPtr<UObject>(ProviderObject));
}
```

표시 시에는 참조의 유효성, 액터 선택 상태, Provider 활성 여부를 확인합니다.
이후 인터페이스에서 받은 정보를 월드 라벨과 연결선으로 표시합니다.

처리 흐름:
`Actor·Component 탐색 → Provider 등록 → 표시 조건 확인 → 정보 조회 → 라벨·연결선 표시`

