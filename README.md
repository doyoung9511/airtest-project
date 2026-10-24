# Airtest 게임 자동 테스트 프로그램

Python + AirtestIDE를 활용한 **NIKKE** 게임 반복 테스트 자동화 스크립트입니다.



https://github.com/user-attachments/assets/bd004373-4d5c-4d3d-ba9d-9175c644a066



---

## 프로젝트 배경

QA 테스트 이후 빌드가 나온 시점에 **기본 기능 테스트**나 **리그레션 테스트**를 진행하면서, 반복적인 체크리스트를 수동으로 수행하는 과정에서 자동화의 필요성을 느꼈습니다.
실제 QA 업무에 적용하는 것을 목표로 Airtest를 이용해 구현했습니다.

---

## 사용 기술

| 기술 | 용도 |
| --- | --- |
| Python | 스크립트 작성 |
| AirtestIDE | 이미지 매칭 기반 자동화 프레임워크 |
| Claude Code | AI 코드 어시스턴트 활용 |

---

## 자동화 콘텐츠

| 함수 | 콘텐츠 | 비고 |
| --- | --- | --- |
| `run_enter()` | 게임 진입 | **필수** — 항상 먼저 실행 |
| `run_mission()` | 퀘스트 보상 수령 | |
| `run_friend()` | 친구 | |
| `run_outpost()` | 전초기지 방어 | |
| `run_shop()` | 상점 (무료 갱신·구매) | |
| `run_dispatch()` | 전초기지 파견 | |
| `run_interception()` | 요격전 | |
| `run_simulation()` | 시뮬레이션 룸 | |

---

## 실행 방법

1. [AirtestIDE](https://airtest.netease.com/)를 설치합니다.
2. Android 기기 또는 에뮬레이터를 연결하고 AirtestIDE의 **Devices** 창에서 `connect` 합니다.
3. 이 저장소를 클론한 뒤 AirtestIDE에서 `untitled.air` 폴더를 엽니다.
   ```bash
   git clone https://github.com/platypus9511/airtest-project.git
   ```
4. `untitled.py`의 `run()` 함수에서 실행할 콘텐츠의 주석(`#`)을 해제합니다.
   ```python
   def run():
       run_enter()        # 로비 진입 — 필수, 주석 처리 금지
       run_mission()
       # run_shop()       # 실행하지 않을 콘텐츠는 주석 처리
   ```
5. 실행 버튼을 누릅니다.

> **이미지 템플릿(`.png`)은 `.gitignore`로 저장소에서 제외되어 있습니다.**
> 직접 실행하려면 AirtestIDE에서 각 버튼을 다시 캡처해 템플릿을 만들어야 합니다. 해상도·기기에 따라 매칭 결과가 달라질 수 있습니다.

---

## 프로젝트 구조

```
airtest-project/
├── untitled.air/
│   └── untitled.py   # 메인 자동화 스크립트
├── .gitignore        # 이미지 템플릿(*.png), 로그 등 제외
└── README.md
```

---

## 코드 구조 설계

### 1. 모듈화 설계

처음에는 하나의 `run()` 함수에 모든 코드를 작성했지만, **유지보수와 선택적 실행**을 위해 콘텐츠별 함수로 분리했습니다.

```python
def run():
    run_enter()        # 게임 진입 (필수)
    run_mission()      # 퀘스트 보상
    run_friend()       # 친구
    run_outpost()      # 전초기지 방어
    run_shop()         # 상점
    run_dispatch()     # 전초기지 파견
    run_simulation()   # 시뮬레이션 룸
    run_interception() # 요격전
```

### 2. 공통 함수 설계

반복되는 동작은 공통 함수로 만들어 코드 중복을 줄였습니다.

```python
# 버튼이 있으면 터치, 없으면 그냥 넘어가기
def close_if_exists(template):
    target = exists(template)
    if target:
        touch(target)
        sleep(1.0)
        return True
    return False

# 여러 팝업이 연속으로 뜨는 경우 처리
def close_all_popups(*templates, max_attempts=10):
    for _ in range(max_attempts):
        closed_any = False
        for t in templates:
            if close_if_exists(t):
                closed_any = True
        if not closed_any:
            break

# 버튼이 나타날 때까지 대기 후 터치
def wait_and_touch(template, timeout=5.5):
    try:
        target = wait(template, timeout=timeout)
        touch(target)
        sleep(0.5)
        return True
    except Exception as e:
        print(f"[WARN] wait_and_touch 실패: {e}")
        return False
```

### 3. 조건부 분기 처리

콘텐츠마다 상황에 따라 다른 동작이 필요해 콘텐츠별로 분기를 다르게 설계했습니다.

**상점 갱신** — 무료 갱신일 때만 진행

```python
# 0원 갱신 이미지를 먼저 체크
free_refresh = exists(Template(r"free_refresh.png"))
if free_refresh:
    wait_and_touch(CONFIRM_BTN)   # 무료 → 확인
    purchase_flow(...)
else:
    wait_and_touch(CANCEL_BTN)    # 유료 → 취소
```

**요격전** — 도전 가능 여부 → 빠른 전투 가능 여부 순으로 판단

```python
# 도전하기 활성화 여부 먼저 체크
challenge_active = exists(Template(r"challenge_active.png", threshold=0.9))
if not challenge_active:
    wait_and_touch(HOME_BTN)
    return

# 주간 빠른 전투 활성화 여부 체크
weekly_quick_active = exists(Template(r"weekly_quick.png"))
if weekly_quick_active:
    for _ in range(3):
        btn = exists(Template(r"weekly_quick.png", threshold=0.9))
        if btn:
            touch(btn)
        else:
            break
else:
    # 전투 진입 후 최대 4분 대기
    wait_and_touch(battle_enter_btn)
    start = time.time()
    while time.time() - start < 240:
        if exists(battle_end_img):
            touch(battle_end_img)
            break
        sleep(3.5)
```

### 4. 반복 작업 처리

횟수가 매번 다른 작업은 **버튼 존재 여부**로 반복 횟수를 결정했습니다.

```python
# 퀘스트 모두받기 — 버튼이 있는 동안만 반복 (최대 5번)
for _ in range(5):
    btn = exists(모두받기_버튼)
    if btn:
        touch(btn)
        sleep(2.0)
        close_if_exists(REWARD_BTN)
        sleep(2.0)
    else:
        break
```

---

## 트러블슈팅

### 1. 이미지 오감지

- **증상**: 버튼이 비활성화 상태인데 활성화 상태로 인식해 잘못된 동작을 수행함
- **원인**: 활성화/비활성화 버튼이 색상만 다르고 형태가 동일함
- **해결**
  1. 활성화 이미지를 먼저 체크하는 방식으로 변경
  2. `threshold` 값을 0.7 → 0.9로 높여 더 엄격하게 매칭
  3. 재발 시에도 "활성화 이미지가 없으면 비활성화로 판단"하는 방식으로 통일

```python
exists(Template(r"button.png", threshold=0.9))

# 파견 수령 버튼 예시
active_receive = exists(Template(r"active_btn.png"))
if active_receive:
    touch(active_receive)    # 활성화 → 진행
else:
    close_if_exists(X_BTN)   # 비활성화 → 닫기
```

### 2. 화면 전환 타이밍 문제

- **증상**: 화면 전환 중 버튼을 찾지 못함
- **원인**: 전환 애니메이션이 끝나기 전에 버튼을 찾음
- **해결**: 화면 전환 후 대기 시간을 늘림

```python
close_if_exists(X_BTN)
sleep(2.0)  # 화면 전환 대기
wait_and_touch(HOME_BTN)
```

### 3. 연속 팝업 처리

- **증상**: 팝업이 연속으로 뜰 때 하나만 닫고 다음 동작으로 넘어감
- **원인**: 기존 `close_if_exists`는 팝업을 한 번만 처리함
- **해결**: 모든 팝업이 사라질 때까지 반복하는 `close_all_popups` 함수 설계

```python
def close_all_popups(*templates, max_attempts=10):
    for _ in range(max_attempts):
        closed_any = False
        for t in templates:
            if close_if_exists(t):
                closed_any = True
        if not closed_any:
            break  # 더 이상 팝업이 없으면 종료
```

### 4. 로그 분석으로 디버깅

AirtestIDE 로그의 **confidence 값**을 분석해 이미지 매칭 정확도를 판단하고, `threshold`를 조정하거나 이미지를 다시 캡처해 문제를 해결했습니다.

```text
# 정상 감지
[MSTemplatePre] confidence: 0.97 → 감지 성공

# 오감지
[MSTemplatePre] confidence: 0.89 → 비활성화인데 감지됨
```

---

## Author

**김도영 (Kim Do-young)** — Game QA
