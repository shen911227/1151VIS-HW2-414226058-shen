<script setup>
import { ref, onMounted, onUnmounted, watch } from 'vue'
import * as d3 from 'd3'


const svgRef = ref(null)
const currentYear = ref(1950)
const minYear = 1950
const maxYear = 2023
const isPlaying = ref(false)
const speed = ref(4)
const growthData = ref([])

let timer = null
const topN = 10 // 畫面顯示前 10 多的國家
const chartWidth = 900
const chartHeight = 450
const margin = { top: 30, right: 50, bottom: 80, left: 120 }
const colorScale = d3.scaleOrdinal(d3.schemeTableau10)

// 從 Our World in Data 讀取人口資料

const DATA_URL =
  'https://ourworldindata.org/grapher/population.csv?v=1&csvType=full&useColumnShortNames=false'

let rawData = []

const loadPopulationData = async () => {
  try {
    const csvData = await d3.csv(DATA_URL, d3.autoType)

    rawData = csvData
      .filter(d =>
        d.Code &&
        /^[A-Z]{3}$/.test(d.Code) &&
        d.Year >= 1950 &&
        d.Year <= 2023 &&
        Number.isFinite(d.Population)
      )
      .map(d => ({
        year: d.Year,
        station: d.Entity,
        type: 'Country',
        volume: d.Population
      }))

    console.log('人口資料載入成功：', rawData.length)

    return true
  } catch (error) {
    console.error('人口資料載入失敗：', error)
    return false
  }
}

// 取得指定年份的資料並排序取前 10
const getDataByYear = (year) => {
  const yearData = rawData.filter(d => d.year === year)
  return yearData
    .sort((a, b) => b.volume - a.volume)
    .slice(0, topN)
}

// 計算各國相較於前一年的成長
const getGrowthData = (year) => {
  if (year === minYear) {
    return getDataByYear(year).map(d => ({
      station: d.station,
      population: d.volume,
      increase: null,
      growthRate: null
    }))
  }

  const currentData = getDataByYear(year)

  return currentData.map(d => {
    const previousData = rawData.find(
      item =>
        item.station === d.station &&
        item.year === year - 1
    )

    if (!previousData || previousData.volume === 0) {
      return null
    }

    const increase = d.volume - previousData.volume

    const growthRate =
      (increase / previousData.volume) * 100

    return {
      station: d.station,
      population: d.volume,
      increase,
      growthRate
    }
  })
  .filter(Boolean)
  .sort((a, b) => b.increase - a.increase)
}

// D3 核心繪圖與轉場邏輯
let svg, x, y, xAxisGroup, yAxisGroup, barsGroup

const initChart = () => {
  const el = svgRef.value
  d3.select(el).selectAll('*').remove()

  svg = d3.select(el)
    .attr('viewBox', `0 0 ${chartWidth} ${chartHeight}`)
    .append('g')
    .attr('transform', `translate(${margin.left}, ${margin.top})`)

  const width = chartWidth - margin.left - margin.right
  const height = chartHeight - margin.top - margin.bottom

  x = d3.scaleLinear().range([0, width])
  y = d3.scaleBand().range([0, height]).padding(0.15)

  xAxisGroup = svg.append('g').attr('transform', `translate(0, ${height})`)
  yAxisGroup = svg.append('g')
  barsGroup = svg.append('g')

  // X 軸 人口數
  svg.append('text')
    .attr('x', width / 2)
    .attr('y', height + 55)
    .attr('fill', '#ffffff')
    .attr('text-anchor', 'middle')
    .attr('font-size', '14px')
    .text('人口數（Population）')

  // Y 軸 國家
  svg.append('text')
    .attr('transform', 'rotate(-90)')
    .attr('x', -height / 2)
    .attr('y', -95)
    .attr('fill', '#ffffff')
    .attr('text-anchor', 'middle')
    .attr('font-size', '14px')
    .text('國家（Country）')


  updateChart(currentYear.value)
}

const updateChart = (year) => {
  const data = getDataByYear(year)
  if (!data.length) return

  growthData.value = getGrowthData(year)

  const width = chartWidth - margin.left - margin.right
  const height = chartHeight - margin.top - margin.bottom

  x.domain([0, d3.max(data, d => d.volume) * 1.15])
  y.domain(data.map(d => d.station))

  const duration = 500 / speed.value

  xAxisGroup.transition().duration(duration).call(d3.axisBottom(x).ticks(5))
  yAxisGroup.transition().duration(duration).call(d3.axisLeft(y))

  const bars = barsGroup.selectAll('g.bar-group')
    .data(data, d => d.station)

  // Enter
  const enterBars = bars.enter()
    .append('g')
    .attr('class', 'bar-group')
    .attr('transform', d => `translate(0, ${y(d.station)})`)

  enterBars.append('rect')
    .attr('x', 0)
    .attr('height', y.bandwidth())
    .attr('width', 0)
    .attr('fill', d => colorScale(d.station))
    .attr('rx', 4)

  enterBars.append('text')
    .attr('class', 'value-text')
    .attr('x', 10)
    .attr('y', y.bandwidth() / 2)
    .attr('dy', '0.35em')
    .attr('fill', '#fff')
    .attr('font-size', '12px')
    .text(d => d3.format(',')(d.volume))

  // Update + Enter 合併
  const allBars = enterBars.merge(bars)

  allBars.transition().duration(duration).ease(d3.easeLinear)
    .attr('transform', d => `translate(0, ${y(d.station)})`)

  allBars.select('rect')
    .transition().duration(duration).ease(d3.easeLinear)
    .attr('width', d => x(d.volume))

  allBars.select('text')
    .transition().duration(duration).ease(d3.easeLinear)
    .attr('x', d => x(d.volume) + 10)
    .tween('text', function(d) {
      const node = this
      const currentVal = parseInt(node.textContent.replace(/,/g, '')) || 0
      const i = d3.interpolate(currentVal, d.volume)
      return function(t) {
        node.textContent = d3.format(',')(Math.round(i(t)))
      }
    })

  // Exit
  bars.exit()
    .transition().duration(duration)
    .attr('opacity', 0)
    .remove()
}

// 控制年份自動遞增、暫停與重置
const startPlay = () => {
  if (isPlaying.value) return
  isPlaying.value = true
  
  timer = setInterval(() => {
    if (currentYear.value < maxYear) {
      currentYear.value++
    } else {
      stopPlay()
    }
  }, 1000 / speed.value)
}

const stopPlay = () => {
  isPlaying.value = false
  clearInterval(timer)
}

const resetPlay = () => {
  stopPlay()
  currentYear.value = minYear
}

watch(currentYear, (newYear) => {
  updateChart(newYear)
})

onMounted(async () => {
  const success = await loadPopulationData()

  if (success) {
    initChart()
  }
})

onUnmounted(() => {
  stopPlay()
})
</script>


<template>
  <div
    style="
      padding: 24px;
      background: #0f172a;
      color: white;
      border-radius: 12px;
      max-width: 1400px;
      width: 100%;
      margin: 0 auto;
      box-sizing: border-box;
    "
  >

    <!-- 標題 -->
    <div style="display: flex; justify-content: space-between; align-items: center;">
      <h2 style="font-size: 20px; font-weight: bold;">
        全球人口資料
      </h2>

        <h3 style="width: 35%; font-size: 18px; font-weight: bold;">
          <span v-if="currentYear === minYear">
            人口成長分析（{{ currentYear }}）
          </span>
          <span v-else>
            人口成長分析（{{ currentYear - 1 }}～{{ currentYear }}）
          </span>
        </h3>
    </div>


    <div
      style="
        display: flex;
        flex-direction: row;
        align-items: flex-start;
        gap: 24px;
        width: 100%;
      "
    >

      <!-- 左 人口長條圖 -->
      <div
        style="
          width: 65%;
          flex: 0 0 65%;
          min-width: 0;
        "
      >

        <div
          style="
            background: #020617;
            border-radius: 8px;
            padding: 12px;
            overflow: hidden;
          "
        >
          <svg
            ref="svgRef"
            style="display: block; width: 100%; height: auto;"
          ></svg>
        </div>

        <div style="display: flex; align-items: center; gap: 12px; margin-top: 24px;">

          <button
            @click="isPlaying ? stopPlay() : startPlay()"
            style="
              padding: 8px 20px;
              background: #4f46e5;
              border-radius: 6px;
              font-weight: bold;
              white-space: nowrap;
            "
          >
            {{ isPlaying ? '暫停 ⏸' : '播放 ▶' }}
          </button>

          <button
            @click="resetPlay"
            style="
              padding: 8px 16px;
              background: #334155;
              border-radius: 6px;
              white-space: nowrap;
            "
          >
            重置 ↺
          </button>

          <input
            type="range"
            :min="minYear"
            :max="maxYear"
            v-model.number="currentYear"
            style="flex: 1; min-width: 0; cursor: pointer;"
          />

        </div>
      </div>

      <!-- 右 人口成長分析 -->
      <div
        style="
          width: 35%;
          flex: 0 0 35%;
          min-width: 0;
        "
      >


        <div style="width: 100%; overflow-x: auto;">

          <table style="width: 100%; font-size: 12px; text-align: left; border-collapse: collapse;">

            <thead style="color: #94a3b8; border-bottom: 1px solid #334155;">
              <tr>
                <th style="padding: 8px 4px;">國家</th>
                <th style="padding: 8px 4px;">目前人口</th>
                <th style="padding: 8px 4px;">增加人口</th>
                <th style="padding: 8px 4px;">成長率</th>
              </tr>
            </thead>

            <tbody>
              <tr
                v-for="item in growthData"
                :key="item.station"
                style="border-bottom: 1px solid #1e293b;"
              >
                <td style="padding: 8px 4px; font-weight: bold;">
                  {{ item.station }}
                </td>

                <td style="padding: 8px 4px; white-space: nowrap;">
                  {{ Math.round(item.population).toLocaleString() }}
                </td>

                <td style="padding: 8px 4px; color: #34d399; white-space: nowrap;">
                  {{ item.increase === null ? '—' : (item.increase >= 0 ? '+' : '') + Math.round(item.increase).toLocaleString() }}
                </td>

                <td style="padding: 8px 4px; color: #38bdf8; white-space: nowrap;">
                  {{ item.growthRate === null ? '—' : item.growthRate.toFixed(2) + '%' }}
                </td>
              </tr>
            </tbody>

          </table>
        </div>
      </div>

    </div>
  </div>
</template>