<script setup>
import { ref, onMounted, reactive } from 'vue'
import axios from 'axios'
import { Advisor, ckbDict, ckb } from '@antv/ava'
import { Chart } from '@antv/g2'
const container = ref(null)
let chartInstance = null
const state = reactive({
  content: '',
  loading: false,
  chart: {},
  tableData: [],
  tableColumn: [],
  isTable: false,
  text: '',
})

const ckbCfg = {
  include: [
    'area_chart',
    'line_chart',
    'column_chart',
    'pie_chart',
    'scatter_plot',
  ],
}
onMounted(() => {
  console.log('localStorage:')
  for (var i = 0; i < localStorage.length; i++) {
    console.log(
      localStorage
        .key(i)
        ?.replace('http://10.83.28.168:8001/fun_bot/demo?content=', '')
    )
  }
})
onMounted(() => {
  chartInstance = new Chart({
    container: container.value,
    theme: 'classic',
  })
})

onMounted(() => {
  document.addEventListener('keydown', function (event) {
    if (event.key === 'Enter') {
      submit()
    }
  })
})

const renderChart = (responseData) => {
  const { chart_type, columns, data } = responseData
  console.group('responseData')
  console.log(chart_type)
  console.log(columns)
  console.log(data)
  console.groupEnd()

  if (chart_type !== 'default') {
    const ruleCfg = {
      custom: {
        'appoint-chart': {
          id: 'appoint-chart',
          type: 'SOFT',
          docs: {
            lintText:
              "We do not use line chart if there is any field named 'year'",
          },
          trigger: (a) => {
            console.log(a)
            return true
          },
          validator: (args) => {
            if (args.chartType !== chart_type) {
              return 0.00001
            } else {
              return 1
            }
          },
        },
      },
    }
    const myChartAdvisor = new Advisor({
      ruleCfg: ruleCfg,
      ckbCfg: ckbCfg,
    })

    const adviseResults = myChartAdvisor.advise({
      data: data,
    })
    console.log(adviseResults)
    if (
      adviseResults.length > 0 &&
      adviseResults.some((item) => {
        return item.type === chart_type
      })
    ) {
      state.text = `指定${chart_type},并渲染`
      console.log(adviseResults)
      chartInstance.options(adviseResults[0].spec)
      chartInstance.render()
    } else {
      state.text = `指定${chart_type},但数据不符合图表渲染条件，改为渲染图表`
      state.isTable = true
      state.tableColumn = columns
      state.tableData = data
    }
  } else {
    const myChartAdvisor = new Advisor({
      ckbCfg: ckbCfg,
    })

    const adviseResults = myChartAdvisor.advise({
      data: data,
    })
    console.log(adviseResults)
    if (adviseResults.length === 0) {
      state.text = `未指定支持的图表，且智能推荐都不符合，渲染表格`
      state.isTable = true
      state.tableColumn = columns
      state.tableData = data
    } else {
      state.text = `未指定支持的图表，智能推荐图表`
      chartInstance.options(adviseResults[0].spec)
      chartInstance.render()
      state.isTable = false
    }
  }
}

const submit = async () => {
  state.loading = true
  const url = `http://10.83.28.168:8001/fun_bot/demo?content=${state.content}`
  let res
  if (localStorage.getItem(url) === null) {
    res = await axios.get(
      `http://10.83.28.168:8001/fun_bot/demo?content=${state.content}`
    )
    localStorage.setItem(url, JSON.stringify(res))
  } else {
    res = JSON.parse(localStorage.getItem(url))
  }
  state.loading = false
  const responseData = res.data.data
  if (responseData) {
    renderChart(responseData)
  }
}
</script>

<template>
  <div class="main-box" v-loading="state.loading">
    <div class="tips">支持的图表: 面积图、折线图、柱状图、饼图、散点图</div>
    <div class="query">
      <div class="query-content">输入查询问题：</div>
      <el-input v-model="state.content"></el-input>
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
      <el-button @click="submit">查询</el-button>
    </div>
    <div class="text" v-show="state.text">{{ state.text }}</div>
    <div class="chart" v-show="!state.isTable">
      <div ref="container"></div>
    </div>
    <div class="table" v-show="state.isTable">
      <el-table :data="state.tableData">
        <el-table-column
          v-for="item in state.tableColumn"
          :prop="item.key"
          :label="item.value"
          width="180"
        />
      </el-table>
    </div>
  </div>
</template>

<style scoped>
.main-box {
  display: flex;
  flex-direction: column;
  width: 80%;
  max-width: 800px;
  .query {
    display: flex;
    flex-direction: row;
    margin-bottom: 50px;
  }
  .query-content {
    width: 150px;
  }
  .tips {
    font-size: 12px;
    margin-bottom: 20px;
    color: #999;
  }
}
</style>
