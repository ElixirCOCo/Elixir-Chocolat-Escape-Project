# 📜 AI 협업 공통 계약서 — shared_contract.md
> 프로젝트: 쇼콜라의 칼퇴 대작전  
> 작성: 엘릭서 & Claude & Gemini & GPT  
> ⚠️ 모든 AI에게 작업 지시 전 이 파일을 반드시 먼저 입력하고 "이 규칙을 엄격히 준수하라"고 명령할 것.  
> ⚠️ 최종 결정권자: 엘릭서. AI는 전부 참고 의견.  
> ⚠️ 타임박스 룰: 30분 고민 → 결정. 1시간 넘으면 → 그냥 구현하고 테스트.

---

## 1. 기술 스택 및 기본 클래스

| 항목 | 내용 |
|------|------|
| Engine | Unity 6 (6000.3.13f1, URP) |
| Networking | Netcode for GameObjects (NGO) v1.5.0+ |
| Asset Pipeline | Addressables (Resources 폴더 사용 절대 금지) |
| Base Class | 네트워크 개체: `NetworkBehaviour` 상속 / 순수 로직: 일반 `class` |

---

## 2. 네임스페이스 규칙 (엄격히 구분)

```csharp
ChocolatEscape.Player       // 플레이어 이동·조작·카메라
ChocolatEscape.Enemy        // 빌런 AI·스폰·웨이브
ChocolatEscape.Interaction  // IInteractable·아이템·문
ChocolatEscape.Data         // ScriptableObject·IDamageable·IStatModifier
ChocolatEscape.Network      // NetworkManager·RPC·동기화
ChocolatEscape.UI           // HUD·메뉴·씬 전환
ChocolatEscape.Character    // ICharacter·캐릭터 교체 시스템
```

---

## 3. ★ 핵심 아키텍처 — PlayerRoot 계층 구조 (GPT 최종 확정)

> "Visual은 갈아끼운다, Core는 절대 안 건드린다" — GPT

### Player 오브젝트 계층 구조 (필수)

```
PlayerRoot                  ← 핵심 기준점
 ├─ PlayerCore (로직)       ← 순수 C# 클래스
 ├─ CharacterController     ← 충돌·이동
 ├─ CameraPivot             ← 카메라 높이 기준 (캐릭터마다 값만 조정)
 └─ VisualRoot              ← ★ 아바타는 여기에만 붙임
      └─ Character (쇼콜라 or 교체 대상)
           ├─ Animator (Humanoid 필수)
           ├─ Hand_R_Socket  ← 무기 위치 기준
           ├─ Hand_L_Socket
           └─ WeaponMount
```

### 캐릭터 교체 3단계 전략
```
1단계 (수업용 지금)
  쇼콜라 그대로 사용
  구조(PlayerRoot/VisualRoot)만 제대로 분리

2단계 (프로토타입 완성 후) ← 반드시 실행
  교체 테스트 1번 실시
  IK·Offset 보정 범위 파악
  이 단계 건너뛰면 출시 전 지옥 확정

3단계 (출시 전)
  오리지널 캐릭터 제작
  Animation Rigging + IK 손/총 위치 보정 완료
```

### 캐릭터 교체 현실 난이도
| 작업 | 난이도 | 설명 |
|------|--------|------|
| 모델 교체 | ⭐ | VisualRoot 하위 교체 |
| 애니메이션 유지 | ⭐⭐ | Humanoid면 대부분 작동, 미세 조정 필요 |
| 총/손 위치 보정 | ⭐⭐⭐ | 캐릭터마다 offset 값 조정 필요 |
| 카메라/충돌 보정 | ⭐⭐⭐ | CameraPivot 높이·CharacterController 반경 |

> ⚠️ "완전 자동 교체"는 없다. 미세 조정은 항상 필요하다.

---

## 4. ★ 캐릭터 시스템 — ICharacter 인터페이스 (필수)

> 캐릭터 전용 로직 하드코딩 절대 금지  
> 모든 캐릭터 행동은 ICharacter 인터페이스로 추상화

```csharp
namespace ChocolatEscape.Character {
    public interface ICharacter {
        void PlayHit();           // 피격 반응
        void PlayShoot();         // 사격 모션
        void PlayReload();        // 재장전 모션
        void PlayDance(int index); // 댄스 (H키, 3종)
        void PlayEscape();        // 탈출 성공 연출
        void PlayFail();          // 패배 연출

        // 캐릭터별 오프셋 (손/총 위치 보정)
        Vector3 HandROffset { get; }
        Vector3 HandLOffset { get; }
        float CameraPivotHeight { get; }
    }
}
```

**위반 검증:**
```csharp
// ❌ 절대 금지 — 캐릭터 하드코딩
if (characterName == "chocolat") { PlayChocolatDance(); }

// ✅ 올바른 방법
_character.PlayDance(danceIndex);
```

---

## 5. ★ 애니메이션 — IK 보정 규칙 (GPT 지뢰 방지)

> Humanoid면 80% 호환, 나머지 20%가 지옥이다 — GPT

### 필수 적용 사항
- **Animation Rigging 패키지** 반드시 사용 (총/손 위치 IK 보정)
- **Hand_R_Socket / Hand_L_Socket** 기준으로 무기 배치
- 캐릭터마다 `ICharacter.HandROffset` 값으로 개별 보정

### Humanoid 교체 시 체크리스트
- [ ] 팔 길이 차이 → IK로 보정됐는가?
- [ ] 총이 얼굴에 박히지 않는가?
- [ ] ADS 조준선이 크로스헤어와 일치하는가?
- [ ] 발이 땅에 박히지 않는가?
- [ ] 카메라 높이가 눈 위치와 일치하는가?

---

## 6. PlayerCore / PlayerNetwork 강제 분리

> GPT: 이거 슬로건으로 끝내면 3일 뒤 코드에서 네트워크 냄새 진동한다.

```csharp
// ✅ 순수 로직 — NetworkBehaviour 절대 몰라야 함
namespace ChocolatEscape.Player {
    public class PlayerCore {
        public PlayerStats stats;
        private ICharacter _character; // 캐릭터는 인터페이스로만 참조

        public Vector3 CalculateMove(Vector2 input, Transform t, float dt) { ... }
        public bool TryGetShootTarget(Ray ray, float range, out RaycastHit hit) { ... }
    }
}

// ✅ 네트워크 래퍼 — 전달자 역할만
namespace ChocolatEscape.Network {
    public class PlayerNetwork : NetworkBehaviour {
        private PlayerCore _core = new PlayerCore();
        void Update() {
            if (!IsOwner) return;
            _core.CalculateMove(GetInput(), transform, Time.deltaTime);
            SubmitMoveServerRpc(GetInput());
        }
    }
}
```

---

## 7. 서버 권한 기준표

| 항목 | 권한 | 방식 |
|------|------|------|
| 이동 입력 | Owner | `if (!IsOwner) return` |
| 공격 입력 | Owner | `if (!IsOwner) return` |
| 공격 판정 (Raycast) | Server | `[ServerRpc]` |
| 대미지 적용 | Server | `[ServerRpc]` |
| 아이템 획득·스폰 | Server | `[ServerRpc]` |
| 게임 승패 판정 | Server | Server 직접 처리 |
| 빌런 웨이브 스폰 | Server | Server 직접 처리 |
| UI 업데이트 | Client | `OnValueChanged` |
| 이펙트·사운드·파티클 | Client | `[ClientRpc]` |

### ★ Raycast 권한 규칙
```
Owner에서 Raycast → ServerRpc로 전달 → Server에서 판정
```

---

## 8. NetworkVariable — 사용 기준

| 사용 ✅ | 금지 ❌ |
|--------|--------|
| HP | 애니메이션 상태 |
| 탄창 수 | 사운드 재생 |
| 게임 시간 | 파티클 연출 |
| 게임 승패 상태 | 이동 보간 연출 |

---

## 9. 핵심 인터페이스 정의

```csharp
namespace ChocolatEscape.Interaction {
    public interface IInteractable {
        string InteractionPrompt { get; }
        void Interact();
    }
}

namespace ChocolatEscape.Data {
    public interface IDamageable {
        void TakeDamage(float damage, Vector3 hitPoint);
        bool IsDead { get; }
    }

    // 런타임 버프·디버프 — 직접 수치 박기 절대 금지
    public interface IStatModifier {
        void Apply(PlayerStats stats);
        void Remove(PlayerStats stats);
        string ModifierName { get; }
    }
}
```

---

## 10. 애니메이션 동기화 규칙

```
✅ 허용: Animator 파라미터(Float, Bool, Int)만 동기화
❌ 금지: NetworkTransform을 VRC 본(Bone)마다 붙이는 것
❌ 금지: 캐릭터 이름으로 분기하는 하드코딩
```

---

## 11. 에셋 관리 규칙

```
❌ 절대 금지: Resources 폴더 사용
✅ 허용: Addressables 또는 Serialized Reference
```

---

## 12. 이벤트 명명 규칙

```csharp
// ✅ On + 과거형
public static event Action OnPlayerDied;
public static event Action<ItemData> OnItemPickedUp;
public static event Action OnGateOpened;
public static event Action OnCharacterSwapped; // 캐릭터 교체 시
```

---

## 13. 코드 품질 규칙

| 규칙 | 내용 |
|------|------|
| PlayerRoot 구조 | VisualRoot 분리 필수, 아바타는 VisualRoot 하위에만 |
| ICharacter 필수 | 캐릭터 전용 로직 하드코딩 절대 금지 |
| Humanoid 리깅 | 모든 아바타는 Humanoid 설정 필수 |
| IK 보정 | Animation Rigging으로 손/총 위치 보정 |
| PlayerCore 순수성 | NetworkBehaviour·IsOwner·RPC 절대 금지 |
| 매직 넘버 금지 | const 또는 ScriptableObject |
| Update() 캐싱 | GetComponent·Find는 Awake/Start에서만 |
| 이벤트 해제 | OnDestroy에서 반드시 `-=` |
| Resources 금지 | Addressables 또는 Serialized Reference만 |
| IStatModifier | 런타임 수치 변경은 반드시 인터페이스로 |

---

## 14. 작업 지시 템플릿 (AI에게 복붙해서 사용)

```
[공통 계약서 준수 명령]
아래 규칙을 엄격히 준수하여 코드를 작성하라.

- Engine: Unity 6 (6000.3.13f1, URP)
- Networking: Netcode for GameObjects (NGO) v1.5.0+
- 네임스페이스: ChocolatEscape.{Player/Enemy/Interaction/Data/Network/UI/Character}
- 아키텍처: PlayerRoot > VisualRoot > Character 계층 구조 필수
- 아바타는 반드시 VisualRoot 하위에만 배치
- ICharacter 인터페이스로 모든 캐릭터 행동 추상화
- 캐릭터 이름 하드코딩 절대 금지
- Humanoid 리깅 + Animation Rigging IK 사용
- PlayerCore: NetworkBehaviour·IsOwner·RPC 절대 몰라야 함
- Raycast: Owner에서 쏘고, 판정은 ServerRpc에서
- 서버 권한: 공격판정·대미지·아이템획득·승패판정 → [ServerRpc]
- NetworkVariable: HP·탄창·게임시간·승패 상태만
- 시각·청각 연출: [ClientRpc] 또는 OnValueChanged
- 애니메이션: 파라미터만 동기화, 본 동기화 금지
- IStatModifier: 런타임 수치 변경은 인터페이스로, 직접 박기 금지
- Resources 폴더 절대 금지
- 이벤트: On + 과거형
- Update() 내 GetComponent 금지

[작업 내용]
(여기에 구체적인 요청 작성)
```

---

## 📅 변경 이력
| 날짜 | 내용 | 작성자 |
|------|------|--------|
| 2025-04-23 | 초안 작성 | Gemini |
| 2025-04-23 | 작업 지시 템플릿 추가 | Claude |
| 2025-04-23 | 로직/네트워크 레이어 분리, IStatModifier, Resources 금지 | GPT 1차 피드백 |
| 2025-04-23 | PlayerCore/Network 강제 구조, 서버 권한 기준표 | GPT 2차 피드백 |
| 2025-04-23 | Unity 6 버전 수정, PlayerRoot 계층 구조, ICharacter 인터페이스, IK 보정 규칙, 캐릭터 교체 전략 추가 | GPT 3차 피드백 |
