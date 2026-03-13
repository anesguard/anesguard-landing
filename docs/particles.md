# Anesguard 랜딩페이지 — 히어로 파티클 효과 기술 명세

## 개요

히어로 섹션 배경에 사용할 커스텀 파티클 애니메이션.
라이브러리 의존성 없이 순수 Canvas API + requestAnimationFrame으로 구현한다.

**핵심 동작:**

- 파티클이 왼쪽 가장자리에서 생성 → 오른쪽으로 일정 속도로 흘러감
- 오른쪽 끝 근처에서 fade-out되며 자연스럽게 사라짐
- 화면 밖으로 나간 파티클은 제거, 왼쪽에서 새로 생성 (무한 루프)
- 마우스 인터랙션: 커서 근처 파티클이 밀려나고, 이탈 후 원래 흐름으로 복귀
- 파티클 간 거리가 가까우면 연결선 자동 생성
- 마우스 근처 파티클은 밝아지고, 커서↔파티클 연결선 표시

---

## 구조

```
┌─────────────────────────────────────────────┐
│  <div> (position: relative, overflow: hidden) │
│  ┌───────────────────────────────────────┐   │
│  │ <canvas> (position: absolute, z:0)    │   │
│  │  파티클 렌더링 레이어                   │   │
│  └───────────────────────────────────────┘   │
│  ┌───────────────────────────────────────┐   │
│  │ <div> (position: absolute, z:1)       │   │
│  │  텍스트 + CTA 버튼 (pointer-events:none)│   │
│  │  (버튼만 pointer-events: auto)         │   │
│  └───────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

텍스트 레이어에 `pointer-events: none`을 주고, CTA 버튼에만 `pointer-events: auto`를 줘야
마우스 이벤트가 캔버스까지 전달되면서도 버튼 클릭이 가능하다.

---

## 전체 코드

```javascript
// === Canvas 초기화 ===
var canvas = document.getElementById("pc");
var ctx = canvas.getContext("2d");
var W, H, dpr;
var mouse = { x: -9999, y: -9999, active: false };

// === 조절 가능한 상수 (아래 "옵션 설명" 참고) ===
var LINK_DIST = 120;
var MOUSE_RADIUS = 180;
var MOUSE_FORCE = 0.5;
var FRICTION = 0.95;
var BASE_SPEED = 0.4;
var MAX_PARTICLES = 80;
var SPAWN_RATE = 0.3;
var particles = [];

// === 리사이즈 처리 ===
function resize() {
  var rect = canvas.parentElement.getBoundingClientRect();
  dpr = window.devicePixelRatio || 1;
  W = rect.width;
  H = rect.height;
  canvas.width = W * dpr;
  canvas.height = H * dpr;
  ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
}

// === 파티클 생성 ===
function spawn() {
  var y = Math.random() * H;
  var speed = BASE_SPEED + Math.random() * 0.3;
  var r = 1.5 + Math.random() * 2;
  var alpha = 0.08 + Math.random() * 0.25;
  return {
    x: -10, // 왼쪽 화면 밖에서 시작
    y: y,
    vx: speed, // 오른쪽 방향 속도
    vy: (Math.random() - 0.5) * 0.15, // 약간의 수직 떨림
    r: r, // 반지름
    alpha: alpha, // 현재 투명도
    maxAlpha: alpha, // 최대 투명도 (개별 파티클마다 다름)
    speed: speed, // 기본 이동 속도 (복귀용 기준값)
    age: 0, // 생성 후 경과 프레임
    fadeIn: 60 + Math.random() * 40, // fade-in 완료까지 프레임 수
  };
}

// === 초기화 (화면에 파티클 미리 배치) ===
function init() {
  resize();
  particles.length = 0;
  for (var i = 0; i < MAX_PARTICLES; i++) {
    var p = spawn();
    p.x = Math.random() * W; // 초기엔 화면 전체에 랜덤 배치
    p.age = p.fadeIn + 1; // 이미 fade-in 완료 상태로
    particles.push(p);
  }
}

// === 마우스 이벤트 ===
canvas.parentElement.addEventListener("mousemove", function (e) {
  var rect = canvas.parentElement.getBoundingClientRect();
  mouse.x = e.clientX - rect.left;
  mouse.y = e.clientY - rect.top;
  mouse.active = true;
});
canvas.parentElement.addEventListener("mouseleave", function () {
  mouse.active = false;
});
window.addEventListener("resize", function () {
  resize();
});

// === 메인 루프 ===
function draw() {
  ctx.clearRect(0, 0, W, H);

  // 파티클 수가 부족하면 확률적으로 생성
  if (particles.length < MAX_PARTICLES && Math.random() < SPAWN_RATE) {
    particles.push(spawn());
  }

  // --- 물리 업데이트 ---
  for (var i = particles.length - 1; i >= 0; i--) {
    var p = particles[i];
    p.age++;

    // 마우스 밀어내기
    if (mouse.active) {
      var dx = p.x - mouse.x;
      var dy = p.y - mouse.y;
      var dist = Math.sqrt(dx * dx + dy * dy);
      if (dist < MOUSE_RADIUS && dist > 0) {
        var force = (1 - dist / MOUSE_RADIUS) * MOUSE_FORCE;
        p.vx += (dx / dist) * force;
        p.vy += (dy / dist) * force;
      }
    }

    // 기본 흐름 방향으로 복귀 (부드럽게)
    var targetVx = p.speed;
    p.vx += (targetVx - p.vx) * 0.02; // 0.02 = 복귀 속도
    p.vy *= FRICTION;

    p.x += p.vx;
    p.y += p.vy;

    // 오른쪽 밖으로 나가면 제거
    if (p.x > W + 30) {
      particles.splice(i, 1);
      continue;
    }

    // 투명도 계산 (fade-in + fade-out)
    var lifeFrac = 1;
    if (p.age < p.fadeIn) {
      lifeFrac = p.age / p.fadeIn; // fade-in
    }
    var fadeOut = W * 0.85; // 화면 85% 지점부터 fade-out 시작
    if (p.x > fadeOut) {
      lifeFrac *= 1 - (p.x - fadeOut) / (W - fadeOut + 30);
    }
    p.alpha = p.maxAlpha * Math.max(0, lifeFrac);
  }

  // --- 파티클 간 연결선 ---
  for (var i = 0; i < particles.length; i++) {
    for (var j = i + 1; j < particles.length; j++) {
      var a = particles[i],
        b = particles[j];
      var dx = a.x - b.x,
        dy = a.y - b.y;
      var dist = Math.sqrt(dx * dx + dy * dy);
      if (dist < LINK_DIST) {
        var opacity = (1 - dist / LINK_DIST) * Math.min(a.alpha, b.alpha) * 0.8;
        ctx.beginPath();
        ctx.moveTo(a.x, a.y);
        ctx.lineTo(b.x, b.y);
        ctx.strokeStyle = "rgba(147, 197, 253," + opacity + ")"; // 연결선 색상
        ctx.lineWidth = 0.7;
        ctx.stroke();
      }
    }
  }

  // --- 마우스↔파티클 연결선 ---
  if (mouse.active) {
    for (var i = 0; i < particles.length; i++) {
      var p = particles[i];
      var dx = p.x - mouse.x,
        dy = p.y - mouse.y;
      var dist = Math.sqrt(dx * dx + dy * dy);
      if (dist < MOUSE_RADIUS) {
        var opacity = (1 - dist / MOUSE_RADIUS) * 0.12;
        ctx.beginPath();
        ctx.moveTo(mouse.x, mouse.y);
        ctx.lineTo(p.x, p.y);
        ctx.strokeStyle = "rgba(59, 130, 246," + opacity + ")"; // 마우스 연결선 색상
        ctx.lineWidth = 0.5;
        ctx.stroke();
      }
    }
  }

  // --- 파티클 원 그리기 ---
  for (var i = 0; i < particles.length; i++) {
    var p = particles[i];
    var a = p.alpha;
    // 마우스 근처 파티클 밝게
    if (mouse.active) {
      var dx = p.x - mouse.x,
        dy = p.y - mouse.y;
      var dist = Math.sqrt(dx * dx + dy * dy);
      if (dist < MOUSE_RADIUS) {
        a = Math.min(a + (1 - dist / MOUSE_RADIUS) * 0.35, 0.65);
      }
    }
    ctx.beginPath();
    ctx.arc(p.x, p.y, p.r, 0, Math.PI * 2);
    ctx.fillStyle = "rgba(59, 130, 246," + a + ")"; // 파티클 색상
    ctx.fill();
  }

  requestAnimationFrame(draw);
}

init();
draw();
```

---

## 옵션 설명

### 파티클 기본

| 상수            | 기본값 | 설명                                | 조절 가이드                                             |
| --------------- | ------ | ----------------------------------- | ------------------------------------------------------- |
| `MAX_PARTICLES` | `80`   | 화면에 동시 존재하는 최대 파티클 수 | 60~100 권장. 높일수록 밀도↑, 성능↓                      |
| `SPAWN_RATE`    | `0.3`  | 프레임당 새 파티클 생성 확률 (0~1)  | 0.2=느리게 채워짐, 0.5=빠르게 채워짐                    |
| `BASE_SPEED`    | `0.4`  | 오른쪽 이동 기본 속도 (px/frame)    | 0.2=느긋, 0.6=빠른 흐름. 개별 파티클은 +0~0.3 랜덤 가산 |

### 파티클 외형

| 속성                | 기본값                    | 설명                                             |
| ------------------- | ------------------------- | ------------------------------------------------ |
| 반지름 (`r`)        | `1.5 ~ 3.5`               | `spawn()` 내 `1.5 + Math.random() * 2` 조절      |
| 투명도 (`maxAlpha`) | `0.08 ~ 0.33`             | `spawn()` 내 `0.08 + Math.random() * 0.25` 조절  |
| 색상                | `rgba(59, 130, 246, ...)` | `#3B82F6` (브라이트 블루). fillStyle 문자열 변경 |

### 연결선

| 상수        | 기본값                     | 설명                                     | 조절 가이드                                |
| ----------- | -------------------------- | ---------------------------------------- | ------------------------------------------ |
| `LINK_DIST` | `120`                      | 파티클 간 연결선이 생기는 최대 거리 (px) | 80=촘촘, 160=넓게 연결. 높일수록 선 많아짐 |
| 연결선 색상 | `rgba(147, 197, 253, ...)` | `#93C5FD` (하늘색). strokeStyle 변경     |
| 연결선 두께 | `0.7`                      | ctx.lineWidth 값. 0.5=가늘게, 1.0=굵게   |

### 마우스 인터랙션

| 상수                  | 기본값                    | 설명                                         | 조절 가이드                           |
| --------------------- | ------------------------- | -------------------------------------------- | ------------------------------------- |
| `MOUSE_RADIUS`        | `180`                     | 마우스 영향 반경 (px)                        | 120=작은 반응, 250=넓은 반응          |
| `MOUSE_FORCE`         | `0.5`                     | 밀어내는 힘의 세기                           | 0.3=부드럽게, 0.8=강하게 밀어냄       |
| `FRICTION`            | `0.95`                    | 속도 감쇠율 (매 프레임 곱셈)                 | 0.90=빠르게 멈춤, 0.98=오래 관성 유지 |
| 복귀 속도             | `0.02`                    | `draw()` 내 `(targetVx - p.vx) * 0.02`       | 0.01=천천히 복귀, 0.05=빠르게 복귀    |
| 마우스 연결선 색      | `rgba(59, 130, 246, ...)` | `#3B82F6`. 파티클 색과 동일 계열             |
| 마우스 근처 밝기 증가 | `+0.35`                   | `draw()` 내 `(1 - dist/MOUSE_RADIUS) * 0.35` |

### Fade-in / Fade-out

| 속성            | 기본값            | 설명                                                                  |
| --------------- | ----------------- | --------------------------------------------------------------------- |
| `fadeIn`        | `60 ~ 100 프레임` | 생성 후 투명→불투명 전환 시간. `60 + Math.random() * 40`              |
| fade-out 시작점 | `W * 0.85`        | 화면 가로 85% 지점부터 사라지기 시작. `0.75`=더 일찍, `0.90`=끝에서만 |

---

## 색상 변경 가이드

코드 내 4곳의 색상을 일관되게 변경해야 한다:

```
1. 파티클 원:        ctx.fillStyle = "rgba(59, 130, 246, ...)"     ← 메인 색상
2. 파티클 간 연결선:  ctx.strokeStyle = "rgba(147, 197, 253, ...)"  ← 메인 색상의 밝은 변형
3. 마우스 연결선:     ctx.strokeStyle = "rgba(59, 130, 246, ...)"   ← 메인 색상
4. (HTML) 텍스트/CTA 색상도 동일 팔레트로 맞출 것
```

현재 팔레트: `#3B82F6` (파티클) + `#93C5FD` (연결선) — Tailwind Blue 500/300 조합.

---

## 성능 참고

- `MAX_PARTICLES` 80 기준, 연결선 탐색이 O(n²) → 파티클 100 이상 시 체감 느려질 수 있음
- 모바일에서는 `MAX_PARTICLES`를 40~50으로 줄이고, `MOUSE_RADIUS` 대신 터치 이벤트 처리 필요
- Retina 대응: `devicePixelRatio` 기반 캔버스 스케일링 이미 포함
- `requestAnimationFrame` 사용으로 탭 비활성 시 자동 일시정지

---

## HTML 구조 예시

```html
<div
  style="position: relative; width: 100%; height: 100vh; background: #FAFAF8; overflow: hidden;"
>
  <!-- 파티클 캔버스 -->
  <canvas
    id="pc"
    style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;"
  ></canvas>

  <!-- 텍스트 오버레이 -->
  <div
    style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;
              display: flex; flex-direction: column; align-items: center; justify-content: center;
              z-index: 1; text-align: center; pointer-events: none;"
  >
    <!-- 콘텐츠 -->
    <h1>마취, AI가 미리 지킵니다.</h1>
    <p>서브카피</p>
    <div style="pointer-events: auto;">
      <!-- 버튼은 클릭 가능하게 -->
      <button>도입 문의하기</button>
    </div>
  </div>
</div>

<script>
  // 위 전체 코드 삽입
</script>
```

# 결정 상수

// 파티클 설정값
var LINK_DIST = 120;
var MOUSE_RADIUS = 230;
var MOUSE_FORCE = 0.1;
var FRICTION = 0.87;
var BASE_SPEED = 0.4;
var MAX_PARTICLES = 200;
var SPAWN_RATE = 0.95;

// spawn() 내 외형 파라미터
// r = 2 + Math.random() _ 2.5
// alpha = 0.08 + Math.random() _ 0.25
// fadeIn = 60 + Math.random() \* 40

// 색상
// 파티클: #3b82f6
// 연결선: #93c5fd (lineWidth: 1)
// 마우스 연결선: #3b82f6 (alpha: 0.12, lineWidth: 0.5)

// 마우스 인터랙션
// 복귀 속도: 0.02
// 근처 밝기 증가: 0.5

// Fade-out 시작점: W \* 0.85

// 배경색: #fafaf8
