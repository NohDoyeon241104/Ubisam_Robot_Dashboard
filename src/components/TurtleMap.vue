<!-- 터틀봇 시뮬레이션 Map 표시 컴포넌트 VUE 생성 -->
<script setup>
import { ref, onMounted } from 'vue'
import { Ros, Topic } from 'roslib'

const canvas = ref(null)
const posX = ref(0)
const posY = ref(0)
const angle = ref(0)
const trail = []  // 이동 궤적 저장

// turtlesim 좌표계: x,y 범위 0~11 → canvas 500px로 변환
const CANVAS_SIZE = 500
const SCALE = CANVAS_SIZE / 11

function draw(ctx) {
  // 배경 초기화
  ctx.fillStyle = '#3a5fcd'
  ctx.fillRect(0, 0, CANVAS_SIZE, CANVAS_SIZE)

  // 궤적 그리기
  trail.forEach((p, i) => {
    ctx.beginPath()
    ctx.arc(p.x, p.y, 2, 0, Math.PI * 2)
    ctx.fillStyle = `rgba(255, 255, 100, ${0.3 + (i / trail.length) * 0.7})`
    ctx.fill()
  })

  if (trail.length === 0) return

  // 현재 위치
  const cur = trail[trail.length - 1]

  // 화살표 (방향 표시)
  const len = 18
  const theta = angle.value  // radian
  const ex = cur.x + Math.cos(theta) * len
  const ey = cur.y - Math.sin(theta) * len  // canvas는 y축 반전

  // 화살표 몸통
  ctx.beginPath()
  ctx.moveTo(cur.x, cur.y)
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
    ey + headLen * Math.sin(theta - headAngle)
  )
  ctx.lineTo(
    ex - headLen * Math.cos(theta + headAngle),
    ey + headLen * Math.sin(theta + headAngle)
  )
  ctx.closePath()
  ctx.fillStyle = '#ff4444'
  ctx.fill()

  // 현재 위치 원
  ctx.beginPath()
  ctx.arc(cur.x, cur.y, 6, 0, Math.PI * 2)
  ctx.fillStyle = '#ffffff'
  ctx.fill()
}

onMounted(() => {
  const ctx = canvas.value.getContext('2d')

  // 초기 배경
  ctx.fillStyle = '#3a5fcd'
  ctx.fillRect(0, 0, CANVAS_SIZE, CANVAS_SIZE)

  const ros = new Ros({ url: 'ws://localhost:9090' })

  const poseTopic = new Topic({
    ros,
    name: '/turtle1/pose',
    messageType: 'turtlesim/msg/Pose'
  })

  poseTopic.subscribe((msg) => {
    // turtlesim y축 반전 (ROS는 아래→위, canvas는 위→아래)
    const cx = msg.x * SCALE
    const cy = CANVAS_SIZE - msg.y * SCALE

    posX.value = msg.x.toFixed(2)
    posY.value = msg.y.toFixed(2)
    angle.value = msg.theta

    trail.push({ x: cx, y: cy })
    if (trail.length > 500) trail.shift()  // 최대 500개 유지

    draw(ctx)
  })
})
</script>

<template>
  <div style="padding: 20px;">
    <h2>🐢 터틀 위치 추적</h2>
    <p>X: {{ posX }} &nbsp;|&nbsp; Y: {{ posY }} &nbsp;|&nbsp; 방향: {{ (angle * 180 / Math.PI).toFixed(1) }}°</p>
    <canvas
      ref="canvas"
      :width="500"
      :height="500"
      style="border: 2px solid #42b883; border-radius: 8px;"
    ></canvas>
  </div>

</template>

<style scoped>  
</style>