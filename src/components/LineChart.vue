<template>
    <Line :data="data" :options="options" />
</template>
  
<script lang="ts">
import {
    Chart as ChartJS,
    CategoryScale,
    LinearScale,
    PointElement,
    LineElement,
    Title,
    Tooltip,
    Legend
} from 'chart.js'
import { Line } from 'vue-chartjs'

ChartJS.register(
    CategoryScale,
    LinearScale,
    PointElement,
    LineElement,
    Title,
    Tooltip,
    Legend
)

export default {
    props: {
        chartData: {
            type: Object,
            required: true
        },
    },
    components: {
        Line
    },
    data() {
        return {
            data: {
                labels: [],
                datasets: [
                    {
                        data: [],
                        borderColor: ['#FF6384', '#36A2EB', '#FFCE56', '#42b983', '#3f51b5', '#e91e63', '#00bcd4', '#8bc34a', '#ff9800', '#9c27b0', '#4caf50', '#2196f3', '#ffc107'][Math.floor(13 * Math.random())], 
                    }
                ]
            },
            options: {
                responsive: true,
                maintainAspectRatio: false,
                plugins: {
                    legend: {
                        display: false
                    }
                },
                scales: {
                    x: {
                        ticks: {
                            // Include a dollar sign in the ticks
                            callback: function () {
                                return '';
                            }
                        }
                    }
                }
            }
        }
    },
    mounted() {
        this.data = this.chartData;
    }
}
</script>