<script setup>
import { ref } from 'vue'

const showModal = ref(false)

const licenses = [
  {
    name: 'roslib.js',
    repo: 'https://github.com/RobotWebTools/roslibjs',
    copyright: 'Copyright (c) 2012-2023 RobotWebTools',
    license: 'BSD 3-Clause License',
    note: 'ROS와 웹 브라우저 간 WebSocket 통신 라이브러리'
  },
  {
    name: 'm-explore-ros2 (explore_lite)',
    repo: 'https://github.com/robo-friends/m-explore-ros2',
    copyright: [
      'Copyright (c) 2008, Robert Bosch LLC',
      'Copyright (c) 2015-2016, Jiri Horner',
      'Copyright (c) 2021, Carlos Alvarez, Juan Galvis'
    ],
    license: 'BSD 3-Clause License',
    note: 'ROS2 Frontier 기반 자율 탐색 패키지. Dashing/Foxy/Galactic/Eloquent 기준 테스트됨.'
  },
  {
    name: 'Vue.js',
    repo: 'https://github.com/vuejs/core',
    copyright: 'Copyright (c) 2018-present, Yuxi (Evan) You and Vue contributors',
    license: 'MIT License',
    note: '프론트엔드 UI 프레임워크'
  },
  {
    name: 'Chart.js',
    repo: 'https://github.com/chartjs/Chart.js',
    copyright: 'Copyright (c) 2014-2022 Chart.js Contributors',
    license: 'MIT License',
    note: '센서 데이터 차트 시각화'
  },
]
</script>

<template>
  <!-- ! 버튼 -->
  <button class="license-btn" @click="showModal = true" title="오픈소스 라이선스 고지">
    !
  </button>

  <!-- 팝업 모달 -->
  <Teleport to="body">
    <div v-if="showModal" class="modal-overlay" @click.self="showModal = false">
      <div class="modal">
        <div class="modal-header">
          <span class="modal-title">📄 오픈소스 라이선스 고지</span>
          <button class="close-btn" @click="showModal = false">✕</button>
        </div>

        <div class="modal-body">
          <p class="modal-desc">
            이 소프트웨어는 아래 오픈소스 라이브러리를 사용합니다.<br>
            각 라이브러리의 라이선스 조건에 따라 저작권을 고지합니다.
          </p>

          <div v-for="lib in licenses" :key="lib.name" class="license-item">
            <div class="lib-header">
              <span class="lib-name">{{ lib.name }}</span>
              <span class="lib-license">{{ lib.license }}</span>
            </div>
            <p class="lib-note">{{ lib.note }}</p>
            <div class="lib-copyright">
              <template v-if="Array.isArray(lib.copyright)">
                <div v-for="c in lib.copyright" :key="c">{{ c }}</div>
              </template>
              <template v-else>{{ lib.copyright }}</template>
            </div>
            <a :href="lib.repo" target="_blank" class="lib-repo">{{ lib.repo }}</a>
          </div>
        </div>

        <div class="modal-footer">
          <button class="close-footer-btn" @click="showModal = false">닫기</button>
        </div>
      </div>
    </div>
  </Teleport>
</template>

<style scoped>
.license-btn {
  width: 24px;
  height: 24px;
  border-radius: 50%;
  background: #21262d;
  color: #aaa;
  border: 1px solid #444;
  font-weight: bold;
  font-size: 13px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.15s;
  flex-shrink: 0;
}
.license-btn:hover {
  background: #f0a500;
  color: #000;
  border-color: #f0a500;
}

/* 모달 오버레이 */
.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.7);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 9999;
}

.modal {
  background: #161b22;
  border: 1px solid #30363d;
  border-radius: 12px;
  width: 600px;
  max-width: 92vw;
  max-height: 80vh;
  display: flex;
  flex-direction: column;
}

.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 16px 20px;
  border-bottom: 1px solid #30363d;
}

.modal-title {
  font-size: 15px;
  font-weight: bold;
  color: #e0e0e0;
}

.close-btn {
  background: none;
  border: none;
  color: #888;
  font-size: 16px;
  cursor: pointer;
  padding: 4px 8px;
  border-radius: 4px;
}
.close-btn:hover { background: #21262d; color: #e0e0e0; }

.modal-body {
  padding: 20px;
  overflow-y: auto;
  flex: 1;
}

.modal-desc {
  font-size: 12px;
  color: #888;
  margin-bottom: 20px;
  line-height: 1.7;
}

.license-item {
  background: #0d1117;
  border: 1px solid #30363d;
  border-radius: 8px;
  padding: 14px 16px;
  margin-bottom: 12px;
}

.lib-header {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 6px;
}

.lib-name {
  font-size: 13px;
  font-weight: bold;
  color: #42b883;
}

.lib-license {
  font-size: 11px;
  background: #21262d;
  color: #aaa;
  padding: 2px 8px;
  border-radius: 4px;
  border: 1px solid #30363d;
}

.lib-note {
  font-size: 11px;
  color: #666;
  margin-bottom: 8px;
  line-height: 1.6;
}

.lib-copyright {
  font-size: 11px;
  color: #aaa;
  font-family: 'Consolas', monospace;
  margin-bottom: 6px;
  line-height: 1.8;
}

.lib-repo {
  font-size: 11px;
  color: #58a6ff;
  text-decoration: none;
  font-family: 'Consolas', monospace;
}
.lib-repo:hover { text-decoration: underline; }

.modal-footer {
  padding: 12px 20px;
  border-top: 1px solid #30363d;
  display: flex;
  justify-content: flex-end;
}

.close-footer-btn {
  background: #21262d;
  color: #e0e0e0;
  border: 1px solid #30363d;
  border-radius: 6px;
  padding: 6px 20px;
  cursor: pointer;
  font-size: 13px;
}
.close-footer-btn:hover { background: #42b883; color: #000; border-color: #42b883; }
</style>
