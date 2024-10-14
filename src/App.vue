<!-- App.vue -->
<template>
  <div id="app">
    <h1>尘白禁区后勤词条折算总分刷取计算器</h1>
    <hr />

    <!-- 权重输入框区域，用于用户输入 9 个属性的权重 -->
    <!-- <h2>调整词条权重(0-1)</h2> -->
    <div class="input-grid">
      <div v-for="(weight, index) in weights" :key="index">
        <label :for="'weight' + (index + 1)">{{ standardValues[index] }}{{ weightsName[index] }}=</label>
        <input :id="'weight' + (index + 1)" type="number" v-model.number="weights[index]" :step="weightsStep[index]"
          min="0" style="width: 50px;" />
        <label>分</label>
      </div>
    </div>

    <!-- 按钮：计算总分并排序 -->
    <div class="button-grid">
      <button @click="calculateAndSort">计算总分并排序</button>
      <button @click="clearWeight">重置分数</button>
      <button @click="toggleDescription">
        {{ showDescription ? '隐藏说明' : '显示说明' }}
      </button>
      <select @change="loadPreset" v-model="selectedPreset">
        <option value="" disabled>选择预设</option>
        <option v-for="(preset, name) in presetOptions" :key="name" :value="name">
          {{ name }}
        </option>
      </select>
      <div><label>目标成功概率 </label>
        <input type="number" v-model.number="ptarget" :step="0.1" min="0" max="1" style="width: 50px;" />
      </div>
    </div>
    <div v-show="showDescription">
    <ul>
      <li>假定词条种类和档位的抽取是等概率完全随机的，不存在保底等特殊机制。不考虑第三词条。</li>
      <li>预设配置仅供参考。实际战斗环境不同或关注的角色不同，折算分数可能发生变化，可按实际需求调整。</li>
      <li>“出货率”表示获得1个后勤时其属性在<b>该档位及以上</b>的概率。</li>
      <li>“出货期望体力”根据<b>获取1个限定后勤100体力</b>，乘以出货期望次数 (1/p)得到。</li>
      <li>“目标成功概率”表示<b>希望有多少概率能够达成目标</b>，根据 log(1-p成功)/log(1-p) 计算概率得到表中最后一列。</li>
      <li>由于官方表示后勤刷取将要优化，所以暂时不考虑进一步开发更多功能了。</li>
    </ul>
  </div>
    <hr />


    <!-- 展示排序结果 -->
    <table>
      <thead>
        <tr>
          <th>排位</th>
          <th>属性</th>
          <th>折算总分</th>
          <th>出货率</th>
          <th>出货期望体力</th>
          <th>达到目标概率所需体力</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="(item, index) in paginatedObjects" :key="index">
          <td>{{ item.rank }}</td>
          <td>{{ getFilteredAttributes(item).join(', ') }}</td>
          <td>{{ item.totalScore.toFixed(2) }}</td>
          <td>{{ item.prob.toFixed(3) }}</td>
          <td>{{ (item.expectres).toFixed(0) }}</td>
          <td>{{ item.successres.toFixed(0) }}</td>
        </tr>
      </tbody>
    </table>
  
  <!-- 分页控制按钮 -->
  <div class="pagination">
    <button @click="prevPage" :disabled="currentPage === 1">上一页</button>
    <span>当前页: {{ currentPage }} / {{ totalPages }}</span>
    <button @click="nextPage" :disabled="currentPage === totalPages">下一页</button>
  </div>
  <!-- 添加 Chart 组件，传入 chartData 和 options -->
  <div class="chart-container">
    <line-chart :data="chartData" :options="chartOptions" />
  </div>
</div>
  <footer>
    <a class="Github" href="https://github.com/kokobird28144/cbjq-hq" target="_blank" title="GitHub Repository">
      Github
    </a>
  </footer>
</template>

<script>
import { Line } from 'vue-chartjs';
import { Chart as ChartJS, Title, Tooltip, Legend, LineElement, CategoryScale, LinearScale, PointElement } from 'chart.js';
ChartJS.register(Title, Tooltip, Legend, LineElement, CategoryScale, LinearScale, PointElement);

import presetsData from './presets.json'; // 导入预设 JSON 数据文件

export default {
  components: {
    LineChart: Line
  },
  data() {
    return {
      standardValues: [10, 10, 10, 106, 5.3, 10.6, 19.3, 14.1, 14.1], // 每个属性的标准值
      weights: Array(9).fill(0), // 用户输入的 9 个权重，初始值为 0
      weightsUsed: Array(9).fill(0),
      weightsStep: ['1', '1', '1', '1', '1', '1', '1', '1', '1'],
      weightsName: ['攻击力%', '生命值%', '防御力%', '同调', '暴击%', '暴伤%', '技能急速', '常规能量%', '爆发能量%'],
      ptarget: 0.7,
      objects: [], // 存储生成的 900 个对象
      displayedObjects: [], // 最终显示的对象列表（前 40 位）
      presetOptions: presetsData, // 存储预设选项（从 JSON 文件中读取）
      selectedPreset: "", // 当前选中的预设名称
      currentPage: 1, // 当前页码
      pageSize: 20, // 每页显示的象数量（默认 10）
      showDescription: true,
      chartOptions: {
        responsive: true, // 响应式
        maintainAspectRatio: false, // 保持宽高比
        scales: {
          x: {
            type: 'linear',
            min: 0,
            max: 10000,
            title: {
              display: true,
              text: '体力' // X轴标题
            }
          },
          y: {
            title: {
              display: true,
              text: '折算总分' // Y轴标题
            }
          }
        }
      }
    };
  },
  created() {
    // 在组件创建时生成 900 个对象
    this.generateObjects();
  },
  mounted() {
    // this.initChartData();
  },
  computed: {
    // 计算总页数
    totalPages() {
      return Math.ceil(this.displayedObjects.length / this.pageSize);
    },

    // 计算当前页需要显示的对象列表
    paginatedObjects() {
      const start = (this.currentPage - 1) * this.pageSize;
      const end = start + this.pageSize;
      return this.displayedObjects.slice(start, end);
    },

    chartData() {
      // let datalabels = this.displayedObjects.map(item => item.expectres); // 设置 X 轴的 labels
      // let data = this.displayedObjects.map(item => item.totalScore); // 设置 Y 轴的数据
      return {
        labels: [], // X轴标签
        datasets: [
          {
            label: '折算总分-出货期望体力', // 数据集名称
            backgroundColor: '#42A5F5', // 背景色
            borderColor: '#1E88E5', // 边框色
            data: this.displayedObjects.map(item => ({
              x: item.expectres, // 将 expectres 作为横坐标
              y: item.totalScore // 将 totalScore 作为纵坐标
            })), // Y轴数据
          },
          {
            label: '折算总分-达到目标概率所需体力', // 数据集名称
            backgroundColor: '#F5AB00', // 背景色
            borderColor: '#c4860d', // 边框色
            data: this.displayedObjects.map(item => ({
              x: item.successres, // 将 expectres 作为横坐标
              y: item.totalScore // 将 totalScore 作为纵坐标
            })), // Y轴数据
          }
        ]
      }
    }
  },
  methods: {
    // 生成 900 个对象
    generateObjects() {
      const standardValues = this.standardValues; // 引用标准值数组
      const numAttributes = standardValues.length; // 属性数量
      const combinations = this.getCombinations(numAttributes, 2); // 获取 9 个属性中任取 2 个属性的组合

      // 倍数选项
      const multipliers = [1, 0.85, 0.7, 0.55, 0.4]; // 每个属性取标准值的倍数

      // 遍历每种属性组合（36 种组合）
      combinations.forEach(([attrIndex1, attrIndex2]) => {
        // 对于每种属性组合，生成 25 个对象
        multipliers.forEach(multiplier1 => {
          multipliers.forEach(multiplier2 => {
            // 创建一个新对象，初始化所有属性值为 0
            const newObject = {
              attr1: 0,
              attr2: 0,
              attr3: 0,
              attr4: 0,
              attr5: 0,
              attr6: 0,
              attr7: 0,
              attr8: 0,
              attr9: 0,
              attrscore: [0, 0, 0, 0, 0, 0, 0, 0, 0],
              totalScore: 0, // 总分初始化为 0
              rank: 0, // 排名初始化为 0
              prob: 0,
              expectres: 0,
              successres: 0,
            };

            // 设置选定属性的值（根据标准值和倍数）
            newObject[`attr${attrIndex1 + 1}`] = standardValues[attrIndex1] * multiplier1;
            newObject[`attr${attrIndex2 + 1}`] = standardValues[attrIndex2] * multiplier2;

            // 将新对象添加到对象数组中
            this.objects.push(newObject);
          });
        });
      });
    },

    // 获取从 numAttributes 个属性中任取 r 个属性的所有组合
    getCombinations(numAttributes, r) {
      const result = [];
      const indices = Array(r).fill(0).map((_, i) => i); // 初始化索引数组

      // 生成组合
      while (indices[0] < numAttributes - r + 1) {
        result.push([...indices]); // 添加当前组合
        let i = r - 1;

        while (i >= 0 && indices[i] === numAttributes - r + i) {
          i--;
        }

        if (i >= 0) {
          indices[i]++;
          for (let j = i + 1; j < r; j++) {
            indices[j] = indices[j - 1] + 1;
          }
        } else {
          break;
        }
      }

      return result;
    },

    clearWeight() {
      for (let i = 1; i <= 9; i++) {
        this.weights[i - 1] = 0;
      }
      this.displayedObjects = [];
      this.currentPage = 1;
      this.selectedPreset = "";
    },

    // 加载选中的预设权重
    loadPreset() {
      if (this.selectedPreset && this.presetOptions[this.selectedPreset]) {
        // 使用选中的预设值更新 weights 数组
        this.weights = [...this.presetOptions[this.selectedPreset]];
      } else {
        alert("请选择一个有效的预设选项！");
      }
    },

    // 根据用户输入的 9 个权重计算每个对象的总分，并进行排序
    calculateAndSort() {
      // 计算每个对象的总分
      this.objects.forEach(obj => {
        obj.totalScore = 0;
        for (let i = 1; i <= 9; i++) {
          const attrValue = obj[`attr${i}`]; // 属性值
          const weight = this.weights[i - 1]; // 对应的权重
          this.weightsUsed[i - 1] = weight;
          obj.attrscore[i - 1] = attrValue * weight / this.standardValues[i - 1];
          obj.totalScore += obj.attrscore[i - 1];
        }
      });

      // 按照总分进行降序排序，并记录排名
      this.objects.sort((a, b) => b.totalScore - a.totalScore);

      let rank = 1;
      let lastScore = -1;
      let cumprob = 1 / 900;
      let sameScoreIndex = [];
      // 记录排名，并处理总分相同的情况
      this.objects.forEach((obj, index, arr) => {
        obj.prob = cumprob;
        obj.expectres = 100 / cumprob;
        obj.successres = (100 * Math.log(1 - this.ptarget) / Math.log(1 - cumprob))
        if (Math.abs(obj.totalScore - lastScore) > 0.000001) {
          obj.rank = rank; // 更新排名
          rank++; // 增加排名
          lastScore = obj.totalScore; // 更新最后的总分
          sameScoreIndex.forEach((obj1) => {
            arr[obj1].prob = arr[index - 1].prob;
            arr[obj1].expectres = arr[index - 1].expectres;
            arr[obj1].successres = arr[index - 1].successres;
          })
          sameScoreIndex = [index];
        } else {
          // 如果总分相同，保持 rank 不变
          obj.rank = rank - 1;
          sameScoreIndex.push(index)
        }
        cumprob += 1 / 900;
      });

      // 筛选显示的对象
      this.displayedObjects = this.objects.filter((obj, index, arr) => {
        if (index === 899) {
          return true;
        }
        if (obj.totalScore !== arr[index + 1].totalScore) {
          return true;
        }
        if (JSON.stringify(obj.attrscore) === JSON.stringify(arr[index + 1].attrscore)) {
          return false;
        } else {
          return true;
        }
      });

      this.currentPage = 1;
    },

    // 过滤对象中属性值和对应权重均不为 0 的属性
    getFilteredAttributes(item) {
      const filteredAttributes = [];
      for (let i = 1; i <= 9; i++) {
        const attrValue = item[`attr${i}`];
        const weight = this.weightsUsed[i - 1];
        if (attrValue !== 0 && weight !== 0) {
          filteredAttributes.push(`${attrValue.toFixed(1)}${this.weightsName[i - 1]}`);
        }
      }
      if (filteredAttributes.length < 2) {
        filteredAttributes.push('无效词条');
      }
      return filteredAttributes;
    },
    // 切换到上一页
    prevPage() {
      if (this.currentPage > 1) {
        this.currentPage--;
      }
    },

    // 切换到下一页
    nextPage() {
      if (this.currentPage < this.totalPages) {
        this.currentPage++;
      }
    },
    toggleDescription() {
      this.showDescription = !this.showDescription; // 切换显示状态
    },
  }

};
</script>

<style>
/* 样式部分，可根据实际需求调整 */
#app {
  display: flex;
  align-items: center;
  /* 垂直居中 */
  min-height: 100vh;
  /* 占满整个视口高度 */
  flex-direction: column;
  /* 使内容按列布局 */
}

button {
  margin: 10px;
}

table {
  width: 100%;
  border-collapse: collapse;
  margin-top: 20px;
}

th,
td {
  border: 1px solid #ddd;
  padding: 8px;
  text-align: center;
}

th {
  background-color: #f4f4f4;
  font-weight: bold;
}

hr {
  border: 0;
  width: 100%;
  /* 去除默认边框样式 */
  height: 1px;
  /* 设置分割线的高度 */
  background-color: black;
  /* 设置分割线颜色 */
  margin: 20px 0px;
  /* 设置分割线的上下边距 */
}

/* 样式部分：定义 .input-grid 的布局 */
.input-grid {
  display: grid;
  /* 使用 CSS Grid 布局 */
  grid-template-columns: repeat(3, 1fr);
  /* 定义 3 列布局，每列平分宽度 */
  gap: 10px;
  /* 设置网格间距 */
}

.button-grid {
  margin-top: 20px;
  display: grid;
  /* 使用 CSS Grid 布局 */
  grid-template-columns: repeat(3, 1fr);
  /* 定义 3 列布局，每列平分宽度 */
  gap: 10px;
  /* 设置网格间距 */
}

/* 样式部分：分页按钮样式 */
.pagination {
  display: flex;
  justify-content: center;
  align-items: center;
  margin-top: 20px;
}

.pagination button {
  margin: 0 10px;
  padding: 5px 10px;
  cursor: pointer;
}

.pagination span {
  font-size: 16px;
  margin: 0 10px;
}

.chart-container {
  width: 100%;
  height: 30vh;
  display: flex;
  justify-content: center;
  align-items: center;
}

.footer {
  position: fixed;
  bottom: 0;
  width: 100%;
  text-align: center;
  background-color: #f9f9f9;
  padding: 10px 0;
}
</style>
