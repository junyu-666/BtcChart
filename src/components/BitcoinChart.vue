<template>
  <div class="bitcoin-chart">
    <div id="chart"></div>
    <div class="error-message" v-if="errorMessage">
      {{ errorMessage }}
    </div>
    <div class="date-label" v-if="currentDate">
      {{ currentDate }}
    </div>
  </div>
</template>

<script>
import * as d3 from 'd3';

export default {
  name: 'BitcoinChart',
  data() {
    return {
      svg: null,
      line: null,
      xScale: null,
      yScale: null,
      data: [],
      priceData: [],
      animationTimer: null,
      currentPrice: null,
      width: 800,
      height: 400,
      margin: { top: 20, right: 120, bottom: 30, left: 50 },
      errorMessage: null, // 错误信息
      currentDate: null, // 添加新的数据属性
      currentIndex: 1, // 添加 currentIndex 到 data
    };
  },
  mounted() {
    this.initChart();
    this.fetchBitcoinData();
  },
  beforeUnmount() {
    if (this.animationTimer) {
      clearInterval(this.animationTimer);
    }
  },
  methods: {
    initChart() {
      const svg = d3.select('#chart')
        .append('svg')
        .attr('width', this.width + this.margin.left + this.margin.right)
        .attr('height', this.height + this.margin.top + this.margin.bottom + 80)
        .append('g')
        .attr('transform', `translate(${this.margin.left},${this.margin.top})`);

      this.svg = svg;

      // 初始化比例尺
      this.xScale = d3.scaleTime()
        .range([0, this.width]);

      this.yScale = d3.scaleLinear()
        .range([this.height, 0]);

      // 创建坐标轴
      this.xAxis = svg.append('g')
        .attr('class', 'x-axis')
        .attr('transform', `translate(0,${this.height})`);

      this.yAxis = svg.append('g')
        .attr('class', 'y-axis');

      this.yAxis.call(d3.axisLeft(this.yScale));

      // 隐藏所有轴线（包括主轴线和刻度线）
      this.yAxis.selectAll('.tick line')
        .style('display', 'none');
      this.yAxis.selectAll('path.domain') // 隐藏主轴线
        .style('display', 'none');
      this.yAxis.selectAll('.tick text')
        .style('display', 'block');

      // 创建平滑曲线生成器
      this.line = d3.line()
        .x(d => this.xScale(new Date(d.date)))
        .y(d => this.yScale(d.price))
        .curve(d3.curveCatmullRom); // 使用Catmull-Rom曲线

      // 创建路径元素
      this.path = svg.append('path')
        .attr('class', 'line')
        .attr('fill', 'none')
        .attr('stroke', '#ff9900')
        .attr('stroke-width', 2);

      // 添加 x 轴的初始化调用和样式
      this.xAxis.call(d3.axisBottom(this.xScale));

      // 修改 x 轴主轴线的样式
      this.xAxis.select('path.domain')
        .attr('d', `M0.5,0.5H${this.width + 0.5}`); // 使用 SVG path 命令画一条完整的直线

      // 添加进度条背景
      svg.append('rect')
        .attr('class', 'progress-bg')
        .attr('x', 0)
        .attr('y', this.height + 60)
        .attr('width', this.width)
        .attr('height', 12)
        .attr('fill', '#eee')
        .attr('rx', 6);

      // 添加进度条前景
      this.progressBar = svg.append('rect')
        .attr('class', 'progress-bar')
        .attr('x', 0)
        .attr('y', this.height + 60)
        .attr('width', 0)
        .attr('height', 12)
        .attr('fill', '#ff9900')
        .attr('rx', 6);
    },

    updateChart() {
      if (this.priceData.length === 0) return;

      // 获取要显示的数据
      let displayData;
      if (this.priceData.length >= 90) {
        // 如果 priceData 超过90个点，取最后90个
        this.priceData = this.priceData.slice(-90);
        displayData = this.priceData.slice(-90);
      } else {
        // 如果 priceData 不足90个点，从 data 中取前90个
        displayData = this.data.slice(0, 90);
      }

      // 更新Y轴范围
      const yMin = d3.min(this.priceData, d => d.price);
      const yMax = d3.max(this.priceData, d => d.price);
      this.yScale.domain([yMin * 0.9, yMax * 1.1]);

      // 使用正确的方式更新 Y 轴
      this.yAxis
        .transition()
        .duration(300)
        .call(d3.axisLeft(this.yScale));

      // 确保轴线保持隐藏（包括主轴线）
      this.yAxis.selectAll('.tick line')
        .style('display', 'none');
      this.yAxis.selectAll('path.domain')
        .style('display', 'none');
      this.yAxis.selectAll('.tick text')
        .style('display', 'block');

      // 更新X轴范围
      const xMin = d3.min(displayData, d => new Date(d.date));
      const xMax = d3.max(displayData, d => new Date(d.date));
      this.xScale.domain([xMin, xMax]);

      // 更新坐标轴
      this.xAxis.call(
        d3.axisBottom(this.xScale)
          .ticks(10)
          .tickFormat(d3.timeFormat("%m/%d"))
      );
      this.yAxis.call(d3.axisLeft(this.yScale));

      // 更新路径数据
      this.path
        .datum(this.priceData)
        .attr('d', this.line);

      // 在最后一个数据点加一个圆点，并在圆点右侧展示价格
      const lastDataPoint = this.priceData[this.priceData.length - 1];
      this.svg.selectAll('.last-point').remove();
      this.svg.append('circle')
        .attr('class', 'last-point')
        .attr('cx', this.xScale(new Date(lastDataPoint.date)))
        .attr('cy', this.yScale(lastDataPoint.price))
        .attr('r', 4)
        .attr('fill', '#ff9900');
      this.svg.append('text')
        .attr('class', 'last-point')
        .attr('x', this.xScale(new Date(lastDataPoint.date)) + 5)
        .attr('y', this.yScale(lastDataPoint.price))
        .attr('dy', '.35em')
        .attr('fill', '#ff9900')
        .style('font-weight', 'bold')
        .text(`$${lastDataPoint.price.toFixed(2)}`);

      // 更新当前价格
      this.currentPrice = lastDataPoint?.price;
      this.currentDate = lastDataPoint ? new Date(lastDataPoint.date).toLocaleDateString('zh-CN') : null;

      // 重新应用 x 轴主轴线的样式
      this.xAxis.select('path.domain')
        .attr('d', `M0.5,0.5H${this.width + 0.5}`);

      // 更新进度条
      this.progressBar
        .transition()
        .duration(100)
        .attr('width', this.width * (this.currentIndex / this.data.length));
    },

    startAnimation() {
      this.currentIndex = 1;
      const animationSpeed = 100;

      this.priceData = this.data.slice(0, this.currentIndex);
      this.updateChart();

      if (this.animationTimer) {
        clearInterval(this.animationTimer);
      }

      this.animationTimer = setInterval(() => {
        if (this.currentIndex >= this.data.length) {
          clearInterval(this.animationTimer);
          return;
        }

        this.priceData.push(this.data[this.currentIndex]);
        this.currentIndex++;
        this.updateChart();
      }, animationSpeed);
    },

    fetchBitcoinData() {
      // 使用 CoinGecko API 获取比特币价格数据
      const today = new Date();
      const oneYearAgo = new Date(today.setFullYear(today.getFullYear() - 1));
      const fromTimestamp = Math.floor(oneYearAgo.getTime() / 1000);
      
      fetch(`https://api.coingecko.com/api/v3/coins/bitcoin/market_chart/range?vs_currency=usd&from=${fromTimestamp}&to=${Math.floor(Date.now() / 1000)}`)
        .then(response => {
          if (!response.ok) {
            throw new Error('网络响应不正常');
          }
          return response.json();
        })
        .then(data => {
          // CoinGecko 返回的数据格式为 [timestamp, price] 数组
          this.data = data.prices.map(item => ({
            date: new Date(item[0]),
            price: item[1]
          }));
          this.startAnimation(); // 获取数据后开始动画
          this.errorMessage = null; // 清除错误信息
        })
        .catch(error => {
          console.error('获取比特币数据失败:', error);
          this.errorMessage = '比特币价格获取失败，请稍后重试';
        });
    }
  },
  computed: {
    progressPercentage() {
      if (!this.data.length || !this.priceData.length) return 0;
      return (this.currentIndex / this.data.length) * 100;
    }
  }
};
</script>

<style scoped>
.bitcoin-chart {
  padding: 20px;
  position: relative;
}

.error-message {
  font-size: 16px;
  color: red;
  margin-top: 10px;
  text-align: center;
}

.line {
  transition: all 0.3s ease;
}

.date-label {
  position: absolute;
  top: -20px;
  left: 70px;
  font-size: 24px;
  font-weight: bold;
  color: #666;
}

/* 删除旧的进度条样式 */
.progress-bar, .progress {
  display: none;
}
</style> 