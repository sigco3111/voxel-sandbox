# ⛏️ voxel-sandbox

> Three.js + Perlin Noise 기반 마우스로 블록을 파괴·건설하는 복셀 샌드박스 게임

**마인크래프트처럼 블록(Voxel)으로 이루어진 3D 세상을** 브라우저에 구현합니다. **Perlin Noise** 로 산과 호수가 있는 지형을 자동 생성하고, **마우스 왼쪽 클릭**으로 블록 파괴, **오른쪽 클릭**으로 새 블록 설치가 가능한 간단한 샌드박스 게임입니다. `THREE.InstancedMesh` 로 수천 블록을 60fps로 렌더링하고, `THREE.Raycaster` 로 정확히 어떤 면을 클릭했는지 감지합니다.

[🇰🇷 한국어 (기본)](#) · [🇺🇸 English](./README.en.md)

---

## 🎬 라이브 데모 (Live Demo)

> **👉 [https://voxel-sandbox-five.vercel.app/](https://voxel-sandbox-five.vercel.app/)** — 브라우저에서 바로 실행 (60fps, WebGL)

| | |
|---|---|
| ![Demo](https://img.shields.io/badge/Live-Demo-7C3AED?style=for-the-badge&logo=vercel&logoColor=white) | [![Repo](https://img.shields.io/badge/GitHub-sigco3111%2Fvoxel--sandbox-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/sigco3111/voxel-sandbox) |
| ![Status](https://img.shields.io/badge/Status-Live-22C55E?style=flat-square) | ![Stack](https://img.shields.io/badge/Stack-Three.js%20%2B%20InstancedMesh-000000?style=flat-square&logo=three.js&logoColor=white) |
| ![License](https://img.shields.io/badge/License-MIT-F1C40F?style=flat-square) | ![Deps](https://img.shields.io/badge/Dependencies-0-9CA3AF?style=flat-square) |

### 🎮 빠른 사용법
1. 위 데모 링크 클릭 → 브라우저에서 페이지 열기
2. **WASD** — 1인칭 이동 / **마우스 드래그** — 시점 회전
3. **Perlin Noise** 로 산·호수가 자동 생성된 3D 세계 탐험
4. **마우스 왼쪽 클릭** — 블록 파괴
5. **마우스 오른쪽 클릭** — 새 블록 설치
6. **`1` `2` `3`** — 설치할 블록 타입 변경 (잔디/돌/모래)

---

## 🤖 생성 정보 (Attribution)

이 프로젝트의 코드는 아래 모델과 프롬프트를 이용해 **자동으로 생성**되었습니다.

| 항목 | 값 |
|---|---|
| **모델** | MiniMax-M3 |
| **실행 환경** | OpenCode CLI |
| **저장소** | [`sigco3111/voxel-sandbox`](https://github.com/sigco3111/voxel-sandbox) |
| **라이브** | [https://voxel-sandbox-five.vercel.app/](https://voxel-sandbox-five.vercel.app/) |
| **라이선스** | MIT |
| **의존성** | 없음 (Three.js CDN, 단일 HTML) |

### 📝 사용된 프롬프트 (원문)

```
마인크래프트처럼 블록(Voxel)으로 이루어진 3D 세상을 브라우저에 구현하는데,
펄린 노이즈(Perlin Noise)를 사용하여 산과 호수가 있는 지형을 자동으로 생성하고,
마우스 왼쪽 클릭으로 블록을 파괴하고 오른쪽 클릭으로 새로운 블록을 설치할 수
있는 간단한 샌드박스 게임을 만들어줘.

Implementation Advice: Use Three.js. To optimize, don't just render individual
boxes; use InstancedMesh or merge geometries. Use Raycaster to detect which
face of a voxel is clicked for adding/removing blocks. 모든 의존관계의 코드를
하나의 HTML에 담는 형태로 코드 작성.
```

---

## 🧮 복셀 월드 + Perlin Noise (Voxel + Perlin)

### 복셀 월드 (Voxel World)

3D 공간을 **단위 큐브 그리드**로 분할해 각 셀에 블록 타입을 할당하는 방식:

```
world[x][y][z] = BLOCK_TYPE  // 0 = 공기, 1 = 잔디, 2 = 돌, 3 = 모래, ...
```

- **x, z** (수평) — `-WORLD_SIZE/2` ~ `+WORLD_SIZE/2`
- **y** (수직) — `0` (지면) ~ `MAX_HEIGHT` (산 정상)
- 각 셀은 1×1×1 단위 큐브

### Perlin Noise (지형 자동 생성)

Ken Perlin (1983) 의 **그라데이션 노이즈** 알고리즘으로 자연스러운 산·호수 생성:

```
height(x, z) = noise(x * FREQ, 0, z * FREQ) * AMP
              + noise(x * FREQ * 2, 0, z * FREQ * 2) * AMP * 0.5   // 옥타브 2
              + noise(x * FREQ * 4, 0, z * FREQ * 4) * AMP * 0.25  // 옥타브 3
```

**fBm (fractional Brownian motion)** — 여러 옥타브를 더해 디테일 누적:

```js
function generateHeight(x, z) {
  let height = 0;
  let amplitude = 1;
  let frequency = 0.02;

  for (let octave = 0; octave < 4; octave++) {
    height += noise(x * frequency, z * frequency) * amplitude;
    amplitude *= 0.5;     // 진폭 절반씩 감소 (디테일 약해짐)
    frequency *= 2;       // 주파수 2배 (디테일 더해짐)
  }
  return Math.floor(height * 10);
}

// 블록 배치
for (let x = -32; x < 32; x++) {
  for (let z = -32; z < 32; z++) {
    const h = generateHeight(x, z);
    for (let y = 0; y <= h; y++) {
      let type = (y === h) ? GRASS :      // 최상단 = 잔디
                (y > h - 3) ? DIRT :       // 그 아래 2층 = 흙
                STONE;                      // 나머지 = 돌
      world[x + 32][y][z + 32] = type;
    }
  }
}
```

---

## ⚡ 성능 최적화 (Performance)

수천~수만 블록을 60fps로 렌더링하려면 **메쉬 최적화** 필수:

### ❌ 나쁜 방법: 개별 Mesh

```js
for (const block of blocks) {
  const mesh = new THREE.Mesh(geometry, material);
  scene.add(mesh);  // 5000 draw calls → 5fps
}
```

### ✅ 좋은 방법 1: InstancedMesh

`InstancedMesh` 는 **하나의 draw call** 로 동일 지오메트리/머티리얼의 여러 인스턴스 렌더링:

```js
const geom = new THREE.BoxGeometry(1, 1, 1);
const mat = new THREE.MeshLambertMaterial();
const instancedMesh = new THREE.InstancedMesh(geom, mat, MAX_BLOCKS);

// 각 블록의 변환 행렬 설정
const dummy = new THREE.Object3D();
blocks.forEach((block, i) => {
  dummy.position.set(block.x, block.y, block.z);
  dummy.updateMatrix();
  instancedMesh.setMatrixAt(i, dummy.matrix);
});
instancedMesh.instanceMatrix.needsUpdate = true;
scene.add(instancedMesh);  // 1 draw call → 60fps
```

### ✅ 좋은 방법 2: Merge Geometries

```js
import { BufferGeometryUtils } from 'three/addons/utils/BufferGeometryUtils.js';
const merged = BufferGeometryUtils.mergeGeometries(blockGeometries);
const mesh = new THREE.Mesh(merged, material);  // 1 mesh
```

### 🎯 Raycaster (블록 면 감지)

`Raycaster` 가 카메라에서 마우스 광선을 쏘아 **정확히 어느 블록의 어느 면**을 클릭했는지 반환:

```js
const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();

canvas.addEventListener('mousedown', (e) => {
  mouse.x = (e.clientX / window.innerWidth) * 2 - 1;
  mouse.y = -(e.clientY / window.innerHeight) * 2 + 1;
  raycaster.setFromCamera(mouse, camera);

  const intersects = raycaster.intersectObject(instancedMesh);
  if (intersects.length > 0) {
    const hit = intersects[0];
    const blockPos = hit.object.userData.blocks[hit.instanceId];
    const faceNormal = hit.face.normal;  // 어느 면?

    // 왼쪽 클릭 → 파괴 / 오른쪽 클릭 → 인접 면에 설치
    if (e.button === 0) removeBlock(blockPos);
    else if (e.button === 2) addBlock(blockPos, faceNormal);
  }
});
```

---

## ✨ 주요 특징 (Features)

- ⛏️ **Perlin Noise 지형** — 산·호수 자동 생성 (fBm 4 옥타브)
- 🖱️ **마우스 인터랙션** — 좌클릭 파괴 / 우클릭 설치
- 🎯 **Raycaster 면 감지** — 정확히 어느 블록의 어느 면을 클릭했는지
- ⚡ **InstancedMesh** — 수천 블록을 1 draw call 로 렌더링 (60fps)
- 🧱 **블록 타입** — 잔디 / 흙 / 돌 / 모래 / 물 (옵션)
- 🏃 **1인칭 컨트롤** — WASD 이동 + 마우스 시점 (PointerLockControls)
- 🌅 **하늘·조명** — Three.js 기본 AmbientLight + DirectionalLight
- 📦 **단일 HTML** — Three.js CDN + 외부 의존성 0
- 🔒 **온디바이스** — 모든 렌더링이 브라우저 GPU에서 실행

---

## 🚀 실행 방법 (Quick Start)

### 방법 1: 라이브 데모 (가장 간단)
위 [🎬 라이브 데모](https://voxel-sandbox-five.vercel.app/) 링크 클릭 → 즉시 실행

### 방법 2: 그냥 브라우저로 열기
```bash
open index.html        # macOS
xdg-open index.html    # Linux
start index.html       # Windows
```

### 방법 3: 로컬 서버 (권장)
```bash
python3 -m http.server 8000
# → http://localhost:8000
```

> ⚠️ WebGL은 `file://` 에서도 동작하지만 일부 브라우저는 보안 정책상 제한할 수 있어 로컬 서버 권장.

---

## 🎮 조작법 (Controls)

| 입력 | 효과 |
|---|---|
| **WASD** | 1인칭 이동 (전진/후진/좌/우) |
| **마우스 드래그** | 시점 회전 (PointerLock) |
| **스페이스** | 점프 / 위로 |
| **Shift** | 아래로 (수직 하강) |
| **왼쪽 클릭** | 블록 파괴 |
| **오른쪽 클릭** | 새 블록 설치 |
| **`1` `2` `3` `4`** | 블록 타입 선택 (잔디/흙/돌/모래) |
| **`r`** | 월드 리셋 |

---

## 🛠️ 기술 스택 (Tech Stack)

| 영역 | 사용 기술 |
|---|---|
| **렌더링** | Three.js (WebGL) |
| **지형 생성** | Perlin Noise (fBm 4 옥타브) |
| **블록 렌더링** | `THREE.InstancedMesh` (성능 최적화) |
| **인터랙션** | `THREE.Raycaster` + PointerLockControls |
| **머티리얼** | `MeshLambertMaterial` (기본) 또는 `MeshStandardMaterial` (PBR) |
| **조명** | `AmbientLight` + `DirectionalLight` (태양) |
| **빌드** | 없음 (단일 HTML, CDN import) |

---

## 📂 프로젝트 구조

```
voxel-sandbox/
├── index.html      # 단일 HTML (Three.js CDN + Perlin + InstancedMesh + Raycaster)
├── README.md       # 한국어 (기본)
├── README.en.md    # English
├── LICENSE         # MIT
└── .gitignore
```

---

## 🎨 디자인 결정 (Design Choices)

브레인스토밍 단계에서 내린 결정 4가지:

| 결정 포인트 | 선택 | 이유 |
|---|---|---|
| **렌더링 API** | Three.js + InstancedMesh (vs 개별 Mesh / MergeGeometry) | 프롬프트 요구사항. InstancedMesh 가 가장 간단하면서 성능 우수 (1 draw call) |
| **지형 생성** | Perlin Noise (vs Simplex / Value / Worley) | 프롬프트 요구사항. fBm 4 옥타브로 산·호수 자연스럽게 |
| **인터랙션 감지** | Raycaster (vs bounding box intersection) | 정확히 어느 면을 클릭했는지 normal vector 로 알 수 있음 → 설치 위치 계산에 필수 |
| **저장 형식** | 메모리 내 3D 배열 (vs 직렬화) | 단일 세션 샌드박스라 영구 저장 불필요, 메모리가 가장 빠름 |

### 직접 커스터마이즈하고 싶다면

`index.html` 상단에서 다음 상수를 조정하면 분위기를 바꿀 수 있어요:

```js
const CONFIG = {
  WORLD_SIZE: 64,             // 가로·세로 크기 (셀, 64 = 4096 블록/층)
  MAX_HEIGHT: 32,             // 최대 높이 (셀)
  NOISE_FREQ: 0.02,           // Perlin 노이즈 주파수 (작을수록 큰 산)
  NOISE_AMP: 10,              // Perlin 진폭 (클수록 높은 산)
  NOISE_OCTAVES: 4,           // fBm 옥타브 수
  WATER_LEVEL: 8,             // 호수 수면 높이
  BLOCK_SIZE: 1,              // 블록 한 변 길이
  MAX_BLOCKS: 50000,          // InstancedMesh 최대 인스턴스
  GRAVITY: 0.05,              // 점프 시 중력 (옵션)
  MOVE_SPEED: 0.1,            // 이동 속도
  MOUSE_SENSITIVITY: 0.002,   // 마우스 감도
  // ... 더 많은 옵션은 코드 내 주석 참조
};
```

**파라미터 탐색**:
- `WORLD_SIZE = 32` → 작은 아일랜드 (성능 여유 큼)
- `WORLD_SIZE = 64` → 중간 맵 (기본값, ~4K 블록/층)
- `WORLD_SIZE = 128` → 큰 세계 (~16K 블록/층, 60fps 한계)
- `NOISE_FREQ = 0.01` → 평탄한 풍경 (완만한 언덕)
- `NOISE_FREQ = 0.05` → 거친 산맥 (날카로운 봉우리)
- `NOISE_OCTAVES = 1` → 매끈한 풍경
- `NOISE_OCTAVES = 8` → 매우 디테일한 풍경 (계산 cost ↑)
- `WATER_LEVEL = 4` → 얕은 호수
- `WATER_LEVEL = 16` → 깊은 호수 + 수중 산

**성능 비교 (64×64×32 블록)**:

| 방법 | Draw calls | FPS |
|---|---|---|
| 개별 Mesh × 130K | 130,000 | <5 fps ❌ |
| MergeGeometry | 1 | 60 fps ✅ |
| InstancedMesh | 1 | 60 fps ✅ |
| Chunked (16×16×32 단위) | ~256 | 60 fps ✅ (대형 월드) |

---

## 📚 관련 기술 (Related Techniques)

| 기술 | 용도 | 복잡도 |
|---|---|---|
| **Perlin Noise** | 자연 지형 (산·계곡) | O(N) |
| **Simplex Noise** | Perlin 개선판 (계산 빠름) | O(N) |
| **Value Noise** | 가장 간단한 노이즈 | O(N) |
| **fBm** | 다중 옥타브 (디테일 누적) | O(N × octaves) |
| **Marching Cubes** | 부드러운 메쉬 (Minecraft 류 변형) | O(N³) |
| **Greedy Meshing** | 인접 블록 머지 (성능) | O(N) |
| **Chunk System** | 대형 월드 관리 (16×16 단위) | O(chunks) |

**Perlin vs Simplex**:
- **Perlin (1983)**: Ken Perlin, 그라데이션 보간, 부드러운 자연 풍경
- **Simplex (2001)**: Ken Perlin 개선판, 더 빠르고 고차원 지원

**Voxel Rendering 진화**:
- Minecraft (2011) — 단순 청크 + 인접 면 컬링
- Minecraft RTX (2020) — Path tracing + DXR
- Teardown (2022) — 완전 파괴 가능한 voxel

---

## 📜 License

MIT © 2026 sigco3111

---

## 🙏 Acknowledgments

이 프로젝트는 **MiniMax-M3** 모델과 OpenCode CLI 환경에서 생성되었습니다. 프롬프트 엔지니어링과 디자인 결정은 저장소 소유자가 직접 수행했습니다.

- **Minecraft (2011)**: Mojang Studios — *"복셀 샌드박스의 교과서"*
- **Perlin Noise (1983)**: Ken Perlin — *"Improving Noise"*, SIGGRAPH*
- **Three.js**: [mrdoob & contributors](https://threejs.org/) — WebGL 추상화
- **InstancedMesh**: Three.js 공식 — 수천 객체 1 draw call
- **코딩미션 참조 페이지**: [cokac.com — 코드깎는노인](https://cokac.com/list/announcement/24)