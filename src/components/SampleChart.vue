<template>
  <div class="chart-container" style="position: relative; height:700px; width:1400px; background-color: white;">
    <canvas id="chart"></canvas>
  </div>
  <br/>
  <div> </div>
  <br/>
  <br/>
  <button style="height:40px; width:100px" @click="preWeek()">&lt;&lt;前週</button>
  <button style="height:40px; width:100px" @click="nextWeek()">次週&gt;&gt;</button>
</template>

<script setup>

import { ref, onMounted } from 'vue';
import axios from 'axios';
import Chart from 'chart.js/auto';
import ChartDataLabels from 'chartjs-plugin-datalabels';
import dayjs from 'dayjs'

dayjs.locale("ja")

Chart.register(ChartDataLabels)

const chart = ref([])
let tideData = []
let timeData = []
let pc = '12'
let hc = '18'
let date = dayjs()
date = date.subtract((date.day() + 6) % 7, 'd')
let yr = date.get('year')
let mn = date.get('month') + 1
let dy = date.get('date')
let rg = 'week'
let myChart

const renderChart = () => {
  const edd = []
  const flood = []
  for (let day in chart.value) {
    tideData.push(...chart.value[day].tide.map(v => v.cm))
    tideData.pop()
    timeData.push(...chart.value[day].tide.map(v => v.time))
    timeData.pop()
    edd.push(chart.value[day].edd)
    flood.push(chart.value[day].flood)
  }
  let ctx = document.getElementById("chart");
  myChart = new Chart(ctx, {
    type: 'line',
    data: {
      labels: getLabels(),
      datasets: [{
        data: tideData,
        lineTension: 0.3,
        borderWidth: 8,
        borderColor: 'blue',
        pointStyle: false,
        datalabels: {
          labels: {
            value: {
              color: 'blue',
              textStrokeWidth: 15,
              textStrokeColor: 'white',
              align: function (ctx) {
                if ((tideData[ctx.dataIndex] >= tideData[ctx.dataIndex - 1] && tideData[ctx.dataIndex] > tideData[ctx.dataIndex + 1])) {
                  return 'end'
                } else {
                  return 'start'
                }
              },
              formatter: function (value, ctx) {
                if (
                  (tideData[ctx.dataIndex] >= tideData[ctx.dataIndex - 1] && tideData[ctx.dataIndex] > tideData[ctx.dataIndex + 1]) &&
                  flood[Math.floor(ctx.dataIndex / 72)]?.[0]
                ) {
                  return Math.round(value)
                } else if (
                  (tideData[ctx.dataIndex] <= tideData[ctx.dataIndex - 1] && tideData[ctx.dataIndex] < tideData[ctx.dataIndex + 1]) &&
                  edd[Math.floor(ctx.dataIndex / 72)]?.[0]
                ) {
                  return '\r\n' + Math.round(value)
                } else {
                  return null
                }

              },
            },
            time: {
              font: {
                family: 'serif',
                weight: 900,
              },
              color: 'blue',
              textStrokeWidth: 15,
              textStrokeColor: 'white',
              align: function (ctx) {
                if ((tideData[ctx.dataIndex] >= tideData[ctx.dataIndex - 1] && tideData[ctx.dataIndex] > tideData[ctx.dataIndex + 1])) {
                  return 'end'
                } else {
                  return 'start'
                }
              },
              formatter: function (value, ctx) {
                if ( (tideData[ctx.dataIndex] >= tideData[ctx.dataIndex - 1] && tideData[ctx.dataIndex] > tideData[ctx.dataIndex + 1]) &&
                  flood[Math.floor(ctx.dataIndex / 72)]?.[0]) {
                  return flood[Math.floor(ctx.dataIndex / 72)].shift()?.time + '\r\n'
                } else if ( (tideData[ctx.dataIndex] <= tideData[ctx.dataIndex - 1] && tideData[ctx.dataIndex] < tideData[ctx.dataIndex + 1]) &&
                  edd[Math.floor(ctx.dataIndex / 72)]?.[0]) {
                  return edd[Math.floor(ctx.dataIndex / 72)].shift()?.time + '\r\n'
                } else {
                  return null
                }

              },
            },
          }
        }
      }],
    },
    options: {
      animation: false,
      spanGaps: true,
      scales: {
        x: {
          ticks: {
            autoSkip: false,
            maxTicksLimit: 60,
            color: function (ctx) {
              if (ctx.tick.label == 0) {
                return 'white'
              }
              return 'blue'
            },
            font: {
              size: 18
            },
          },
          grid: {
            color: 'blue',
            lineWidth: function (context) {
              if (context.tick.label == '0') {
                return 3
              }
              return 1
            }
          },
        },
        y: {
          min: -50,
          max: 300,
          ticks: {
            color: 'blue',
            font: {
              size: 18
            },
          }
        }
      },
      plugins: {
        datalabels: {
          font: {
            size: 24,
            weight: 'bold'
          }
        },
        legend: {
          display: false
        },
        tooltip: {
          enabled: false
        },
      }
    },
    plugins: [{
      beforeDraw: function (chart, args, options) {
        writeCanvasWhite()
      },
      afterRender: function (chart, args, options) {
        writeCanvas()
      }
    }]
  });
}
const getLabels = () => {
  const labels = []
  for (let i = 0; i < timeData.length; i++) {
    if (['00:00', '03:00', '06:00', '09:00', '12:00', '15:00', '18:00', '21:00'].includes(timeData[i])) {
      labels.push(Number(timeData[i].substr(0, 2)))
    } else {
      labels.push("")
    }
  }
  labels.push(Number(0))
  return labels
}
const writeCanvasWhite = () => {
  const canvas = document.getElementById("chart")
  const ctx = canvas.getContext("2d")

  ctx.beginPath()
  ctx.fillStyle = "white"
  ctx.fillRect(0, 0, canvas.clientWidth, canvas.clientHeight)
}
const preWeek = () =>{
  date = date.subtract(1, 'w')
  yr = date.get('year')
  mn = date.get('month') + 1
  dy = date.get('date')
  init()
}
const nextWeek = () =>{
  date = date.add(1, 'w')
  yr = date.get('year')
  mn = date.get('month') + 1
  dy = date.get('date')
  init()
}
const writeCanvas = () => {
  const canvas = document.getElementById("chart")
  const ctx = canvas.getContext("2d")

  // boxes
  ctx.strokeStyle = 'blue'
  ctx.lineWidth = 3
  ctx.rect(2, 2, canvas.clientWidth -14, canvas.clientHeight - 3)
  ctx.stroke()

  ctx.beginPath()
  ctx.lineWidth = 3
  ctx.rect(5, 5, canvas.clientWidth - 18, canvas.clientHeight/6.6)
  ctx.stroke()
  ctx.fillStyle = "white"
  ctx.fillRect(4, 4, canvas.clientWidth - 17, canvas.clientHeight/6.6)

  ctx.beginPath()
  ctx.lineWidth = 3
  ctx.moveTo(0, canvas.clientHeight/6.6/2 + 2.5)
  ctx.lineTo(canvas.clientWidth - 13, canvas.clientHeight/6.6/2 + 2.5)
  ctx.stroke()

  // vertical lines
  ctx.beginPath()
  ctx.lineWidth = 3
  ctx.moveTo(44, 2)
  ctx.lineTo(44, canvas.clientHeight/6.6 + 3)
  ctx.stroke()

  ctx.beginPath()
  ctx.lineWidth = 3
  ctx.moveTo(44 + (canvas.clientWidth-47)/7.02, 2)
  ctx.lineTo(44 + (canvas.clientWidth-47)/7.02, canvas.clientHeight/6.6 + 3)
  ctx.stroke()

  ctx.beginPath()
  ctx.lineWidth = 3
  ctx.moveTo(44 + (canvas.clientWidth-47)/7.045*2, 2)
  ctx.lineTo(44 + (canvas.clientWidth-47)/7.045*2, canvas.clientHeight/6.6 + 3)
  ctx.stroke()

  ctx.beginPath()
  ctx.lineWidth = 3
  ctx.moveTo(44 + (canvas.clientWidth-47)/7.039*3, 2)
  ctx.lineTo(44 + (canvas.clientWidth-47)/7.039*3, canvas.clientHeight/6.6 + 3)
  ctx.stroke()

  ctx.beginPath()
  ctx.lineWidth = 3
  ctx.moveTo(44 + (canvas.clientWidth-47)/7.049*4, 2)
  ctx.lineTo(44 + (canvas.clientWidth-47)/7.049*4, canvas.clientHeight/6.6 + 3)
  ctx.stroke()

  ctx.beginPath()
  ctx.lineWidth = 3
  ctx.moveTo(44 + (canvas.clientWidth-48)/7.039*5, 2)
  ctx.lineTo(44 + (canvas.clientWidth-48)/7.039*5, canvas.clientHeight/6.6 + 3)
  ctx.stroke()

  ctx.beginPath()
  ctx.lineWidth = 3
  ctx.moveTo(44 + (canvas.clientWidth-48)/7.039*6, 2)
  ctx.lineTo(44 + (canvas.clientWidth-48)/7.039*6, canvas.clientHeight/6.6 + 3)
  ctx.stroke()

  // vertical bottom
  ctx.beginPath()
  ctx.lineWidth = 3
  ctx.moveTo(44, canvas.clientHeight - 3)
  ctx.lineTo(44, canvas.clientHeight - 30)
  ctx.stroke()

  ctx.beginPath()
  ctx.lineWidth = 3
  ctx.moveTo(44 + (canvas.clientWidth-47)/7.02, canvas.clientHeight - 3)
  ctx.lineTo(44 + (canvas.clientWidth-47)/7.02, canvas.clientHeight - 30)
  ctx.stroke()

  ctx.beginPath()
  ctx.lineWidth = 3
  ctx.moveTo(44 + (canvas.clientWidth-47)/7.045*2, canvas.clientHeight - 3)
  ctx.lineTo(44 + (canvas.clientWidth-47)/7.045*2, canvas.clientHeight - 30)
  ctx.stroke()

  ctx.beginPath()
  ctx.lineWidth = 3
  ctx.moveTo(44 + (canvas.clientWidth-47)/7.039*3, canvas.clientHeight - 3)
  ctx.lineTo(44 + (canvas.clientWidth-47)/7.039*3, canvas.clientHeight - 30)
  ctx.stroke()

  ctx.beginPath()
  ctx.lineWidth = 3
  ctx.moveTo(44 + (canvas.clientWidth-47)/7.049*4, canvas.clientHeight - 3)
  ctx.lineTo(44 + (canvas.clientWidth-47)/7.049*4, canvas.clientHeight - 30)
  ctx.stroke()

  ctx.beginPath()
  ctx.lineWidth = 3
  ctx.moveTo(44 + (canvas.clientWidth-48)/7.039*5, canvas.clientHeight - 3)
  ctx.lineTo(44 + (canvas.clientWidth-48)/7.039*5, canvas.clientHeight - 30)
  ctx.stroke()

  ctx.beginPath()
  ctx.lineWidth = 3
  ctx.moveTo(44 + (canvas.clientWidth-48)/7.039*6, canvas.clientHeight - 3)
  ctx.lineTo(44 + (canvas.clientWidth-48)/7.039*6, canvas.clientHeight - 30)
  ctx.stroke()

  // text
  ctx.fillStyle = "blue"
  ctx.beginPath()
  ctx.font = '22px sans-serif'
  ctx.textAlign = 'end'
  const week = ['(月)', '(火)', '(水)', '(木)', '(金)', '(土)', '(日)']
  let i = 0
  let preMonth = ''
  for (let day in chart.value) {
    let text = preMonth !== day.substring(5, 7) ? Number(day.substring(5, 7)) + '月 ' : ''
    text = text + Number(day.substring(8, 10)) + '日 ' + week.shift()
    ctx.fillText(text, 230 + i * 192, 44)
    preMonth = day.substring(5, 7)

    let text2 = chart.value[day].moon.title.substring(0, 1)
    ctx.fillText(text2, 80 + i * 192, 100)

    let text3 = chart.value[day].sun.rise //.substring(0, 1)
    ctx.fillText(text3, 220 + i * 192, 85)
    let text4 = chart.value[day].sun.set//.substring(0, 1)
    ctx.fillText(text4, 220 + i * 192, 105)
    i++
    preMonth = day.substring(5, 7)
  }
}
const init = async () =>{
  myChart?.destroy()
  tideData = []
  timeData = []
  try {
    const response = await axios.get("https://tide736.net/api/get_tide.php", {
      params: {
        pc: '12',
        hc: '18',
        yr: yr,
        mn: mn,
        dy: dy,
        rg: 'week'
      },
    },{
      headers: {
        'Content-Type': 'application/json'
      }
    })
    chart.value = response.data.tide.chart
  } catch (error) {
    console.log(error)
  }
  renderChart();
}

onMounted(async () => {
  init()
})
</script>