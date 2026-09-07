# It Takes Two 모작 · 공구통 보스전

**못을 던져 목표물에 박고 회수하는 전투와, 못에 망치를 걸어 이동하는 협동 액션을 Unreal Engine 5로 구현한 팀 프로젝트입니다.**

[▶ 실행 영상 보기](https://www.youtube.com/watch?v=Ag4aGCEbZQQ)

| 항목 | 내용 |
|---|---|
| 개발 기간 | 2024.07–2024.08 · 메타버스 아카데미 2차 융합 프로젝트 |
| 팀 규모 | 6명 |
| 담당 | **한승우 · 클라이언트 개발** — 못 전투·회수·보관, 망치 상호작용 |
| 개발 범위 | 협동 플랫포머 *It Takes Two*의 공구통 보스전 모작 |
| 핵심 기술 | Unreal Engine 5.4, C++, Blueprint, FSM, Object Pooling, TArray, Socket |

## 먼저 볼 구현 3가지

| 구현 | 살펴볼 내용 | 코드 바로가기 |
|---|---|---|
| **못 전투 · FSM** | 장전·발사·박힘·회수 상태에 따라 행동을 분리 | [HSW_Bullet.cpp](MTVS_ItTakesTwo/Source/MTVS_ItTakesTwo/Private/HSW_Bullet.cpp#L79) · [상태 선언](MTVS_ItTakesTwo/Source/MTVS_ItTakesTwo/Public/HSW_Bullet.h#L10) |
| **못 재사용 · 회수 오류 해결** | 3개의 못을 재사용하고, 돌아온 객체와 보관되는 객체를 일치시킴 | [HSW_BulletManager.cpp](MTVS_ItTakesTwo/Source/MTVS_ItTakesTwo/Private/HSW_BulletManager.cpp#L20) → [NailPush](MTVS_ItTakesTwo/Source/MTVS_ItTakesTwo/Private/HSW_BulletManager.cpp#L77) |
| **망치 상호작용 · 진자 움직임** | 못에 접근해 매달리고 Sin 함수로 주기적인 회전 구현 | [HSW_Hammer.cpp](MTVS_ItTakesTwo/Source/MTVS_ItTakesTwo/Private/HSW_Hammer.cpp#L122) → [HammerRotation](MTVS_ItTakesTwo/Source/MTVS_ItTakesTwo/Private/HSW_Hammer.cpp#L201) |

## 담당 역할

- C++·Blueprint 기반 못 발사·충돌·회수 시스템 구현
- 못의 상태를 열거형으로 정의하고 FSM으로 상태 전환 관리
- 3개의 못 객체를 미리 생성해 재사용하고, 보관 배열과 박힌 못 배열 관리
- 회수되는 못과 보관 배열에 등록되는 객체가 어긋나는 문제 수정
- 보관함 Socket에 못을 배치해 보관·장전 상태를 시각적으로 표현
- 망치가 못에 매달려 주기적으로 흔들리는 상호작용 구현

이 README는 팀의 전체 보스전 구현 중 **한승우의 못·망치 기능**을 중심으로 안내합니다. 입력과 캐릭터 연동을 확인하기 위해 연결한 `CSR_*` 파일은 팀원의 연동 코드이며, 파일 전체를 개인 구현으로 표시하지 않습니다.

## 1. 못 전투 — 상태에 따라 역할 분리

핵심 구현은 `HSW_Bullet.cpp`의 `Tick`과 `SetState`에 있습니다. 별도의 FSM 컴포넌트가 아니라 못 액터 안에서 상태별 동작을 분기합니다.

| 주요 상태 | 동작 |
|---|---|
| `BASIC` | 보관함 Socket의 위치·회전 반영 |
| `LOAD` | 장전 상태 처리 |
| `SHOOT` | 발사 후 이동, 시간 조건에 따른 자동 회수 |
| `EMBEDDED` | 목표물에 박힌 상태 유지 |
| `UNEMBEDDED` | 박히지 못한 상태 처리 후 자동 회수 |
| `RETURNING` | 플레이어 방향으로 복귀하고 도착 조건에서 보관 등록 |

충돌 시 대상의 `NailTag`를 확인해 박힘 여부를 구분합니다. 수동 회수는 박힌 못 배열에서 대상을 선택하고, 자동 회수는 못의 상태·시간 조건에 따라 시작합니다.

매 프레임 현재 상태에 해당하는 함수만 호출합니다. 발사 이동과 회수 이동을 각 함수에 나눠 두어 상태별 동작을 확인할 수 있습니다.

```cpp
// Tick의 상태 분기 발췌 · 공백 정리
switch (State)
{
case ENailState::BASIC:      TickBasic(DeltaTime);      break;
case ENailState::LOAD:       TickLoad(DeltaTime);       break;
case ENailState::SHOOT:      TickShoot(DeltaTime);      break;
case ENailState::EMBEDDED:   TickEmbedded(DeltaTime);   break;
case ENailState::UNEMBEDDED: TickUnembedded(DeltaTime); break;
case ENailState::RETURNING:  TickReturning(DeltaTime);  break;
case ENailState::GOTOBAG:    TickGoToBag(DeltaTime);    break;
}
```

원본: [HSW_Bullet.cpp · Tick](MTVS_ItTakesTwo/Source/MTVS_ItTakesTwo/Private/HSW_Bullet.cpp#L79-L95)

## 2. 회수 오류 해결 — 돌아온 못의 참조를 등록

### 객체 재사용의 기본 구조

시작할 때 못 3개를 생성해 `Magazine`에 보관합니다. 장전 시에는 `NailPop`으로 하나를 꺼내고, 회수 후 같은 객체를 다시 등록해 재사용합니다.

```cpp
// BeginPlay 핵심 발췌 · 생성 옵션, Socket 배치, 실패 로그 생략
for (int32 i = 0; i < 3; i++)
{
    // ... FActorSpawnParameters params 설정 ...
    Nail = GetWorld()->SpawnActor<AHSW_Bullet>(BulletFactory, params);
    if (Nail)
    {
        Nail->SetNailBag(this);
        Magazine.Add(Nail);
        // ... 보관함 Socket에 배치 ...
    }
}
```

원본: [HSW_BulletManager.cpp · BeginPlay](MTVS_ItTakesTwo/Source/MTVS_ItTakesTwo/Private/HSW_BulletManager.cpp#L20-L47)

### 문제와 원인

회수 순서가 달라질 때 못이 겹치거나 정상적으로 보관되지 않는 문제가 있었습니다. **회수할 못을 선택하는 일과, 실제 도착한 못을 보관하는 일을 구분해야 했습니다.**

회수 대상 선택과 도착 등록을 분리한 뒤에도, 등록 함수는 전달받은 `currentNail` 대신 관리자에 저장돼 있던 다른 참조 `Nail`을 배열에 넣고 있었습니다.

### 변경한 흐름

1. `NailOutPop`으로 수동 회수할 못을 선택하고 해당 못을 `RETURNING` 상태로 전환합니다.
2. 각 못은 `TickReturning`에서 플레이어와의 거리를 확인합니다.
3. 도착 거리 조건을 만족하면 `NailBag->NailPush(this)`로 **실제 돌아온 자신의 참조**를 전달합니다.
4. 관리자는 `Magazine.Push(currentNail)`로 받은 객체를 등록합니다.
5. 못은 `BASIC` 상태로 돌아가 배열 순서에 맞는 `NailBag_0`~`NailBag_2` Socket에 배치됩니다.

**① 도착한 못이 자신을 전달합니다.** `RETURNING`에 진입한 즉시 등록하는 것이 아니라, 플레이어와의 거리가 기준값보다 작아졌을 때 등록합니다.

```cpp
// TickReturning 핵심 발췌 · 이동 처리와 카메라 연출 생략
Distance = (Player->GetActorLocation() - this->GetActorLocation()).Size();
// ... 플레이어 방향으로 이동 ...
if (Distance < NailDefaultDist)
{
    // ... 카메라 연출 ...
    NailBag->NailPush(this);
    SetState(ENailState::BASIC);
}
```

원본: [HSW_Bullet.cpp · TickReturning](MTVS_ItTakesTwo/Source/MTVS_ItTakesTwo/Private/HSW_Bullet.cpp#L237-L266)

**② 관리자는 전달받은 참조를 그대로 등록합니다.** 이전에는 `currentNail`을 인자로 받아도 멤버 변수 `Nail`을 넣고 있어, 도착한 못과 등록한 못이 달라질 수 있었습니다.

```diff
// NailPush 내부의 실제 변경
- Magazine.Push(Nail);
+ Magazine.Push(currentNail);
```

원본: [HSW_BulletManager.cpp · NailPush](MTVS_ItTakesTwo/Source/MTVS_ItTakesTwo/Private/HSW_BulletManager.cpp#L77-L90) · [수정 커밋](https://github.com/H-SeungWoo/ItTakesTwo_copy/commit/0e4f4b98e80164c7ec5dd7419b88c33c3be28693)

**결과:** 회수된 객체와 배열에 등록되는 객체의 불일치를 수정했습니다. 회수 대상을 고르는 `Pop`은 유지하고, 도착 후 등록은 도착한 객체의 참조를 사용하도록 연결했습니다.

| 확인할 지점 | 파일 |
|---|---|
| 못 3개 생성·재사용 | [BulletManager::BeginPlay](MTVS_ItTakesTwo/Source/MTVS_ItTakesTwo/Private/HSW_BulletManager.cpp#L20) |
| 도착 조건·자기 참조 전달 | [Bullet::TickReturning](MTVS_ItTakesTwo/Source/MTVS_ItTakesTwo/Private/HSW_Bullet.cpp#L237) |
| 전달받은 참조의 실제 등록 | [BulletManager::NailPush](MTVS_ItTakesTwo/Source/MTVS_ItTakesTwo/Private/HSW_BulletManager.cpp#L77) |
| 보관함 Socket 선택 | [Bullet의 Socket 처리](MTVS_ItTakesTwo/Source/MTVS_ItTakesTwo/Private/HSW_Bullet.cpp#L371) |
| 입력에서 수동 회수까지 · 팀 연동 코드 | [CSR_CodyPile.cpp](MTVS_ItTakesTwo/Source/MTVS_ItTakesTwo/Private/CSR_CodyPile.cpp#L251) |

변경 이력: [회수 선택·도착 등록 분리](https://github.com/H-SeungWoo/ItTakesTwo_copy/commit/bd029f032dbd235f7dbefada0f355a94852af9d1) → [등록 참조를 currentNail로 수정](https://github.com/H-SeungWoo/ItTakesTwo_copy/commit/0e4f4b98e80164c7ec5dd7419b88c33c3be28693).

세 객체를 반복 사용하는 구조이며, FPS나 메모리 절감량을 별도로 측정한 것은 아닙니다.

## 3. 망치 상호작용 — 주기적인 흔들림 구현

`HSW_Hammer.cpp`에서 못과의 겹침을 감지하고, 접근 후 못의 `AttachingPoint` Socket에 망치를 배치합니다. 매달린 동안 누적 시간과 Sin 함수를 이용해 회전 각도를 계산합니다.

```cpp
// HammerRotation 핵심 발췌 · 주석 및 공백 정리
CurrentTime += DeltaTime;
float Angle = Amplitude * FMath::Sin(CurrentTime * Frequency * 1.5f * PI);
newRotation = FRotator(0.0f, 110.0f, Angle);
this->SetActorRelativeRotation(newRotation);
```

원본: [HSW_Hammer.cpp · HammerRotation](MTVS_ItTakesTwo/Source/MTVS_ItTakesTwo/Private/HSW_Hammer.cpp#L201-L211)

누적 시간에 따라 회전 각도가 양방향으로 반복됩니다. `Amplitude`는 흔들림의 크기를, `Frequency`는 빠르기를 조절하며, 계산한 각도를 실제 액터 회전에 적용합니다.

- `MoveToNail`: 못에 접근하고 매달림 상태로 전환
- `HammerRotation`: 진폭·주파수에 따른 회전 각도 계산
- `GetHammerSocketLocation`: 캐릭터 연결에 사용할 `PlayerAttachingPoint` 위치 제공

실제 중력·장력으로 계산하는 물리 시뮬레이션이 아니라, 게임 상호작용에 필요한 **진자 형태의 주기적 움직임**을 구현했습니다.

## 저장소 구조

```text
MTVS_ItTakesTwo/
  MTVS_ItTakesTwo.uproject     # Unreal Engine 5.4
  Source/MTVS_ItTakesTwo/
    Private/
      HSW_Bullet.cpp         # 못 상태·충돌·회수·Socket
      HSW_BulletManager.cpp  # 못 생성·보관·회수 대상 관리
      HSW_Hammer.cpp         # 망치 매달림·주기적 회전
      CSR_*.cpp              # 팀 캐릭터·입력 연동
    Public/                  # 상태·클래스 선언
  Content/                   # 맵·Blueprint·모델·애니메이션
  Config/                    # 엔진·맵·입력·충돌 설정
```

## 실행 환경과 시연

- 엔진: **Unreal Engine 5.4** (`MTVS_ItTakesTwo.uproject` 기준)
- Windows C++ 빌드 환경과 프로젝트에 설정된 플러그인이 필요합니다. 플러그인 목록은 [프로젝트 파일](MTVS_ItTakesTwo/MTVS_ItTakesTwo.uproject)에서 확인할 수 있습니다.
- 전체 저장소와 Content 에셋을 준비한 뒤 프로젝트 파일을 생성하고 `MTVS_ItTakesTwoEditor`를 빌드해 에디터에서 엽니다.
- 에디터 시작 맵은 `/Game/JBY/AlphaDemo`로 설정돼 있으며 `BetaDemo` 등 다른 맵도 포함됩니다. 패키지 기본 맵은 별도로 설정돼 있어 시작 맵과 같지 않습니다.
- Blueprint 연결·입력 장치·실행할 맵에 따라 동작 확인이 필요합니다. 현재 환경에서 엔진 빌드와 플레이를 다시 검증하지 않았으므로, 완성 당시 동작은 [실행 영상](https://www.youtube.com/watch?v=Ag4aGCEbZQQ)을 먼저 참고해 주세요.

## 추가 개선 대상으로 남긴 부분

- 보관 배열에 같은 객체를 중복 등록하는 경우와 용량 경계에 대한 방어 조건 보강
- 고정 Lerp 계수를 사용하는 회수 이동의 프레임 속도 의존성 개선

위 항목은 현재 소스를 읽으며 구분한 후속 개선 대상입니다. 이번 README 정비에서는 게임 코드와 에셋을 변경하지 않았습니다.

본 프로젝트는 *It Takes Two*의 공구통 보스전을 학습 목적으로 모작한 교육 프로젝트이며, 원작의 공식 프로젝트가 아닙니다.
