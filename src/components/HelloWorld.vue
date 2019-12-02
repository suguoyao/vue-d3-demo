<template>
  <div class="hello">
    <multiple-relation-map :relation-data="mockData" :options="options"/>
  </div>
</template>

<script>
  import MultipleRelationMap from './MultipleRelationMap'

  const mockData = require('../assets/multiple-relation-data')

  export default {
    name: 'HelloWorld',
    components: {MultipleRelationMap},
    data() {
      return {
        mockData,
        options: {
          circleStyle: {
            sizeDomain: ["root", "enterprise", "person"],
            sizeRange: [50, 40, 30],
            // 圆形样式
            bgColor: {
              domainField: "type", // 作用域对应node中的字段
              domain: ["root", "enterprise", "person"], // 为空默认使用levelDomain,此时颜色将按照不同层区分
              range: {
                // 默认显示色值
                default: ["#F2814C", "#129BE7", "#16B4C3"],
                // 选择后的色值
                selected: ["#D97344", "#108BCF", "#13A1AF"]
              }
            },
          },
          linkStyle: {
            // 线条名称
            nameField: 'name',
            // 偏移角度基础大小, 度为单位
            offsetAngle: 20,
            // 连接线作用域
            colorDomain: ['高管关系', '投资', '法人'],
            // 连接线职域名
            colorRange: ['#B0BCED', '#F8C0A5', '#97E3C1'],
            font: {
              size: '12px',
              color: '#888888',
              colorSameLink: false, // 连接线字体颜色是否与连接线颜色一致
            },
          },
          toolTips: {
            formatter: node => {
              return `<h3>${node.name}</h3>`;
            }
          },
          levelActive: {
            range: ['SIGLE', 'SIGLE', 'SIGLE-ROUTE', 'SIGLE-ROUTE'],
          }
        }
      }
    },
    created() {
      console.log(this.mockData)
    }
  }
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>

</style>
