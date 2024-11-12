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
import bitcoinEvents from '../assets/bitcoin-events.json';

export default {
  name: 'BitcoinChart',
  data() {
    const CHART_CONFIG = {
      width: 1200,
      height: 400,
      margin: { top: 20, right: 120, bottom: 100, left: 50 },
      eventAreaHeight: 100,
      priceAreaHeight: 400,
      displayPoints: 200,
      animationDuration: 250000,
    };

    return {
      config: CHART_CONFIG,
      svg: null,
      line: null,
      xScale: null,
      yScale: null,
      data: [],
      priceData: [],
      animationTimer: null,
      currentPrice: null,
      errorMessage: null,
      currentDate: null,
      currentIndex: 1,
      events: [],
      visibleEvents: [],
    };
  },
  mounted() {
    this.initChart();
    this.events = bitcoinEvents.events.map(event => ({
      ...event,
      date: new Date(event.date),
    }));
    this.fetchBitcoinData();
  },
  beforeUnmount() {
    if (this.animationTimer) {
      clearInterval(this.animationTimer);
    }
  },
  methods: {
    initChart() {
      const totalHeight = this.config.eventAreaHeight + this.config.priceAreaHeight + this.config.eventAreaHeight + 150;
      
      const svg = d3.select('#chart')
        .append('svg')
        .attr('width', this.config.width + this.config.margin.left + this.config.margin.right)
        .attr('height', totalHeight + this.config.margin.top + this.config.margin.bottom)
        .append('g')
        .attr('transform', `translate(${this.config.margin.left},${this.config.margin.top + this.config.eventAreaHeight})`);

      this.svg = svg;

      // 初始化比例尺
      this.xScale = d3.scaleTime()
        .range([0, this.config.width]);

      this.yScale = d3.scaleLinear()
        .range([this.config.priceAreaHeight, 0]);

      // 创建坐标轴
      this.xAxis = svg.append('g')
        .attr('class', 'x-axis')
        .attr('transform', `translate(0,${this.config.priceAreaHeight + this.config.eventAreaHeight})`);

      this.yAxis = svg.append('g')
        .attr('class', 'y-axis')
        .attr('transform', 'translate(0,0)');

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
        .attr('d', `M0.5,0.5H${this.config.width + 0.5}`); // 使用 SVG path 命令画一条完整的直线

      // 添加略缩图容器
      this.miniContainer = svg.append('g')
        .attr('class', 'mini-chart')
        .attr('transform', `translate(0,${this.config.priceAreaHeight + this.config.eventAreaHeight + 80})`);

      // 添加略缩图背景
      this.miniContainer.append('rect')
        .attr('class', 'mini-background')
        .attr('x', 0)
        .attr('y', 0)
        .attr('width', this.config.width)
        .attr('height', 40)
        .attr('fill', '#F5F5F5') // 使用白色加一点点灰色
        .attr('rx', 4);

      // 创建略缩图比例尺
      this.miniXScale = d3.scaleTime()
        .range([0, this.config.width]);

      this.miniYScale = d3.scaleLinear()
        .range([40, 0]);

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
        .attr('height', 40)
        .attr('fill', '#e6f3ff')
        .attr('opacity', 0.7);

      // 添加左侧柄
      this.leftHandle = this.miniContainer.append('rect')
        .attr('class', 'progress-handle left-handle')
        .attr('x', -2) // 向左偏移2px以对齐略缩图边缘
        .attr('y', -5) // 向上延伸5px
        .attr('width', 4)
        .attr('height', 50) // 增加高度使其延伸超出略缩图
        .attr('fill', '#1890ff')
        .attr('rx', 2);

      // 添加右侧柄
      this.rightHandle = this.miniContainer.append('rect')
        .attr('class', 'progress-handle right-handle')
        .attr('x', -2) // 初始位置向左偏移2px
        .attr('y', -5) // 向上延伸5px
        .attr('width', 4)
        .attr('height', 50) // 增加高度使其延伸超出略缩图
        .attr('fill', '#1890ff')
        .attr('rx', 2);

      // 添加左侧日期文本
      this.startDateText = this.miniContainer.append('text')
        .attr('class', 'date-label start-date')
        .attr('x', 0)
        .attr('y', 40 + 20)
        .attr('text-anchor', 'start')
        .attr('fill', '#666')
        .attr('font-size', '15px');

      // 添加右侧日期文本
      this.endDateText = this.miniContainer.append('text')
        .attr('class', 'date-label end-date')
        .attr('x', this.config.width)
        .attr('y', 40 + 20)
        .attr('text-anchor', 'end')
        .attr('fill', '#666')
        .attr('font-size', '15px');

      // 添加当前进度日期文本
      this.currentDateText = this.miniContainer.append('text')
        .attr('class', 'date-label current-date')
        .attr('y', 40 + 20)
        .attr('text-anchor', 'middle')
        .attr('fill', '#666')  // 使用和柄相同的颜色
        .attr('font-size', '15px');

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

      // 修路径元素，使用渐变色和阴影
      this.path = svg.append('path')
        .attr('class', 'line')
        .attr('fill', 'none')
        .attr('stroke', 'url(#line-gradient)')
        .attr('stroke-width', 3)
        .attr('filter', 'url(#glow)');

      // 创建面积生成器用于阴影效果
      this.area = d3.area()
        .x(d => this.xScale(new Date(d.date)))
        .y0(this.config.height)
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
        .attr("y2", this.config.height);

      areaGradient.append("stop")
        .attr("offset", "0%")
        .attr("stop-color", "#1890ff")  // 使用相同的蓝色
        .attr("stop-opacity", 0.2);     // 降低透明

      areaGradient.append("stop")
        .attr("offset", "100%")
        .attr("stop-color", "#1890ff")
        .attr("stop-opacity", 0);

      // 添加面积路径
      this.areaPath = svg.append('path')
        .attr('class', 'area')
        .attr('fill', 'url(#area-gradient)');

      // 创建点的发散动画效
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

      // 添加裁剪区域定义
      this.clipPath = svg.append("defs")
        .append("clipPath")
        .attr("id", "chart-area")
        .append("rect")
        .attr("x", 0)
        .attr("y", -this.config.eventAreaHeight)
        .attr("width", this.config.width)
        .attr("height", this.config.priceAreaHeight + this.config.eventAreaHeight * 2);

      // 创建一个包含事件容器的组，并应用裁剪
      const eventClipGroup = svg.append('g')
        .attr('clip-path', 'url(#chart-area)');

      // 将事件容器移到裁剪组内
      this.topEventContainer = eventClipGroup.append('g')
        .attr('class', 'top-event-container')
        .attr('transform', `translate(0,${-this.config.eventAreaHeight})`);

      this.bottomEventContainer = eventClipGroup.append('g')
        .attr('class', 'bottom-event-container')
        .attr('transform', `translate(0,${this.config.priceAreaHeight})`);
    },

    updateChart() {
      if (this.priceData.length === 0) return;

      // 获取要显示的数据
      let displayData;
      if (this.priceData.length >= this.config.displayPoints) {
        // 如果数据超过300个点，取最后300个
        displayData = this.priceData.slice(-this.config.displayPoints);
        this.priceData = this.priceData.slice(-this.config.displayPoints);
      } else {
        // 如果数据不足300个，则取this.data的前300个数据点
        displayData = this.data.slice(0, this.config.displayPoints);
      }

      // 修改Y轴范围，使用 priceData 计算
      const yMin = d3.min(displayData, d => d.price);
      const yMax = d3.max(displayData, d => d.price);
      const yPadding = (yMax - yMin) * 0.2; // 添加20%的padding使图表不会太贴边
      this.yScale.domain([yMin - yPadding, yMax + yPadding]);

      // 更新Y轴
      this.yAxis.call(
        d3.axisLeft(this.yScale)
          .ticks(5)  // 设置刻度数量为 5
          .tickFormat(d => `$${d.toLocaleString()}`)
      )
      .call(g => {
        g.selectAll('.tick line').remove();  // 移除刻度线
        g.select('.domain').remove();        // 移除主轴线
        g.selectAll('.tick text')            // 设置文本样式
          .attr('x', -10)
          .style('text-anchor', 'end')
          .style('display', 'block');
      });

      // 更新X轴范围
      const xMin = d3.min(displayData, d => new Date(d.date));
      const xMax = d3.max(displayData, d => new Date(d.date));
      this.xScale.domain([xMin, xMax]);

      // 更新坐标轴，增加刻度数量
      this.xAxis.call(
        d3.axisBottom(this.xScale)
          .ticks(6) // 增加刻度数量从10到15
          .tickFormat(d3.timeFormat("%Y/%m/%d"))
      );

      // 更新路径数据
      this.path
        .datum(this.priceData)
        .attr('d', this.line);

      // 更新面积
      this.areaPath
        .datum(this.priceData)
        .attr('d', this.area);

      // 在最后一个数据点加一个圆点，并在圆点右侧展示格
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
        .style('font-size', '20px')
        .text(`$${lastDataPoint.price.toFixed(2)}`);

      // 更新当前价格
      this.currentPrice = lastDataPoint?.price;
      this.currentDate = lastDataPoint ? new Date(lastDataPoint.date).toLocaleDateString('zh-CN') : null;

      // 重新应用 x 轴主轴线的样式
      this.xAxis.select('path.domain')
        .attr('d', `M0.5,0.5H${this.config.width + 0.5}`);

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
      const maskWidth = this.config.width * (this.currentIndex / this.data.length);
      this.progressMask
        .attr('x', 0)
        .attr('width', maskWidth);

      // 更新左侧柄位置（固定在略缩图左侧边缘）
      this.leftHandle
        .attr('x', -2); // 向左偏移2px以对齐略缩图边缘

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

      // 更当前日期文本位置和内容
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
          currentX + textWidth/2 > this.config.width - textWidth - margin ? 'none' : 'block');
      }

      // 更新渐变的范围
      d3.select("#line-gradient")
        .attr("y1", this.yScale(d3.min(this.priceData, d => d.price)))
        .attr("y2", this.yScale(d3.max(this.priceData, d => d.price)));

      // 更新裁剪区域的范围
      const clipX = this.xScale(new Date(this.priceData[0].date));
      const clipWidth = this.xScale(new Date(this.priceData[this.priceData.length - 1].date)) - clipX;

      this.clipPath
        .attr("x", clipX)
        .attr("width", clipWidth);
      
      // 在所有图表元素更新后再更新事件标记
      this.addEventMarkers();
    },

    startAnimation() {
      this.currentIndex = 1;
      const animationDuration = this.config.animationDuration; // 总动画时长（毫秒）
      const totalPoints = this.data.length;
      const animationSpeed = Math.floor(animationDuration / totalPoints); // 每点的动画间隔

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
    },

    // 添加事件标记的方法
    addEventMarkers() {
      // 清除现有事件标记
      this.topEventContainer.selectAll('*').remove();
      this.bottomEventContainer.selectAll('*').remove();

      // 获取当前最新价格数据点的日期
      const currentDate = this.priceData[this.priceData.length - 1]?.date;
      if (!currentDate) return;

      // 计算未来60天的日期
      const futureDate = new Date(currentDate);
      futureDate.setDate(futureDate.getDate() + 60);

      // 获取当前可见的事件：在当前日期之后且在未来60天内的事件
      this.visibleEvents = this.events.filter(event => {
        const pastDate = new Date(currentDate);
        pastDate.setDate(pastDate.getDate() - this.config.displayPoints - 120);
        return event.date > pastDate && event.date <= futureDate;
      });

      // 其余代码保持不变
      const topEvents = this.visibleEvents.filter(event => event.type === 'positive');
      const bottomEvents = this.visibleEvents.filter(event => event.type === 'negative');

      const addEventGroup = (events, container, isTop) => {
        // 用于存储已放置事件的位置信息
        const placedEvents = [];
        
        events.forEach((event) => {
          const x = this.xScale(event.date);
          let baseY = isTop ? this.config.eventAreaHeight / 2 : this.config.eventAreaHeight / 2;
          
          // 创建文本元素以获取其尺寸
          const tempText = container.append('text')
            .attr('font-size', '14px')
            .text(event.text);
          const bbox = tempText.node().getBBox();
          tempText.remove();

          // 计算事件标记的宽度和高度（包含padding）
          const padding = 8;
          const eventWidth = bbox.width + padding * 2;
          const eventHeight = bbox.height + padding * 2;
          
          // 检查重叠并调整y坐标
          let yOffset = 0;
          let overlap = true;
          const maxAttempts = 10; // 最大尝试次数
          let attempts = 0;
          
          while (overlap && attempts < maxAttempts) {
            overlap = placedEvents.some(placed => {
              const xOverlap = Math.abs(placed.x - x) < (eventWidth + 20); // 添加20px的额外间距
              const yOverlap = Math.abs((baseY + yOffset) - placed.y) < (eventHeight + 5);
              return xOverlap && yOverlap;
            });

            if (overlap) {
              // 根据是顶部还是底部事件调整偏移方向
              yOffset += isTop ? eventHeight : -eventHeight;
              attempts++;
            }
          }

          // 记录已放置的事件位置
          placedEvents.push({
            x: x,
            y: baseY + yOffset,
            width: eventWidth,
            height: eventHeight
          });

          const y = baseY + yOffset;

          // 添加事件标记组
          const eventGroup = container.append('g')
            .attr('class', 'event-group')
            .style('opacity', 1)
            .attr('transform', `translate(${x},${y})`);

          const text = event.text;
          
          // 先创建文本元素以获取其尺寸
          const textElement = eventGroup.append('text')
            .attr('x', 0)
            .attr('y', 0)
            .attr('text-anchor', 'middle')
            .attr('dominant-baseline', 'middle') // 添加垂直居中对齐
            .attr('fill', '#333')
            .attr('font-size', '14px')
            .text(text);
          
          // 获取文本的边界框
          const bbbox = textElement.node().getBBox();

          // 添加背景框，调整定位计算
          eventGroup.insert('rect', 'text')
            .attr('x', -bbbox.width/2 - padding) // 居中对齐
            .attr('y', -bbbox.height/2 - padding) // 垂直居中对齐
            .attr('width', bbbox.width + padding * 2)
            .attr('height', bbbox.height + padding * 2)
            .attr('rx', 4)
            .attr('fill', 'white')
            .attr('fill-opacity', 0.9)
            .attr('stroke', event.type === 'positive' ? '#52c41a' : '#ff4d4f')
            .attr('stroke-width', 1);
        
          // 计算价格点的y坐标
          const pricePoint = this.data.find(d => d.date.getTime() === event.date.getTime());
          let priceY = 0;
          
          if (pricePoint) {
            if (isTop) {
              // 对于顶部事件，连接到价格点
              priceY = this.yScale(pricePoint.price) + this.config.eventAreaHeight;
            } else {
              // 对于底部事件，连接到价格点
              priceY = this.config.priceAreaHeight - this.yScale(pricePoint.price);
            }
          }
          
          // 修改连接线的路径，考虑偏移后的位置
          const lineStart = isTop ? 
            [x, y + (eventHeight / 2)] : 
            [x, y - (eventHeight / 2)];
          
          const lineEnd = isTop ? [x, priceY] : [x, -priceY];

          // 添加连接线的渐变定义
          const gradientId = `line-gradient-${event.date.getTime()}`;
          container.append("defs")
            .append("linearGradient")
            .attr("id", gradientId)
            .attr("gradientUnits", "userSpaceOnUse")
            .attr("x1", 0)
            .attr("y1", lineStart[1])
            .attr("x2", 0)
            .attr("y2", lineEnd[1])
            .selectAll("stop")
            .data([
              { offset: "0%", color: event.type === 'positive' ? '#52c41a' : '#ff4d4f', opacity: 0.8 },
              { offset: "100%", color: event.type === 'positive' ? '#52c41a' : '#ff4d4f', opacity: 0.2 }
            ])
            .enter().append("stop")
            .attr("offset", d => d.offset)
            .attr("stop-color", d => d.color)
            .attr("stop-opacity", d => d.opacity);

          // 修改连接线样式
          container.append('path')
            .attr('class', 'connector-line')
            .attr('d', d3.line()([lineStart, lineEnd]))
            .attr('stroke', `url(#${gradientId})`)
            .attr('stroke-width', 1.5) // 增加线条宽度
            .attr('fill', 'none')
            .style('opacity', 1); // 增加不透明度
            // 移除 stroke-dasharray

          textElement.raise();
        });
      };

      // 分别添加上下两组事件
      addEventGroup(topEvents, this.topEventContainer, true);
      addEventGroup(bottomEvents, this.bottomEventContainer, false);
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
  /* 添加水平滚动支持 */
  overflow-x: auto;
  /* 确保容器能完整显示更宽的图表 */
  min-width: 1200px;
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
  top: 10px;
  left: 70px;
  font-size: 24px;
  font-weight: bold;
  color: #666;
  background-color: white;
  padding: 5px 10px;
  z-index: 1000;
  border-radius: 4px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
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
  filter: drop-shadow(0 0 2px rgba(24, 144, 255, 0.5));
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

.event-marker {
  cursor: pointer;
  transition: all 0.3s ease;
}

.event-marker:hover {
  transform: scale(1.5);
}

.event-group {
  transition: all 0.3s ease;
}

.event-group text {
  font-size: 14px;
  fill: #333;
}

/* 添加阴影滤镜 */
.bitcoin-chart >>> #shadow {
  filter: drop-shadow(0 2px 4px rgba(0,0,0,0.1));
}

.connector-line {
  transition: all 0.3s ease;
  pointer-events: none;
}
</style> 