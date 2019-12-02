<template>
  <canvas ref="multipleRelationMap" class="multiple-relation-map"></canvas>
</template>

<script>
  import * as d3 from 'd3';
  import _ from 'lodash';

  const svgIntersections = require('svg-intersections');

  const {intersect, shape} = svgIntersections;

  // 配置项
  const defaultConfig = {
    // 圆形样式
    circleStyle: {
      sizeRange: [], // size 用户可自定义类型圆大小
      // 背景色值域
      bgColor: {
        // 色带定义
        domainField: null, // 作用域对应node中的字段
        domain: [],
        range: {
          // 默认显示色值
          default: [],
          // 选择后的色值
          selected: []
        }
      },
      textField: 'name',
      // 圆中字体样式
      font: {
        size: '12px',
        fontFamily: 'PingFang-SC-Bold',
        scale: 1,
        color: '#ffffff'
      },
      // 控制显示关系路径
      showNodeRelation: {
        triggerOn: 'mouseover',
        enable: true
      }
    },
    // 连接线样式
    linkStyle: {
      distanceRange: [150, 180],
      // 偏移角度基础大小
      offsetAngle: 20,
      // 线条名称
      nameField: null,
      // 连接线作用域
      colorDomain: [],
      // 连接线值域名
      colorRange: [],
      font: {
        size: '12px',
        color: '#888888',
        colorSameLink: false, // 连接线字体颜色是否与连接线颜色一致
      }
    },
    // 隐藏元素的透明度
    hideOpacity: 0.18,
    // 缩放
    zoomStyle: {
      enable: true,
      scaleRange: [0.5, 5],
      zoomInStep: 1.15,
      zoomOutStep: 0.85
    },
    // hover圆后，显示高亮的层次配置
    levelActive: {
      // 当前层级被激活时，级联激活层级的类型
      // SIGLE: 表示激活当前层相邻的层，SIGLE-ROUTE：表示激活相邻层级与当前元素到根路径
      // range: ['SIGLE', 'SIGLE', 'SIGLE-ROUTE', 'SIGLE-ROUTE'],
      range: [],
    },
    toolTips: {
      enable: true, // 是否开启
      triggerOn: 'click', // 触发toolTips的事件类型, click/mouseover
      formatter: '{name}', // 字符串或函数，字符串形如: '公司名称：{object.path}'; 函数返回一个html字符串
      style: {
        className: '',
      },
    },
    interpolateRgb: ['#F2814C', '#129BE7'], // 随机颜色区间
  };

  export default {
    name: 'MultipleRelationMap',
    props: {
      // 图表配置
      options: {
        type: Object,
        default: null
      },
      // 关系数据
      relationData: {
        type: Object,
        default: null
      },
    },
    data() {
      return {
        distanceDomain: null, // 距离的作用域
        nodesData: null, // 节点数据
        linksData: null, // 连线数据
        config: null, // 配置
        levelDomain: [10000, -1], // 层级
      };
    },
    mounted() {
      this.render();
    },
    methods: {
      destroyRelation() {
        // this.canvas && this.context.clearRect(0, 0, this.canvasInfo.width, this.canvasInfo.height);
        this.canvas && d3.select('canvas').remove();
      },
      render() {
        this.destroyRelation();
        this.config = _.merge(_.cloneDeep(defaultConfig), this.options);
        // 合法性检查
        if (_.isEmpty(this.relationData)) {
          return;
        }
        // 数据库格式化
        this.genData();
        // 初始化关系图
        this.init();
      },
      init() {
        // 初始化比例尺
        this.genScale();

        // 获取 2D 上下文对象
        this.canvasInfo = {
          width: this.$refs.multipleRelationMap.clientWidth,
          height: this.$refs.multipleRelationMap.clientHeight
        };
        this.canvas = d3.select(this.$refs.multipleRelationMap)
          .attr('width', `${this.canvasInfo.width}px`)
          .attr('height', `${this.canvasInfo.height}px`);

        // mouse hover事件
        this.canvas.on('mousemove', () => {
          const point = d3.mouse(this.$refs.multipleRelationMap);
          let node = null;
          let minDistance = Infinity;
          const currentScale = d3.zoomTransform(this.canvas.node()).k;
          console.log('currentScale', currentScale);
          this.nodesData.forEach((d) => {
            var dx = d.x - point[0];
            var dy = d.y - point[1];
            var r = this.circleScale(d.type);
            var distance = Math.sqrt((dx * dx) + (dy * dy));
            if (distance < minDistance && distance < r + 1) {
              minDistance = distance;
              node = d;
            }
          });
          // console.log(node);
          this.tickActions(node);
        });

        this.canvas.on('mouseout', () => {
          this.tickActions()
          console.log('mouseout');
        });

        this.context = this.canvas.node().getContext('2d');

        // 创建力学仿真模型
        this.simulation = d3.forceSimulation()
          .nodes(this.nodesData);

        // 新建连线排斥力模型
        this.linkForce = d3.forceLink(this.linksData)
          .distance(10)
          .strength(0.1);

        // 配置模拟器运动参数
        this.simulation
          .force('charge', d3.forceManyBody().strength(-100))
          .force('center', d3.forceCenter(this.canvasInfo.width / 2, this.canvasInfo.height / 2))
          .force('links', this.linkForce);

        // 拖拽
        this.genDrag();
        // 缩放
        this.genZoom();
        // 启动模拟器
        this.simulation.on('tick', this.tickActions);
      },
      // 数据库格式化
      genData() {
        this.nodesData = _.cloneDeep(this.relationData.nodes);
        this.linksData = _.cloneDeep(this.relationData.links);

        // // 记录两两节点之间的连接数
        // const nodeLinkCount = {};
        // // 计算连线
        // this.linksData.forEach(item => {
        //     const source = this.nodesKeyMap[item.source];
        //     const target = this.nodesKeyMap[item.target];
        // });
        // // 色值
        // this.interpolateColor = d3.interpolateRgb(
        //     this.config.interpolateRgb[0],
        //     this.config.interpolateRgb[1]
        // );

        // 计算距离的作用域
        const [min, max] = this.levelDomain;
        this.distanceDomain = [min, max - 1];
      },
      // 初始化比例尺
      genScale() {
        // 圆半径比例尺
        this.circleScale = d3.scaleOrdinal()
          .domain(this.config.circleStyle.sizeDomain)
          .range(this.config.circleStyle.sizeRange);

        // 色值比例尺
        this.circleBgDefaultScale = d3.scaleOrdinal()
          .domain(this.config.circleStyle.bgColor.domain)
          .range(this.config.circleStyle.bgColor.range.default);


        // 连线距离比例尺
        this.distanceScale = d3.scaleLinear()
          .domain(this.distanceDomain)
          .range(this.config.linkStyle.distanceRange);

        // 连接线颜色比例尺
        // 如果colorRange 为空，则使用颜色比例尺
        if (!_.isEmpty(this.config.linkStyle.colorRange)) {
          this.linkColorScale = d3.scaleOrdinal()
            .domain(this.config.linkStyle.colorDomain)
            .range(this.config.linkStyle.colorRange);
        } else {
          this.linkColorScale = val => this.interpolateColor(this.levelScale(val));
        }
      },
      // 自定义主体访问器
      dragSubject() {
        let i,
          x = this.transform.invertX(d3.event.x),
          y = this.transform.invertY(d3.event.y),
          dx,
          dy,
          d2,
          s2;

        for (i = this.nodesData.length - 1; i >= 0; --i) {
          let node = this.nodesData[i];
          dx = x - node.x;
          dy = y - node.y;
          d2 = dx * dx + dy * dy;
          s2 = this.circleScale(this.nodesData[i].type) * this.circleScale(this.nodesData[i].type);

          if (d2 < s2) {
            node.x = this.transform.applyX(node.x);
            node.y = this.transform.applyY(node.y);
            return node;
          }
        }
      },
      // 拖动
      genDrag() {
        // 拖动标识
        let isDraging = false;
        // 拖动最小间隔时间，大于该时间标识拖动
        let dragTimeGap = 80;
        let startTime = null;
        let dragTime = null;
        this.drag = d3.drag()
          .subject(() => this.dragSubject())
          .on('start', () => {
            if (!d3.event.active) {
              // 拖拽开始回调
              this.simulation.alphaTarget(0.1).restart(); //  这个方法可以用在在交互时重新启动仿真，比如拖拽了某个节点，重新进行布局。这个必须要进行设置不然会拖动不了。
            }
            d3.event.sourceEvent.stopPropagation();
            d3.event.subject.fx = this.transform.invertX(d3.event.x);
            d3.event.subject.fy = this.transform.invertY(d3.event.y);
            // 表示未处于拖动
            isDraging = false;
            startTime = new Date().getTime();
          })
          .on('drag', () => {
            dragTime = new Date().getTime();
            // 过滤误操作
            if (!isDraging && dragTime - startTime > dragTimeGap) {
              // 拖拽开始回调
              this.simulation.alphaTarget(0.1).restart(); //  这个方法可以用在在交互时重新启动仿真，比如拖拽了某个节点，重新进行布局。这个必须要进行设置不然会拖动不了。
              // 表示处于拖动
              isDraging = true;
            }
            d3.event.subject.fx = this.transform.invertX(d3.event.x);
            d3.event.subject.fy = this.transform.invertY(d3.event.y);

          })
          .on('end', () => {
            if (isDraging) {
              this.simulation.alphaTarget(0);
              // this.genTick();
            }
            // 表示未处于拖动
            isDraging = false;
            // d3.event.subject.fx = null;
            // d3.event.subject.fy = null;
          });

        this.canvas.call(this.drag);
      },
      // 缩放
      genZoom() {
        // 如果未开启，则返回
        if (!this.config.zoomStyle.enable) {
          return;
        }
        this.zoom = d3.zoom()
          .scaleExtent(this.config.zoomStyle.scaleRange) // 设置可缩放系数大小
          .on('zoom', () => {
            // 监听缩放事件
            this.transform = d3.event.transform;
            this.tickActions();
          });
        this.canvas.call(this.zoom)
          .on('wheel', () => d3.event.preventDefault());
        this.canvas.call(this.zoom.transform, d3.zoomIdentity);
      },
      // 绘制圆
      genCircle(x, y, radius, color) {
        this.context.beginPath();
        this.context.fillStyle = color;
        this.context.strokeStyle = color;
        this.context.arc(x, y, radius, 0, 2 * Math.PI);
        this.context.fill();
        this.context.stroke();
      },
      // 获取连线两点圆的形状
      getCircleShape(d) {
        const result = {};
        const sourceIndex = _.findIndex(this.nodesData, ['name', d.source.name]);
        const targetIndex = _.findIndex(this.nodesData, ['name', d.target.name]);
        if (sourceIndex !== -1 && targetIndex !== -1) {
          const source = this.nodesData[sourceIndex];
          const target = this.nodesData[targetIndex];
          result.sourceShape = {
            cx: source.x,
            cy: source.y,
            r: this.circleScale(source.type)
          };
          result.targetShape = {
            cx: target.x,
            cy: target.y,
            r: this.circleScale(target.type)
          };
        }
        return result;
      },
      // 计算直线连接两个圆，该直线与圆上相交点坐标
      getCenterPoint(d, straightLineD) {
        // const dx = d.target.x - d.source.x;
        // const dy = d.target.y - d.source.y;
        // const dr = Math.sqrt(dx * dx + dy * dy);
        const {sourceShape, targetShape} = this.getCircleShape(d);
        const sourceInter = intersect(
          shape('circle', sourceShape),
          shape('path', {d: straightLineD})
        );
        const targetInter = intersect(
          shape('circle', targetShape),
          shape('path', {d: straightLineD})
        );
        let centerSourcePoint = null;
        let centerTargetPoint = null;
        if (sourceInter.points.length) {
          centerSourcePoint = sourceInter.points[0];
        }
        if (targetInter.points.length) {
          centerTargetPoint = targetInter.points[0];
        }
        return {
          centerSourcePoint,
          centerTargetPoint,
          sourceShape,
          targetShape,
        };
      },
      // 圆内文本换行
      warpText(text, x, y, maxWidth, lineHeight = 14) {
        if (typeof text !== 'string' || typeof x !== 'number' || typeof y !== 'number') {
          return;
        }

        if (typeof maxWidth === 'undefined') {
          maxWidth = (this.canvas && Math.max(this.config.circleStyle.sizeRange) * 2) || 100;
        }

        // 字体行数
        const row = Math.ceil(this.context.measureText(text).width / maxWidth);
        if (row > 1 && row < 4) {
          y = y - lineHeight * row / 2 / 2;
        } else if (row >= 4) {
          y = y - lineHeight * row / 2 / 2 - lineHeight / 2;
        }
        // 字符分隔为数组
        let [arrText, line] = [text.split(''), ''];

        arrText.forEach((item, i) => {
          const testLine = line + item;

          // measureText() 方法返回一个关于被测量文本TextMetrics 对象包含的信息（例如它的宽度）mdn
          const metrics = this.context.measureText(testLine);
          const testWidth = metrics.width;
          if (testWidth > maxWidth && i > 0) {
            this.context.fillText(line, x, y);
            line = item;
            y += lineHeight;
          } else {
            line = testLine;
          }
        });
        this.context.fillText(line, x, y);

      },
      // 绘制圆中的文本
      genCircleText(text, x, y, maxWidth, fontColor, lineHeight) {
        this.context.font = `${this.config.circleStyle.font.size} ${this.config.circleStyle.font.fontFamily}`;
        this.context.fillStyle = fontColor;
        this.context.textAlign = 'center';
        this.context.textBaseline = 'middle';
        this.warpText(text, x, y, maxWidth, lineHeight);
      },
      // 绘制连线
      genLink(d) {
        const {source, target, relation} = d;
        this.context.save();

        this.context.beginPath();
        this.context.moveTo(source.x, source.y);
        this.context.lineTo(target.x, target.y);
        this.context.strokeStyle = this.linkColorScale(relation);
        this.context.stroke();

        this.context.restore();
      },
      // 绘制箭头
      genMatker(d) {
        const {source, target, relation} = d;
        const endRadian = this.getRadian(source.x, source.y, target.x, target.y);
        const offset = this.circleScale(target.type);
        const strokeStyle = this.linkColorScale(relation);
        this.context.save();
        this.context.beginPath();
        this.context.strokeStyle = strokeStyle;
        this.context.translate(target.x, target.y);
        this.context.rotate(endRadian);
        this.context.moveTo(0, offset);
        this.context.lineTo(5, offset + 10);
        this.context.lineTo(-5, offset + 10);
        this.context.closePath();
        this.context.restore();
        this.context.fillStyle = strokeStyle;
        this.context.fill();
      },
      // start point, end point
      getRadian(x0, y0, x1, y1) {
        let endRadian = Math.atan((y1 - y0) / (x1 - x0));
        endRadian += ((x1 >= x0 ? 90 : -90) * Math.PI) / 180;
        return endRadian;
      },
      genLinkText(d) {
        const {source, target, relation} = d;
        const endRadian = this.getRadian(source.x, source.y, target.x, target.y);
        const strokeStyle = this.linkColorScale(relation);
        this.context.save();

        this.context.beginPath();
        this.context.translate(target.x, target.y);
        this.context.rotate(endRadian - Math.PI * 0.5);
        this.context.fillStyle = strokeStyle;
        this.context.textAlign = 'center';
        this.context.textBaseline = 'bottom';
        this.context.fillText(relation, -100, 0);
        this.context.closePath();

        this.context.restore();

        this.context.fillStyle = strokeStyle;
        this.context.fill();
      },
      // sim tick action
      tickActions(node) {
        // canvas 状态保存
        this.context.save();
        this.context.clearRect(0, 0, this.canvasInfo.width, this.canvasInfo.height);

        this.context.translate(this.transform.x, this.transform.y);
        this.context.scale(this.transform.k, this.transform.k);


        // 连线/箭头、箭头上的文字
        this.linksData.forEach(d => {
          // console.log(this.getPathD(d));
          this.genLink(d);
          this.genMatker(d);
          this.genLinkText(d);
        });

        // 圆
        this.nodesData.forEach(d => {
          // mousemove时，传入了指定的node时，高亮显示对应node并置灰其他node；否则默认全部高亮
          let nodeBgColor = node && d !== node ? "#e4e5e5" : this.circleBgDefaultScale(d[this.config.circleStyle.bgColor.domainField]);
          let nodeTextColor = node && d !== node ? "#eee" : this.config.circleStyle.font.color;

          this.genCircle(d.x, d.y, this.circleScale(d.type), nodeBgColor);
          this.genCircleText(d.name, d.x, d.y, this.circleScale(d.type) * 2 - 10, nodeTextColor);
        });

        // canvas 状态恢复
        this.context.restore();
      },
      // 放大
      zoomIn() {
        if (!this.config.zoomStyle.enable) {
          return;
        }
        this.zoom.scaleBy(this.canvas, this.config.zoomStyle.zoomInStep);
      },
      // 缩小
      zoomOut() {
        if (!this.config.zoomStyle.enable) {
          return;
        }
        this.zoom.scaleBy(this.canvas, this.config.zoomStyle.zoomOutStep);
      }
    },
  };
</script>

<style>
  .multiple-relation-map {
    width: 100%;
    height: 100%;
  }
</style>
