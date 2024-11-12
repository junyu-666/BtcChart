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
      errorMessage: null,
      currentDate: null,
      currentIndex: 1,
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

      // 添加略缩图容器
      const miniHeight = 40;
      const handleHeight = 50; // 柄的高度比略缩图高
      const handleOverflow = (handleHeight - miniHeight) / 2; // 计算柄超出的部分
      this.miniContainer = svg.append('g')
        .attr('class', 'mini-chart')
        .attr('transform', `translate(0,${this.height + 40})`);

      // 添加略缩图背景
      this.miniContainer.append('rect')
        .attr('class', 'mini-background')
        .attr('x', 0)
        .attr('y', 0)
        .attr('width', this.width)
        .attr('height', miniHeight)
        .attr('fill', '#F5F5F5') // 使用白色加一点点灰色
        .attr('rx', 4);

      // 创建略缩图比例尺
      this.miniXScale = d3.scaleTime()
        .range([0, this.width]);

      this.miniYScale = d3.scaleLinear()
        .range([miniHeight, 0]);

      // 创建略缩图径
      this.miniLine = d3.line()
        .x(d => this.miniXScale(new Date(d.date)))
        .y(d => this.miniYScale(d.price))
        .curve(d3.curveCatmullRom);

      // 添加略缩图路径
      this.miniPath = this.miniContainer.append('path')
        .attr('class', 'mini-line')
        .attr('fill', 'none')
        .attr('stroke', '#ff9900')
        .attr('stroke-width', 1)
        .attr('opacity', 0.5);

      // 修改进度遮罩的初始位置和属性
      this.progressMask = this.miniContainer.append('rect')
        .attr('class', 'progress-mask')
        .attr('x', 0)
        .attr('y', 0)
        .attr('width', 0)
        .attr('height', miniHeight)
        .attr('fill', '#e6f3ff')
        .attr('opacity', 0.7);

      // 添加左侧柄
      this.leftHandle = this.miniContainer.append('rect')
        .attr('class', 'progress-handle left-handle')
        .attr('x', 0)
        .attr('y', -handleOverflow) // 向上偏移以居中
        .attr('width', 4)
        .attr('height', handleHeight)
        .attr('fill', '#1890ff')
        .attr('rx', 2);

      // 添加右侧柄
      this.rightHandle = this.miniContainer.append('rect')
        .attr('class', 'progress-handle right-handle')
        .attr('x', 0)
        .attr('y', -handleOverflow) // 向上偏移以居中
        .attr('width', 4)
        .attr('height', handleHeight)
        .attr('fill', '#1890ff')
        .attr('rx', 2);

      // 添加左侧日期文本
      this.startDateText = this.miniContainer.append('text')
        .attr('class', 'date-label start-date')
        .attr('x', 0)
        .attr('y', miniHeight + 20)
        .attr('text-anchor', 'start')
        .attr('fill', '#666')
        .attr('font-size', '12px');

      // 添加右侧日期文本
      this.endDateText = this.miniContainer.append('text')
        .attr('class', 'date-label end-date')
        .attr('x', this.width)
        .attr('y', miniHeight + 20)
        .attr('text-anchor', 'end')
        .attr('fill', '#666')
        .attr('font-size', '12px');

      // 添加当前进度日期文本
      this.currentDateText = this.miniContainer.append('text')
        .attr('class', 'date-label current-date')
        .attr('y', miniHeight + 20)
        .attr('text-anchor', 'middle')
        .attr('fill', '#666')  // 使用和柄相同的颜色
        .attr('font-size', '12px');

      // 创建渐变定义
      const gradient = svg.append("defs")
        .append("linearGradient")
        .attr("id", "line-gradient")
        .attr("gradientUnits", "userSpaceOnUse")
        .attr("x1", 0)
        .attr("y1", this.yScale(d3.min(this.data, d => d.price)))
        .attr("x2", 0)
        .attr("y2", this.yScale(d3.max(this.data, d => d.price)));

      // 添加渐变色停止点
      gradient.append("stop")
        .attr("offset", "0%")
        .attr("stop-color", "#00ffff");  // 青色

      gradient.append("stop")
        .attr("offset", "100%")
        .attr("stop-color", "#1890ff");  // 蓝色

      // 创建阴影滤镜
      const filter = svg.append("defs")
        .append("filter")
        .attr("id", "glow")
        .attr("width", "300%")
        .attr("height", "300%")
        .attr("x", "-100%")
        .attr("y", "-100%");

      // 添加高斯模糊效果
      filter.append("feGaussianBlur")
        .attr("class", "blur")
        .attr("stdDeviation", "4")
        .attr("result", "coloredBlur");

      // 合并原始图形和模糊效果
      const feMerge = filter.append("feMerge");
      feMerge.append("feMergeNode")
        .attr("in", "coloredBlur");
      feMerge.append("feMergeNode")
        .attr("in", "SourceGraphic");

      // 修改路径元素，使用渐变色和阴影
      this.path = svg.append('path')
        .attr('class', 'line')
        .attr('fill', 'none')
        .attr('stroke', 'url(#line-gradient)')
        .attr('stroke-width', 3)
        .attr('filter', 'url(#glow)');

      // 创建面积生成器用于阴影效果
      this.area = d3.area()
        .x(d => this.xScale(new Date(d.date)))
        .y0(this.height)
        .y1(d => this.yScale(d.price))
        .curve(d3.curveCatmullRom);

      // 添加渐变填充区域
      const areaGradient = svg.append("defs")
        .append("linearGradient")
        .attr("id", "area-gradient")
        .attr("gradientUnits", "userSpaceOnUse")
        .attr("x1", 0)
        .attr("y1", 0)
        .attr("x2", 0)
        .attr("y2", this.height);

      areaGradient.append("stop")
        .attr("offset", "0%")
        .attr("stop-color", "#1890ff")  // 使用相同的蓝色
        .attr("stop-opacity", 0.2);     // 降低透明度

      areaGradient.append("stop")
        .attr("offset", "100%")
        .attr("stop-color", "#1890ff")
        .attr("stop-opacity", 0);

      // 添加面积路径
      this.areaPath = svg.append('path')
        .attr('class', 'area')
        .attr('fill', 'url(#area-gradient)');

      // 创建点的发散动画效果
      const pulseFilter = svg.append("defs")
        .append("filter")
        .attr("id", "pulse")
        .attr("width", "300%")
        .attr("height", "300%")
        .attr("x", "-100%")
        .attr("y", "-100%");
      
      pulseFilter.append("feGaussianBlur")
        .attr("class", "blur")
        .attr("stdDeviation", "2")
        .attr("result", "coloredBlur");

      // 修改变量名为 pulseMerge
      const pulseMerge = pulseFilter.append("feMerge");
      pulseMerge.append("feMergeNode")
        .attr("in", "coloredBlur");
      pulseMerge.append("feMergeNode")
        .attr("in", "SourceGraphic");

      // 创建动画定义
      const pulseAnimation = svg.append("defs")
        .append("radialGradient")
        .attr("id", "pulseGradient");

      pulseAnimation.append("stop")
        .attr("offset", "0%")
        .attr("stop-color", "#1890ff")
        .attr("stop-opacity", 0.8);

      pulseAnimation.append("stop")
        .attr("offset", "100%")
        .attr("stop-color", "#1890ff")
        .attr("stop-opacity", 0);
    },

    updateChart() {
      if (this.priceData.length === 0) return;

      // 获取要显示的数据
      let displayData;
      if (this.priceData.length >= 120) {
        // 如果 priceData 超过120个点，取最后120个
        this.priceData = this.priceData.slice(-120);
        displayData = this.priceData.slice(-120);
      } else {
        // 如果 priceData 不足120个点，从 data 中取前120个
        displayData = this.data.slice(0, 120);
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

      // 更新面积
      this.areaPath
        .datum(this.priceData)
        .attr('d', this.area);

      // 在最后一个数据点加一个圆点，并在圆点右侧展示价格
      const lastDataPoint = this.priceData[this.priceData.length - 1];
      this.svg.selectAll('.last-point').remove();
      this.svg.selectAll('.pulse-circle').remove();
      this.svg.selectAll('.price-text').remove();

      // 添加脉冲动画圆圈
      const pulseCircle = this.svg.append('circle')
        .attr('class', 'pulse-circle')
        .attr('cx', this.xScale(new Date(lastDataPoint.date)))
        .attr('cy', this.yScale(lastDataPoint.price))
        .attr('r', 8)
        .attr('fill', '#1890ff')
        .attr('opacity', 0.3)
        .attr('filter', 'url(#pulse)');

      // 添加动画效果
      function pulseAnimate() {
        pulseCircle
          .transition()
          .duration(1500)
          .attr('r', 20)
          .attr('opacity', 0)
          .on('end', function() {
            d3.select(this)
              .attr('r', 8)
              .attr('opacity', 0.3)
              .call(() => pulseAnimate());
          });
      }
      pulseAnimate();

      // 添加中心点
      this.svg.append('circle')
        .attr('class', 'last-point')
        .attr('cx', this.xScale(new Date(lastDataPoint.date)))
        .attr('cy', this.yScale(lastDataPoint.price))
        .attr('r', 4)
        .attr('fill', '#1890ff');

      // 添加价格文本
      this.svg.append('text')
        .attr('class', 'price-text')
        .attr('x', this.xScale(new Date(lastDataPoint.date)) + 10)
        .attr('y', this.yScale(lastDataPoint.price))
        .attr('dy', '.35em')
        .attr('fill', '#333333')  // 使用黑色
        .style('font-weight', 'bold')
        .style('font-size', '14px')
        .text(`$${lastDataPoint.price.toFixed(2)}`);

      // 更新当前价格
      this.currentPrice = lastDataPoint?.price;
      this.currentDate = lastDataPoint ? new Date(lastDataPoint.date).toLocaleDateString('zh-CN') : null;

      // 重新应用 x 轴主轴线的样式
      this.xAxis.select('path.domain')
        .attr('d', `M0.5,0.5H${this.width + 0.5}`);

      // 更新略缩图
      // 使用完整的数据集更新略缩图
      const miniXMin = d3.min(this.data, d => new Date(d.date));
      const miniXMax = d3.max(this.data, d => new Date(d.date));
      const miniYMin = d3.min(this.data, d => d.price);
      const miniYMax = d3.max(this.data, d => d.price);

      this.miniXScale.domain([miniXMin, miniXMax]);
      this.miniYScale.domain([miniYMin * 0.9, miniYMax * 1.1]);

      // 更新略缩图路径
      this.miniPath
        .datum(this.data)
        .attr('d', this.miniLine);

      // 更新进度遮罩和柄的位置
      const maskWidth = this.width * (this.currentIndex / this.data.length);
      this.progressMask
        .attr('x', 0)
        .attr('width', maskWidth);

      // 更新左侧柄位置（固定在遮罩左侧）
      this.leftHandle
        .attr('x', 0);

      // 更新右侧柄位置（跟随遮罩右侧边缘）
      this.rightHandle
        .attr('x', maskWidth - 2); // 减2是为了让柄的中心对齐遮罩边缘

      // 更新日期显示
      if (this.data.length > 0) {
        const startDate = new Date(this.data[0].date);
        const endDate = new Date(this.data[this.data.length - 1].date);
        
        // 使用中文格式化日期
        const dateFormatter = new Intl.DateTimeFormat('zh-CN', {
          year: 'numeric',
          month: '2-digit',
          day: '2-digit'
        });

        this.startDateText.text(dateFormatter.format(startDate));
        this.endDateText.text(dateFormatter.format(endDate));
      }

      // 更新当前日期文本位置和内容
      if (this.currentIndex > 0 && this.currentIndex <= this.data.length) {
        const currentDate = new Date(this.data[this.currentIndex - 1].date);
        const dateFormatter = new Intl.DateTimeFormat('zh-CN', {
          year: 'numeric',
          month: '2-digit',
          day: '2-digit'
        });
        
        const currentX = maskWidth - 2; // 与右侧柄对齐
        this.currentDateText
          .attr('x', currentX)
          .text(dateFormatter.format(currentDate))
          .style('display', 'block');

        // 检查日期文本是否会重叠
        const textWidth = 100; // 估计的日期文本宽度
        const margin = 10; // 文本之间的最小间距

        // 处理开始日期文本的显示/隐藏
        this.startDateText.style('display', 
          currentX - textWidth/2 < textWidth + margin ? 'none' : 'block');

        // 处理结束日期文本的显示/隐藏
        this.endDateText.style('display', 
          currentX + textWidth/2 > this.width - textWidth - margin ? 'none' : 'block');
      }

      // 更新渐变的范围
      d3.select("#line-gradient")
        .attr("y1", this.yScale(d3.min(this.priceData, d => d.price)))
        .attr("y2", this.yScale(d3.max(this.priceData, d => d.price)));
    },

    startAnimation() {
      this.currentIndex = 1;
      const animationDuration = 300000; // 总动画时长（毫秒）
      const totalPoints = this.data.length;
      const animationSpeed = Math.floor(animationDuration / totalPoints); // 每个点的动画间隔

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

    async fetchBitcoinData() {
      try {
        // 计算时间范围
        const now = new Date();
        const endTime = now.getTime();
        const tenYearsAgo = new Date(now.setFullYear(now.getFullYear() - 8));
        let startTime = tenYearsAgo.getTime();
        
        let allData = [];
        
        // 分段获取数据，每次获取1000条
        while (startTime < endTime) {
          const response = await fetch(
            `https://api.binance.com/api/v3/klines?symbol=BTCUSDT&interval=1d&startTime=${startTime}&limit=365`
          );

          if (!response.ok) {
            throw new Error('网络响应不正常');
          }

          const data = await response.json();
          if (data.length === 0) break; // 如果没有更多数据则退出

          allData = allData.concat(data);
          
          // 更新开始时间为最后一条数的时间加一天
          startTime = data[data.length - 1][0] + 24 * 60 * 60 * 1000;
          
          // 添加延时以避免触发频率限制
          await new Promise(resolve => setTimeout(resolve, 300));
        }

        // 处理数据
        this.data = allData
          .map(item => ({
            date: new Date(item[0]),
            price: parseFloat(item[4]) // 使用收盘价
          }))
          .sort((a, b) => a.date - b.date); // 确保按时间排序


        // 确保包含最新的数据点
        const lastPoint = allData[allData.length - 1];
        const lastDate = new Date(lastPoint[0]);
        if (this.data[this.data.length - 1].date < lastDate) {
          this.data.push({
            date: lastDate,
            price: parseFloat(lastPoint[4])
          });
        }

        this.startAnimation();
        this.errorMessage = null;
      } catch (error) {
        console.error('获取比特币数据失败:', error);
        this.errorMessage = '比特币价格获取失败，请稍后重试';
      }
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

.area {
  transition: all 0.3s ease;
  pointer-events: none;
}

.date-label {
  position: absolute;
  top: -20px;
  left: 70px;
  font-size: 24px;
  font-weight: bold;
  color: #666;
}

.mini-line {
  transition: all 0.3s ease;
}

.progress-mask {
  transition: all 0.1s ease;
}

.progress-handle {
  cursor: pointer;
  transition: all 0.1s ease;
}

.progress-handle:hover {
  fill: #40a9ff;
}

.current-date {
  font-weight: bold;
  transition: all 0.1s ease;
}

.pulse-circle {
  pointer-events: none;
}

.price-text {
  transition: all 0.3s ease;
}
</style> 