# 🍫 쇼콜라의 칼퇴 대작전 — 개발 로드맵 TODO
> 장르: TPS 숄더뷰 스텔스+런앤건  
> 엔진: Unity 6 (6000.3.13f1, URP)  
> 네트워크: NGO Host Mode (수업용은 로컬만)  
> 개발 방식: 바이브 코딩 + AI 협업 (Claude · Gemini · GPT)  
> 최종 결정권자: 엘릭서 / AI는 전부 참고 의견  
> 공통 계약서: shared_contract.md 필수  
> 감성 설계: game_feel_design.md 참조  
> 상세 기획: game_design_document.md 참조  
> 핵심 원칙: **"먼저 재밌게, 나중에 있어 보이게 (Fun First, Shine Later)"**

---

## 🗺️ 2트랙 전략

```
Track A (수업 제출용)          Track B (출시용)
─────────────────────          ─────────────────
구조 단순 (로컬만)              NGO + 전체 시스템
핵심 재미 루프만                모든 Tier 완성
"와 이거 게임이네" 수준         출시 가능 품질
          ↓                           ↑
     Track A 완성                살 붙이기
          ↓                      (점진적 확장)
     살 붙이기 → 괜찮으면 제출
          ↓
       출시작
```

> ⚠️ PlayerRoot 계층 구조와 PlayerCore 분리는 처음부터 필수  
> ⚠️ VisualRoot 분리해야 나중에 캐릭터 교체 시 코드 재작성 없음

---

## 🎮 개발 우선순위 — Tier 구조

> 뭔가 추가하고 싶을 때: **"이거 없으면 재미가 죽냐?"**  
> YES → 바로 구현 / NO → 뒤 Tier로 미루기

| Tier | 내용 | 기준 |
|------|------|------|
| **0** | 이동 · 사격 · HitStop | 없으면 게임 아님 |
| **1** | 적 추격 · 피격 · 타이머 · 탈출 판정 | 제출 가능 상태 |
| **2** | LineRenderer · 사운드 · UI | 무게감 시작 |
| **3** | ScriptableObject · 카메라 연출 | 교수 감탄용 |

> ⚠️ 제출 직전까지 Tier 0~1이 완벽하지 않으면 Tier 2~3은 과감히 포기한다

---

## 🎯 게임 컨셉 요약

| 항목 | 내용 |
|------|------|
| 주인공 | 쇼콜라 (VRC 아바타 · 수업용은 Capsule) |
| 목표 | 사원증 + 키카드 파밍 → 게이트 탈출 |
| 장애물 | 야간 근무자(빌런) |
| 제한 | 타이머 압박 |
| 핵심 재미 | 스텔스 긴장감 → 발각 시 전투 전환 → 욕심 트리거 |
| 뷰 | 숄더뷰 (3인칭) |

---

## ⌨️ 조작

| 키 | 기능 |
|----|------|
| W A S D | 이동 |
| Mouse Left | 사격 |
| Mouse Right | 조준 (ADS) |
| R | 재장전 |
| F | 상호작용 |
| 1 2 3 | 무기 교체 |
| G | 투척물 |
| C | 앉기 (스텔스 이동) |
| H | 댄스 3종 |

---

## 🔥 Tier 0 — 반응성 【오늘 목표】

> 완료 기준: "쏘면 맞고, 맞으면 멈춘다"  
> 여기서 손맛 확인되면 70% 성공

- [ ] **프로젝트 세팅**
  - [ ] Unity 6 (6000.3.13f1) URP 프로젝트 생성 ✅ (완료)
  - [ ] GameScene에 Plane + Capsule 배치
  - [ ] Resources 폴더 생성 금지

- [ ] **PlayerRoot 계층 구조 세팅** ← 처음부터 필수
  - [ ] `PlayerRoot` 빈 오브젝트 생성
  - [ ] `CharacterController` 부착
  - [ ] `CameraPivot` 빈 오브젝트 (어깨 높이)
  - [ ] `VisualRoot` 빈 오브젝트 (아바타는 여기에만)
  - [ ] Capsule을 VisualRoot 하위에 배치

- [ ] **PlayerCore.cs** (순수 로직 — 네트워크 코드 없음)
  - [ ] WASD 이동 벡터 계산 (`CalculateMove`)
  - [ ] 중력 계산 (`ApplyGravity`)
  - [ ] Raycast 사격 타겟 계산 (`TryGetShootTarget`)

- [ ] **PlayerController.cs** (수업용 MonoBehaviour 래퍼)
  - [ ] PlayerCore 호출
  - [ ] CharacterController.Move 실행
  - [ ] 마우스 클릭 → TryGetShootTarget

- [ ] **HitStop ★ (최우선)**
  - [ ] 적중 시 `Time.timeScale = 0.05f` → 0.08초 후 복구
  - [ ] 이것만 있어도 타격감 70% 완성

- [ ] **숄더뷰 카메라**
  - [ ] Cinemachine 3rd Person Follow
  - [ ] CameraPivot 기준으로 설정
  - [ ] 마우스 상하 → 피벗 / 마우스 좌우 → 캐릭터

---

## ⚡ Tier 1 — 생존 루프 【Tier 0 확인 후】

> 완료 기준: "적한테 쫓기며 탈출구로 뛰는 긴장감"  
> 여기까지 = 수업 제출 가능 상태

- [ ] **적 AI 1종 (추격형)**
  - [ ] NavMesh 베이크
  - [ ] 플레이어 추격
  - [ ] 근접 시 공격 (체력 감소)
  - [ ] 피격 → 사망 처리

- [ ] **체력 시스템**
  - [ ] IDamageable 인터페이스 연결
  - [ ] 플레이어 체력 감소 → 사망 판정

- [ ] **아이템 기초**
  - [ ] IInteractable 인터페이스
  - [ ] F키 사원증(큐브) 줍기
  - [ ] F키 키카드(큐브) 줍기

- [ ] **타이머 + 승패 판정**
  - [ ] 카운트다운 타이머
  - [ ] 시간 초과 → 패배
  - [ ] 사원증 + 키카드 보유 → 게이트 F키 → 승리

- [ ] **최소 HUD**
  - [ ] 타이머 표시
  - [ ] 아이템 보유 여부 표시

---

## 🎭 Tier 2 — 무게감 【시간 남을 때만】

- [ ] LineRenderer 탄환 궤적 (찰나만 표시)
- [ ] 총소리 SFX 1개
- [ ] 피격 이펙트 (Spark 파티클)
- [ ] 대미지 숫자 팝업
- [ ] 카메라 미세 반동(Recoil)

---

## 🧠 Tier 3 — 교수 감탄용 【여유 있을 때만】

- [ ] WeaponData ScriptableObject 1개
- [ ] 착지 카메라 출렁임 (Cinemachine Noise)
- [ ] 오브젝트 풀링 총알

---

## 🚩 Track B 확장 — 출시용 【Track A 완성 후 점진적】

### 프로토타입 2차 (게임다운 느낌)
- [ ] PlayerRoot 계층 → 쇼콜라 실제 아바타 이식
- [ ] ICharacter 인터페이스 구현 (PlayHit·PlayShoot·PlayReload·PlayDance)
- [ ] Animation Rigging + IK 손/총 위치 보정
- [ ] **교체 테스트 1회 실시** ← 반드시 이 단계에서 실행
- [ ] WeaponData · EnemyData · ItemData ScriptableObject
- [ ] 히트 피드백 5종
- [ ] IStatModifier 런 내 아이템 강화
- [ ] 욕심 트리거 MVP
- [ ] 스텔스 시스템 기초 (시야각·발소리 감지)
- [ ] 야근 스택 시스템 기초
- [ ] 빌런 AI 5종
- [ ] AudioManager
- [ ] NGO Host Mode 전환

### 프로토타입 3차 (비주얼)
- [ ] Addressables 본격 전환
- [ ] 사무실 텍스처·조명·Post Processing
- [ ] HUD 디자인 완성
- [ ] 선택 기반 리스크 이벤트 UI

### 프로토타입 4차 (폴리싱)
- [ ] 애니메이션 Blend Tree·상체 레이어
- [ ] 스텔스 → 라우드 전환 연출
- [ ] 야근 스택 보스 해금 구조
- [ ] 욕심 트리거 완성 (game_feel_design.md)
- [ ] 실패 코믹 연출 완성
- [ ] 퍼포먼스 기반 해금 (노킬·라우드·속도)
- [ ] 최적화 (Profiler·LOD·이벤트 해제)

### 오픈 단계
- [ ] **2차**: 멀티플레이 (UGS Relay + Dedicated Server)
- [ ] **3차**: DLC (캐릭터·맵·무기 패키지)
- [ ] **4차**: 퇴사 전쟁 모드 (익스트렉션)
- [ ] **5차**: 유저 모드 (Steam Workshop)

---

## 💡 개발 순서 (이대로 가라)

```
1단계 (오늘)
  PlayerRoot 구조 세팅 → 이동 → 사격 → HitStop
  → 테스트: "손맛 있나?"

2단계
  적 1마리 → 추격 → 피격

3단계
  타이머 → 탈출 판정
  → 여기까지 = 제출 가능

4단계 (시간 남으면)
  LineRenderer → 사운드 → 카메라

5단계 (여유 있으면)
  ScriptableObject 1개 "언급용"
```

---

## 📅 변경 이력
| 날짜 | 내용 | 작성자 |
|------|------|--------|
| 2025-04-23 | 초안 작성 | 엘릭서 & Claude & Gemini |
| 2025-04-23 | 4단계 세분화, 타격감·런강화 추가 | Claude |
| 2025-04-23 | NGO 오프라인 퍼스트, 장기 플랜 | Claude |
| 2025-04-23 | Addressables 시점 조정, NGO 기준 | Gemini |
| 2025-04-23 | 로직/네트워크 분리, 히트 피드백, IStatModifier, 빌런 5종 | GPT 1차 |
| 2025-04-23 | PlayerCore/Network, 욕심 트리거 MVP | GPT 2차 |
| 2025-04-23 | 2트랙 전략, Tier 구조, Fun First | 전체 팀 합의 |
| 2025-04-23 | PlayerRoot 계층 구조, ICharacter, 스텔스 시스템, 야근 스택, 퇴사 전쟁 모드, 캐릭터 교체 전략 추가 | GPT 3차 + 기획 확정 |
