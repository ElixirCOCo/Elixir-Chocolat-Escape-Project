# 🎮 유니티 게임 개발 시 주의사항 TODO
> 목적: 유니티 기반 게임 프로젝트에서 AI 협업 시 빼먹기 쉬운 핵심 원칙 명시  
> 대상: 장르 무관 모든 유니티 프로젝트에 공통 적용  
> 작성: 엘릭서 & Claude & Gemini & GPT

---

## ⚠️ AI 협업 시 필독 (최우선)

> 코드를 짜기 전에 반드시 이 파일 전체를 읽어라.  
> 초반에 무시하면 프로젝트 중반에 전체 리팩토링이 필요해진다.

### 🏆 핵심 원칙 한 줄
> **"먼저 재밌게 만들고, 나중에 있어 보이게 만들어라 (Fun First, Shine Later)"**

### ⏱️ 타임박스 룰
- **30분 고민 → 결정**
- **1시간 넘으면 → 그냥 구현하고 테스트**
- 게임 개발은 "맞는 선택"보다 "빠른 선택"이 더 중요하다

---

## 🎮 0. 재미를 기능보다 먼저 고정하라

### 개발 우선순위 Tier

> 뭔가 추가하고 싶을 때: **"이거 없으면 재미가 죽냐?"**  
> YES → 바로 구현 / NO → 뒤 Tier로 미루기

| Tier | 내용 | 기준 |
|------|------|------|
| **0** | 이동 · 사격 · HitStop | 없으면 게임 아님 |
| **1** | 적 추격 · 피격 · 타이머 · 승패 | 플레이 가능 상태 |
| **2** | 탄환 궤적 · 사운드 · UI | 무게감 |
| **3** | ScriptableObject · 카메라 연출 | 완성도 |

> ⚠️ 마감 직전까지 Tier 0~1이 완벽하지 않으면 Tier 2~3은 과감히 포기한다

### 개발 순서 원칙 (이 순서 그대로)
```
1. 이동 만든다
2. 총 쏜다
3. 맞으면 HitStop (이 순간이 손맛의 70%)
4. 적이 쫓아온다
5. 타이머 압박 속에 탈출한다
6. "더 먹을까 탈출할까" 고민 생긴다
```

### 감성 설계 원칙
- **안내하지 말고 부추겨라** — HUD는 공지사항이 아닌 감정 조작 도구
- **실패도 재미있어야 한다** — 리플레이 욕구 없으면 1회용
- 자세한 내용: `game_feel_design.md` 참조

---

## 🏗️ 1. 확장성 및 유연성

### ✅ PlayerRoot 계층 구조 — 캐릭터 교체 대비 필수

```
PlayerRoot
 ├─ PlayerCore (로직)
 ├─ CharacterController
 ├─ CameraPivot          ← 높이 기준 분리
 └─ VisualRoot           ← 아바타는 여기에만
      └─ Character
           ├─ Animator (Humanoid 필수)
           ├─ Hand_R_Socket
           └─ WeaponMount
```

- `VisualRoot` 분리 안 하면 캐릭터 교체 시 전체 재작업
- "Visual은 갈아끼운다, Core는 절대 안 건드린다" — GPT

### ✅ 데이터 분리 — ScriptableObject
```csharp
// ❌ 절대 금지
public float damage = 25f;

// ✅ 올바른 방법
[CreateAssetMenu]
public class WeaponData : ScriptableObject {
    public float damage;
    public int maxAmmo;
    public float reloadTime;
}
```

### ✅ 런타임 수치 변경 — IStatModifier
```csharp
public interface IStatModifier {
    void Apply(PlayerStats stats);
    void Remove(PlayerStats stats);
    string ModifierName { get; }
}

// ❌ 절대 금지
player.moveSpeed += 10f;

// ✅ 올바른 방법
new SpeedBoostModifier().Apply(playerStats);
```

### ✅ 캐릭터 추상화 — ICharacter
```csharp
public interface ICharacter {
    void PlayHit();
    void PlayShoot();
    void PlayReload();
    void PlayDance(int index);
    float CameraPivotHeight { get; }
    Vector3 HandROffset { get; }
}

// ❌ 절대 금지 — 캐릭터 하드코딩
if (characterName == "chocolat") { ... }

// ✅ 올바른 방법
_character.PlayDance(danceIndex);
```

### ✅ 인터페이스 중심 설계
```csharp
public interface IInteractable {
    void Interact();
    string GetPromptText();
}

public interface IDamageable {
    void TakeDamage(float amount);
    bool IsDead { get; }
}
```

### ✅ 씬 구조 — 초반 확정
```
TitleScene / GameScene / ResultScene
```

### ✅ 네임스페이스
- 프로젝트 초반부터 구조 확정, 파일 많아지면 충돌 발생

---

## 🌐 2. 네트워크 프로젝트 — Core / Network 강제 분리

```csharp
// ✅ 순수 로직 — NetworkBehaviour 절대 몰라야 함
public class PlayerCore {
    public Vector3 CalculateMove(Vector2 input, Transform t) { ... }
    public bool TryGetShootTarget(Ray ray, float range, out RaycastHit hit) { ... }
}

// ✅ 네트워크 래퍼 — 전달자 역할만
public class PlayerNetwork : NetworkBehaviour {
    private PlayerCore _core = new PlayerCore();
    void Update() {
        if (!IsOwner) return;
        _core.CalculateMove(GetInput(), transform);
        SubmitMoveServerRpc(GetInput());
    }
}
```

**위반 검증:**
- `PlayerCore`에 `NetworkBehaviour`·`IsOwner`·`RPC` → 구조 위반
- `PlayerNetwork`에 게임 로직 직접 실행 → 구조 위반

---

## 🚀 3. 최적화

### ✅ 오브젝트 풀링
```csharp
private ObjectPool<GameObject> _pool;
void Awake() {
    _pool = new ObjectPool<GameObject>(
        createFunc:      () => Instantiate(prefab),
        actionOnGet:     obj => obj.SetActive(true),
        actionOnRelease: obj => obj.SetActive(false)
    );
}
```

### ✅ 캐싱 — Update() 내 GetComponent 절대 금지
```csharp
// ❌ 절대 금지
void Update() { GetComponent<Rigidbody>().AddForce(...); }

// ✅ 올바른 방법
private Rigidbody _rb;
void Awake() { _rb = GetComponent<Rigidbody>(); }
```

### ✅ 텍스처 압축 / LOD / Profiler 습관화

---

## 🎯 4. 타이머 긴장감 연출 기준

| 단계 | 조건 | 연출 |
|------|------|------|
| 1 | 50% 이하 | HUD 색 변화 (흰→주황) |
| 2 | 30% 이하 | 심장박동 사운드 |
| 3 | 10% 이하 | 화면 가장자리 흔들림 + 숨소리 + 빨간 비네트 |

---

## 📦 5. 에셋 관리 — Resources 폴더 절대 금지

```
❌ 절대 금지: Resources 폴더
✅ 허용: Addressables 또는 Serialized Reference
```

---

## 🎮 6. 입력 시스템 — 초반 확정

| 방식 | 장점 | 단점 |
|------|------|------|
| `Input.GetKey()` 구형 | 즉시 사용 | 리맵핑 어려움 |
| `Input System` 패키지 | 유연·멀티플랫폼 | 초기 설정 복잡 |

- **혼용 절대 금지**

---

## 🔊 7. 오디오 구조

```csharp
public class AudioManager : MonoBehaviour {
    public static AudioManager Instance;
    public AudioSource bgmSource;
    public AudioSource sfxSource;

    void Awake() {
        if (Instance == null) { Instance = this; DontDestroyOnLoad(gameObject); }
        else Destroy(gameObject);
    }

    public void PlaySFX(AudioClip clip) => sfxSource.PlayOneShot(clip);
    public void PlayBGM(AudioClip clip) { bgmSource.clip = clip; bgmSource.Play(); }
}
```

---

## 📦 8. 콘텐츠 추가가 쉬운 구조

| 매니저 | 역할 |
|--------|------|
| `GameManager` | 게임 상태 관리 |
| `UIManager` | HUD·팝업·씬 전환 |
| `AudioManager` | BGM/SFX |
| `EnemyManager` | 스폰·웨이브 |

```csharp
void Start()     { PlayerHealth.OnPlayerDead += ShowGameOverUI; }
void OnDestroy() { PlayerHealth.OnPlayerDead -= ShowGameOverUI; }
```

---

## 📝 9. 코드 품질 규칙

| 규칙 | 내용 |
|------|------|
| **Fun First** | Tier 0~1 없으면 Tier 2~3 손대지 마라 |
| **타임박스** | 30분 고민·1시간 넘으면 구현 |
| **PlayerRoot 분리** | VisualRoot 없으면 캐릭터 교체 시 지옥 |
| **ICharacter 필수** | 캐릭터 이름 하드코딩 절대 금지 |
| **Core/Network 분리** | 순수 로직에 네트워크 코드 금지 |
| 매직 넘버 금지 | const 또는 ScriptableObject |
| IStatModifier | 런타임 수치 변경은 인터페이스로 |
| Resources 금지 | Addressables 또는 Serialized Reference |
| Update() 캐싱 | GetComponent는 Awake/Start에서만 |
| 이벤트 해제 | OnDestroy에서 반드시 `-=` |
| 타이머 3단계 | 50%·30%·10% 각각 다른 연출 |

---

## ✅ 개발 단계별 체크리스트

### 🚩 시작 전
- [ ] Tier 0 목표 확인 (이동·사격·HitStop)
- [ ] PlayerRoot 계층 구조 잡았는가?
- [ ] 씬 구조 확정
- [ ] Input System 방식 결정
- [ ] URP 확인
- [ ] 네임스페이스 확정
- [ ] Resources 폴더 미사용 확인

### 🔧 구현 중
- [ ] Tier 0~1 완료 전에 Tier 2~3 손대지 않았는가?
- [ ] VisualRoot 하위에만 아바타 배치했는가?
- [ ] ICharacter로 캐릭터 행동 추상화했는가?
- [ ] PlayerCore에 네트워크 코드 없는가?
- [ ] "이거 없으면 재미 죽냐?" 판단했는가?
- [ ] Update() 내 GetComponent 없는가?
- [ ] Resources.Load 없는가?

### 🎨 폴리싱
- [ ] HitStop 있는가? (Tier 0 핵심)
- [ ] 생존 루프 작동하는가? (Tier 1 핵심)
- [ ] 욕심 트리거 연출 있는가? (game_feel_design.md)
- [ ] 타이머 3단계 연출 있는가?
- [ ] 캐릭터 교체 테스트 1회 실시했는가? (Track B 진입 전)
- [ ] 이벤트 구독 해제(-=) 확인

---

> 📌 살아있는 문서(Living Document)로 관리한다.

---

## 📅 변경 이력
| 날짜 | 내용 | 작성자 |
|------|------|--------|
| 2025-04-23 | 초안 작성 | 엘릭서 & Claude & Gemini |
| 2025-04-23 | 공통 가이드라인 정제 | Claude |
| 2025-04-23 | Resources 금지·IStatModifier·로직/네트워크 분리 | GPT 1차 |
| 2025-04-23 | 타임박스·재미 루프·Core/Network·타이머 3단계 | GPT 2차 |
| 2025-04-23 | Fun First·Tier 구조·수업용 맥락 반영 | 전체 팀 합의 |
| 2025-04-23 | PlayerRoot 계층 구조·ICharacter·캐릭터 교체 체크리스트 추가 | GPT 3차 피드백 |
