<!-- 센서차트 _chart.js vue-chartjs 설치 후 만드는 컴포넌트 파일 -->
 <script setup>
//  ROS2 웹소켓 연결 및 데이터 수신을 위한 코드
import {ref, onMounted} from 'vue'
import { Ros, Topic } from 'roslib';
// 차트 관련 라이브러리 임포트
import { Line } from 'vue-chartjs'
import {
  Chart as ChartJS,
  LineElement,
  PointElement,
  LinearScale,
  CategoryScale,
  Title,
  Tooltip,
  Legend
} from 'chart.js'

ChartJS.register(LineElement, PointElement, LinearScale, CategoryScale, Title, Tooltip, Legend)
// 차트 데이터와 옵션을 저장할 반응형 변수
const chartData = ref({
    labels: [], // 시간 또는 인덱스
    datasets: [
    {
        label: 'ROS2 메시지 수신 카운트',
        data: [],
        borderColor: '#42b883',
        backgroundColor: 'rgba(66,184,131,0.1)',
        tension: 0.4,
        fill: true,
    }]
})
// 차트 옵션 설정
const chartOptions={
    // 차트가 반응형으로 동작하도록 설정
    responsive: true,
    // 애니메이션 효과를 끄고 실시간 업데이트에 최적화
    animation:false,
    // 차트의 다양한 플러그인 설정
    plugins:{
        // 범례를 차트 상단에 표시하도록 설정
        legend:{position: 'top'},
        // 차트 제목 설정
        title : {
            // 차트 제목을 표시하도록 설정
            display: true,
            // 차트 제목 텍스트 설정
            text: 'ROS2 실시간 데이터'
        },
        // 툴팁 설정
        scales:{
            // y축 설정
            y:{
                // Y축의 최소값을 0으로 설정하여 그래프가 항상 0에서 시작하도록 함
                beginAtZero: true 
            }
        }
    }
}
// ROS2 메시지 수신 카운트를 저장할 변수
let count = 0
// 컴포넌트가 마운트될 때 ROS2 웹소켓에 연결하고 토픽을 구독하여 데이터를 수신하는 로직
onMounted(() => {
  const ros = new Ros({ url: 'ws://localhost:9090' })

  const listener = new Topic({
    ros,
    name: '/chatter',
    messageType: 'std_msgs/msg/String'
  })

  listener.subscribe(() => {
    const now = new Date().toLocaleTimeString()
    count++

    chartData.value = {
      labels: [...chartData.value.labels, now].slice(-20),
      datasets: [{
        ...chartData.value.datasets[0],
        data: [...chartData.value.datasets[0].data, count].slice(-20)
      }]
    }
  })
})
 </script>

 <template>
   <div style="padding: 20px;">
     <h2>센서 데이터 차트</h2>
    <Line :data="chartData" :options="chartOptions" />
    </div>   
</template> 

<style scoped>
</style>