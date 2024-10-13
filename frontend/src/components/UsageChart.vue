<template>
    <div>
        <v-chart class="chart" :option="chartOption" />
    </div>
</template>

<script>
import { defineComponent, ref, onMounted } from 'vue';
import axios from 'axios';
import VChart from 'vue-echarts'; // VChart` 组件用于在 Vue.js 应用程序中渲染聊天界面
import { use } from 'echarts/core'; // 这允许在整个应用程序中使用E - Charts功能。
import { CanvasRenderer } from 'echarts/renderers';  // Canvas Renderer负责使用HTML5 Canvas元素呈现E - Charts图表 
import { BarChart } from 'echarts/charts';
import { GridComponent, TooltipComponent, LegendComponent } from 'echarts/components';

use([CanvasRenderer, BarChart, GridComponent, TooltipComponent, LegendComponent]);

// 基于浏览器的主题设置图例字体颜色
function getLegendTextColor() {
    return window.matchMedia && window.matchMedia('(prefers-color-scheme: dark)').matches ? '#FFFFFF' : '#000000';
}

export default defineComponent({
    name: 'UsageChart',
    components: {
        VChart
    },
    setup() {
        const chartOption = ref({}); // 反应性数据属性，保存使用图表的图表选项。该属性用于配置图表的外观和行为,显示使用数据。
        // const data = { currentPage: 1, pageSize: 100 };
        const fetchData = async () => {
            try {
                const response = await axios.post("https://api.chatanywhere.tech/v1/query/usage",
                    { currentPage: 1, pageSize: 999999 },
                    {
                        headers: {
                            "Authorization": 'sk-iamvpLmgpG7cpdLcgxSArODwWHBvxmN3F9prUF39mGXBk1Wh',
                        }
                    }
                );
                const usageLogs = response.data.usageLogs;
                const models = [... new Set(usageLogs.map(log => log.model))]; // *从 `usageLogs` 数组中提取一组唯一的模型名称。这对于生成应用程序中使用的不同模型的列表非常有用。
                const dates = [...new Set(usageLogs.map(log => new Date(log.createTime).toLocaleDateString()))]
                    .sort((a, b) => new Date(a) - new Date(b)); // *从 `usageLogs` 数组中提取一组唯一的日期。这对于生成应用程序中使用的不同日期的列表非常有用。

                const series = models.map(model => ({
                    name: model,
                    type: 'bar',
                    stack: 'total',
                    /**
                    *计算 `usageLogs` 数组中每个日期和模型组合的总标记。
                    *该函数用于生成 `UsageChart` 组件中条形图的数据。
                    *
                    *@param {string[]} 日期 -从 `usageLogs` 数组中提取的唯一日期数组。
                    *@param {string[]} models -从 `usageLogs` 数组中提取的唯一模型数组。
                    *@param {Object[]} useLogs -使用日志对象数组，每个对象包含有关特定使用事件的信息。
                    *@returns {number[]} -每个日期和模型组合的总标记计数数组。
                     */
                    data: dates.map(date => {
                        const logsForData = usageLogs.filter(log =>
                            new Date(log.createTime).toLocaleDateString() === date && log.model === model
                        );

                        return logsForData.reduce((total, log) => total + log.cost, 0);
                    })
                }));

                chartOption.value = {
                    tooltip: {
                        formatter: function (params) {
                            let result = '<div style="text-align: left;">';
                            result += params[0].axisValueLabel + '<br/>';
                            params.forEach(param => {
                                // 使用 toFixed(2) 来控制小数位数
                                let value = typeof param.value === 'number' ? param.value.toFixed(2) : param.value;
                                result += param.marker + param.seriesName + ': ' + value + '元<br/>';
                            });
                            return result;
                        },
                        align: 'left',
                        trigger: 'axis',
                        axisPointer: {
                            type: 'shadow'
                        }
                    },
                    legend: {
                        textStyle: {
                            color: getLegendTextColor() // 动态设置图例字体颜色
                        }
                    },
                    grid: {
                        left: '3%',
                        right: '4%',
                        bottom: '3%',
                        containLabel: true
                    },
                    xAxis: {
                        type: 'category',
                        data: dates
                    },
                    yAxis: {
                        axisLabel: {
                            formatter: function (value) {
                                return value.toFixed(0);
                            }
                        },
                        type: 'value'
                    },
                    series: series
                }
            } catch (error) {
                console.error('Error fetching data:', error);
            }
        };

        onMounted(fetchData);

        return {
            chartOption
        };

    }
},

)
</script>

<style scoped>
.chart {
    height: 400px;
}
</style>