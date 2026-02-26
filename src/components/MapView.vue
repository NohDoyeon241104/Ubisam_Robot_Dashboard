<!-- SLAM OccupancyGrid 맵 표시 컴포넌트 -->
<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'
import { Ros, Topic, Message } from 'roslib'

// ── 캔버스 ref ──────────────────────────────
const canvas = ref(null)

// ── 연결 상태 ────────────────────────────────
const rosStatus = ref('연결 중...')

// ── 맵 데이터 ────────────────────────────────
const mapInfo = ref(null)   // { resolution, width, height, origin }
const mapData = ref([])     // OccupancyGrid 1차원 배열

// ── 로봇 위치 ────────────────────────────────
const robotX   = ref(0)
const robotY   = ref(0)
const robotYaw = ref(0)

// ── Nav Goal ─────────────────────────────────
const goalX = ref(null)
const goalY = ref(null)

// ── 줌 ───────────────────────────────────────
const zoom = ref(1)

// ── 경로(/plan) ──────────────────────────────
const navPath = ref([])  // [{ x, y }, ...]

// ── ROS 인스턴스 (cleanup용) ─────────────────
let ros = null
let mapListener   = null
let odomListener  = null
let planListener  = null
let goalPublisher = null

// ────────────────────────────────────────────
// [질문 3] 좌표 변환 유틸
// ────────────────────────────────────────────

// 실제 좌표(m) → 격자 인덱스
function worldToGrid(wx, wy) {
  if (!mapInfo.value) return null
  const { resolution, origin, width, height } = mapInfo.value
  const gx = Math.floor((wx - origin.position.x) / resolution)
  const gy = Math.floor((wy - origin.position.y) / resolution)
  if (gx < 0 || gx >= width || gy < 0 || gy >= height) return null
  return { gx, gy }
}

// 격자 인덱스 → 1차원 배열 인덱스 (index = gy * width + gx)
function gridToIndex(gx, gy) {
  if (!mapInfo.value) return -1
  return gy * mapInfo.value.width + gx
}

// 실제 좌표(m) → 캔버스 픽셀 (Y축 반전 포함)
function worldToCanvas(wx, wy) {
  if (!mapInfo.value) return null
  const { resolution, origin, height } = mapInfo.value
  const px = (wx - origin.position.x) / resolution
  const py = height - (wy - origin.position.y) / resolution  // Y반전
  return { px, py }
}

// 캔버스 픽셀 → 실제 좌표(m) — 클릭 시 Nav Goal 계산용
function canvasToWorld(px, py) {
  if (!mapInfo.value) return null
  const { resolution, origin, width, height } = mapInfo.value
  const wx = px * resolution + origin.position.x
  const wy = (height - py) * resolution + origin.position.y  // Y반전 복원
  return { wx, wy }
}

// ────────────────────────────────────────────
// 현재 로봇의 격자 인덱스 (computed — 화면 표시용)
// ────────────────────────────────────────────
const currentGrid = computed(() => worldToGrid(robotX.value, robotY.value))
const currentIndex = computed(() => {
  if (!currentGrid.value) return -1
  return gridToIndex(currentGrid.value.gx, currentGrid.value.gy)
})

// ────────────────────────────────────────────
// 캔버스 렌더링
// ────────────────────────────────────────────
function drawMap() {
  const canvasEl = canvas.value
  if (!canvasEl || !mapInfo.value || mapData.value.length === 0) return

  const ctx             = canvasEl.getContext('2d')
  const { width, height } = mapInfo.value
  const data            = mapData.value

  canvasEl.width  = width
  canvasEl.height = height

  // ── 1. OccupancyGrid → ImageData (Y축 반전 포함) ──
  const imageData = ctx.createImageData(width, height)

  for (let i = 0; i < data.length; i++) {
    const value = data[i]
    const ix = i % width
    const iy = Math.floor(i / width)
    // Y반전: ROS 좌하단 origin → Canvas 좌상단 origin
    const canvasIdx = ((height - 1 - iy) * width + ix) * 4

    let r, g, b
    if (value === -1) {
      r = 180; g = 180; b = 180   // 미탐색 — 회색
    } else if (value === 0) {
      r = 255; g = 255; b = 255   // 빈 공간 — 흰색
    } else if (value >= 100) {
      r = 30;  g = 30;  b = 30    // 장애물 — 검정
    } else {
      const c = 255 - Math.floor((value / 100) * 225)
      r = c; g = c; b = c         // 1~99 확률 장애물 — 회색 보간
    }

    imageData.data[canvasIdx]     = r
    imageData.data[canvasIdx + 1] = g
    imageData.data[canvasIdx + 2] = b
    imageData.data[canvasIdx + 3] = 255
  }
  ctx.putImageData(imageData, 0, 0)

  // ── 2. 경로(/plan) 오버레이 ──
  if (navPath.value.length > 1) {
    ctx.beginPath()
    ctx.strokeStyle = '#00BFFF'
    ctx.lineWidth = 1.5
    navPath.value.forEach((pt, idx) => {
      const cp = worldToCanvas(pt.x, pt.y)
      if (!cp) return
      idx === 0 ? ctx.moveTo(cp.px, cp.py) : ctx.lineTo(cp.px, cp.py)
    })
    ctx.stroke()
  }

  // ── 3. Nav Goal 마커 ──
  if (goalX.value !== null) {
    const gcp = worldToCanvas(goalX.value, goalY.value)
    if (gcp) {
      ctx.font = '16px Arial'
      ctx.fillText('🎯', gcp.px - 8, gcp.py + 7)
    }
  }

  // ── 4. 로봇 마커 (TurtleMap.vue 화살표 스타일 참고) ──
  const rcp = worldToCanvas(robotX.value, robotY.value)
  if (rcp) {
    const { px, py } = rcp
    const len = 14
    const theta = -robotYaw.value  // Y반전으로 인한 방향 반전
    const ex = px + Math.cos(theta) * len
    const ey = py + Math.sin(theta) * len

    // 화살표 몸통
    ctx.beginPath()
    ctx.moveTo(px, py)
    ctx.lineTo(ex, ey)
    ctx.strokeStyle = '#ff4444'
    ctx.lineWidth = 3
    ctx.stroke()

    // 화살표 머리
    const headLen = 8
    const headAngle = Math.PI / 6
    ctx.beginPath()
    ctx.moveTo(ex, ey)
    ctx.lineTo(
      ex - headLen * Math.cos(theta - headAngle),
      ey - headLen * Math.sin(theta - headAngle)
    )
    ctx.lineTo(
      ex - headLen * Math.cos(theta + headAngle),
      ey - headLen * Math.sin(theta + headAngle)
    )
    ctx.closePath()
    ctx.fillStyle = '#ff4444'
    ctx.fill()

    // 현재 위치 원
    ctx.beginPath()
    ctx.arc(px, py, 6, 0, Math.PI * 2)
    ctx.fillStyle = '#ffffff'
    ctx.fill()
  }
}

// ────────────────────────────────────────────
// 캔버스 클릭 → Nav Goal 발행
// ────────────────────────────────────────────
function onCanvasClick(e) {
  if (!mapInfo.value || !goalPublisher) return

  const canvasEl = canvas.value
  const rect = canvasEl.getBoundingClientRect()
  // zoom 보정
  const px = (e.clientX - rect.left)  / zoom.value
  const py = (e.clientY - rect.top)   / zoom.value

  const world = canvasToWorld(px, py)
  if (!world) return

  goalX.value = world.wx
  goalY.value = world.wy

  goalPublisher.publish(new Message({
    header: { frame_id: 'map' },
    pose: {
      position: { x: world.wx, y: world.wy, z: 0 },
      orientation: { x: 0, y: 0, z: 0, w: 1 }
    }
  }))

  drawMap()
}

// 마우스 휠 줌
function onWheel(e) {
  e.preventDefault()
  const delta = e.deltaY > 0 ? -0.1 : 0.1
  zoom.value = Math.min(Math.max(zoom.value + delta, 0.5), 4)
}

// ────────────────────────────────────────────
// ROS 연결 (TurtleMap.vue와 동일한 패턴)
// ────────────────────────────────────────────
onMounted(() => {
  ros = new Ros({ url: 'ws://localhost:9090' })

  ros.on('connection', () => { rosStatus.value = '연결됨' })
  ros.on('error',      () => { rosStatus.value = '오류' })
  ros.on('close',      () => { rosStatus.value = '연결 끊김' })

  // /map
  mapListener = new Topic({
    ros,
    name: '/map',
    messageType: 'nav_msgs/OccupancyGrid',
    throttle_rate: 500
  })
  mapListener.subscribe((msg) => {
    mapInfo.value = msg.info
    mapData.value = msg.data
    drawMap()
  })

  // /odom — 로봇 위치 + 쿼터니언 → Yaw 변환
  odomListener = new Topic({
    ros,
    name: '/odom',
    messageType: 'nav_msgs/Odometry',
    throttle_rate: 100
  })
  odomListener.subscribe((msg) => {
    const pos = msg.pose.pose.position
    const ori = msg.pose.pose.orientation
    robotX.value = pos.x
    robotY.value = pos.y
    // 쿼터니언 → Yaw (라디안)
    robotYaw.value = Math.atan2(
      2 * (ori.w * ori.z + ori.x * ori.y),
      1 - 2 * (ori.y * ori.y + ori.z * ori.z)
    )
    drawMap()
  })

  // /plan — Nav2 전역 경로
  planListener = new Topic({
    ros,
    name: '/plan',
    messageType: 'nav_msgs/Path',
    throttle_rate: 300
  })
  planListener.subscribe((msg) => {
    navPath.value = msg.poses.map((p) => ({
      x: p.pose.position.x,
      y: p.pose.position.y
    }))
    drawMap()
  })

  // /goal_pose 발행용
  goalPublisher = new Topic({
    ros,
    name: '/goal_pose',
    messageType: 'geometry_msgs/PoseStamped'
  })
})

onUnmounted(() => {
  mapListener?.unsubscribe()
  odomListener?.unsubscribe()
  planListener?.unsubscribe()
  ros?.close()
})
</script>

<template>
  <div style="padding: 20px;">
    <h2>🗺️ SLAM 맵 뷰어</h2>

    <!-- 상태 & 좌표 (TurtleMap.vue와 동일한 스타일) -->
    <p>
      ROS: {{ rosStatus }} &nbsp;|&nbsp;
      X: {{ robotX.toFixed(2) }} &nbsp;|&nbsp;
      Y: {{ robotY.toFixed(2) }} &nbsp;|&nbsp;
      방향: {{ (robotYaw * 180 / Math.PI).toFixed(1) }}°
    </p>

    <!-- 질문 3 좌표 변환 결과 -->
    <p v-if="mapInfo" style="font-size: 12px; color: #888; margin-top: 4px; font-family: monospace;">
      격자 인덱스: <span style="color: #42b883;">gx={{ currentGrid?.gx ?? '-' }}, gy={{ currentGrid?.gy ?? '-' }}</span>
      &nbsp;→&nbsp;
      1D index: <span style="color: #42b883;">{{ currentIndex }}</span>
      &nbsp;|&nbsp;
      맵 크기: {{ mapInfo.width }} × {{ mapInfo.height }} cells
      ({{ (mapInfo.width  * mapInfo.resolution).toFixed(1) }}m
       × {{ (mapInfo.height * mapInfo.resolution).toFixed(1) }}m)
    </p>
    <p v-else style="font-size: 12px; color: #555; margin-top: 4px;">
      🛰️ /map 토픽 대기 중... (SLAM을 먼저 실행해주세요)
    </p>

    <!-- 줌 컨트롤 -->
    <div style="display: flex; align-items: center; gap: 8px; margin: 8px 0;">
      <button class="ctrl-btn" @click="zoom = Math.min(zoom + 0.25, 4)">🔍+</button>
      <span style="font-size: 12px; color: #aaa; min-width: 42px; text-align: center;">
        {{ (zoom * 100).toFixed(0) }}%
      </span>
      <button class="ctrl-btn" @click="zoom = Math.max(zoom - 0.25, 0.5)">🔍-</button>
      <button class="ctrl-btn" @click="zoom = 1">리셋</button>
      <span style="font-size: 11px; color: #555; margin-left: 6px;">
        💡 맵 클릭 → Nav Goal 발행 &nbsp;|&nbsp; 휠로 줌
      </span>
    </div>

    <!-- 범례 -->
    <div style="display: flex; gap: 14px; font-size: 11px; color: #888; margin-bottom: 8px; flex-wrap: wrap;">
      <span><i class="legend-cell" style="background:#ffffff;" /> 빈 공간</span>
      <span><i class="legend-cell" style="background:#1e1e1e;" /> 장애물</span>
      <span><i class="legend-cell" style="background:#b4b4b4;" /> 미탐색</span>
      <span>🤖 로봇 위치</span>
      <span>🎯 Nav Goal</span>
      <span style="color:#00BFFF;">━ 경로(/plan)</span>
    </div>

    <!-- 캔버스 (줌 적용) -->
    <div class="canvas-wrapper">
      <canvas
        ref="canvas"
        :style="{
          transform: `scale(${zoom})`,
          transformOrigin: 'top left',
          border: '2px solid #42b883',
          borderRadius: '8px',
          cursor: mapInfo ? 'crosshair' : 'default',
          imageRendering: 'pixelated'
        }"
        @click="onCanvasClick"
        @wheel.prevent="onWheel"
      />
      <div v-if="!mapInfo" class="placeholder">
        <p>🛰️ SLAM 실행 후<br>맵이 여기 표시됩니다</p>
      </div>
    </div>

    <!-- Nav Goal 발행 결과 -->
    <p v-if="goalX !== null"
       style="font-size: 12px; color: #f0a500; margin-top: 8px; font-family: monospace;">
      🎯 Nav Goal 발행됨 → ({{ goalX.toFixed(2) }}, {{ goalY.toFixed(2) }})m
    </p>
  </div>
</template>

<style scoped>
.ctrl-btn {
  background: #21262d;
  color: #e0e0e0;
  border: 1px solid #333;
  border-radius: 6px;
  padding: 3px 10px;
  cursor: pointer;
  font-size: 12px;
}
.ctrl-btn:hover { background: #2d333b; }

.canvas-wrapper {
  position: relative;
  overflow: auto;
  max-width: 100%;
  max-height: 600px;
  background: #111;
  border-radius: 8px;
}

.placeholder {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #555;
  text-align: center;
  font-size: 14px;
  line-height: 2;
  min-height: 300px;
}

.legend-cell {
  display: inline-block;
  width: 10px;
  height: 10px;
  border-radius: 2px;
  border: 1px solid #444;
  margin-right: 3px;
  vertical-align: middle;
}
</style>
