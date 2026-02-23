<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
import { Ros, Topic } from 'roslib'

const canvas = ref(null)
const posX = ref(0)
const posY = ref(0)
const angle = ref(0)
const trail = []

const CANVAS_SIZE = 500
const SCALE = CANVAS_SIZE / 11

// 조종 속도 설정
const LINEAR_SPEED = 2.0   // 전진/후진 속도
const ANGULAR_SPEED = 2.0  // 회전 속도

let cmdVelTopic = null  // publish용 토픽 (onMounted 후 사용)

// 키보드 입력 → cmd_vel 발행
function handleKeyDown(e) {
  if (!cmdVelTopic) return

  const keyMap = {
    ArrowUp:    { linear: { x: LINEAR_SPEED,  y: 0, z: 0 }, angular: { x: 0, y: 0, z: 0 } },
    ArrowDown:  { linear: { x: -LINEAR_SPEED, y: 0, z: 0 }, angular: { x: 0, y: 0, z: 0 } },
    ArrowLeft:  { linear: { x: 0, y: 0, z: 0 }, angular: { x: 0, y: 0, z: ANGULAR_SPEED } },
    ArrowRight: { linear: { x: 0, y: 0, z: 0 }, angular: { x: 0, y: 0, z: -ANGULAR_SPEED } },
  }

  const twist = keyMap[e.key]
  if (!twist) return

  e.preventDefault()  // 브라우저 스크롤 방지

  cmdVelTopic.publish(twist)
}

// 키를 떼면 정지
function handleKeyUp(e) {
  if (!cmdVelTopic) return
  if (!['ArrowUp','ArrowDown','ArrowLeft','ArrowRight'].includes(e.key)) return

  cmdVelTopic.publish({
    linear:  { x: 0, y: 0, z: 0 },
    angular: { x: 0, y: 0, z: 0 }
  })
}

function draw(ctx) {
  ctx.fillStyle = '#3a5fcd'
  ctx.fillRect(0, 0, CANVAS_SIZE, CANVAS_SIZE)

  trail.forEach((p, i) => {
    ctx.beginPath()
    ctx.arc(p.x, p.y, 2, 0, Math.PI * 2)
    ctx.fillStyle = `rgba(255, 255, 100, ${0.3 + (i / trail.length) * 0.7})`
    ctx.fill()
  })

  if (trail.length === 0) return

  const cur = trail[trail.length - 1]
  const len = 18
  const theta = angle.value
  const ex = cur.x + Math.cos(theta) * len
  const ey = cur.y - Math.sin(theta) * len

  ctx.beginPath()
  ctx.moveTo(cur.x, cur.y)
  ctx.lineTo(ex, ey)
  ctx.strokeStyle = '#ff4444'
  ctx.lineWidth = 3
  ctx.stroke()

  const headLen = 8
  const headAngle = Math.PI / 6
  ctx.beginPath()
  ctx.moveTo(ex, ey)
  ctx.lineTo(ex - headLen * Math.cos(theta - headAngle), ey + headLen * Math.sin(theta - headAngle))
  ctx.lineTo(ex - headLen * Math.cos(theta + headAngle), ey + headLen * Math.sin(theta + headAngle))
  ctx.closePath()
  ctx.fillStyle = '#ff4444'
  ctx.fill()

  ctx.beginPath()
  ctx.arc(cur.x, cur.y, 6, 0, Math.PI * 2)
  ctx.fillStyle = '#ffffff'
  ctx.fill()
}

onMounted(() => {
  const ctx = canvas.value.getContext('2d')
  ctx.fillStyle = '#3a5fcd'
  ctx.fillRect(0, 0, CANVAS_SIZE, CANVAS_SIZE)

  const ros = new Ros({ url: 'ws://localhost:9090' })

  // ① 위치 구독 (기존과 동일)
  const poseTopic = new Topic({
    ros,
    name: '/turtle1/pose',
    messageType: 'turtlesim/msg/Pose'
  })

  poseTopic.subscribe((msg) => {
    const cx = msg.x * SCALE
    const cy = CANVAS_SIZE - msg.y * SCALE

    posX.value = msg.x.toFixed(2)
    posY.value = msg.y.toFixed(2)
    angle.value = msg.theta

    trail.push({ x: cx, y: cy })
    if (trail.length > 500) trail.shift()

    draw(ctx)
  })

  // ② 속도 명령 발행용 토픽 (새로 추가)
  cmdVelTopic = new Topic({
    ros,
    name: '/turtle1/cmd_vel',
    messageType: 'geometry_msgs/msg/Twist'
  })

  // 키보드 이벤트 등록
  window.addEventListener('keydown', handleKeyDown)
  window.addEventListener('keyup', handleKeyUp)
})

// 컴포넌트 종료 시 이벤트 해제
onUnmounted(() => {
  window.removeEventListener('keydown', handleKeyDown)
  window.removeEventListener('keyup', handleKeyUp)
})
</script>

<template>
  <div style="padding: 20px;">
    <h2>🐢 터틀 위치 추적</h2>
    <p>X: {{ posX }} &nbsp;|&nbsp; Y: {{ posY }} &nbsp;|&nbsp; 방향: {{ (angle * 180 / Math.PI).toFixed(1) }}°</p>
    <p style="color: #42b883; font-size: 14px;">
      ⌨️ 브라우저에서 화살표 키로 직접 조종 가능합니다
    </p>
    <canvas
      ref="canvas"
      :width="500"
      :height="500"
      style="border: 2px solid #42b883; border-radius: 8px;"
    ></canvas>
  </div>
</template>