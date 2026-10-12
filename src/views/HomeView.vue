<script setup>
import LineChart from '@/components/LineChart/LineChart.vue'

const testData = {
  type: 'ChartTemplate',
  id: 'salesChart2',
  isHide: false,
  margin: { top: 8, bottom: 8 },
  component: {
    chart: {
      breakoutOffset: { left: -12, right: -8 },
      yLabelSuffix: '$',
      pointTitleSuffix: '$',
      xLabelDate: {
        type: 'Text', content: '', typography: 'badge', color: 'contrastPrimary',
      },
      pointTitle: {
        type: 'Text', content: '', typography: 'badge', color: 'contrastPrimary',
      },
      xLabelDescription: {
        type: 'Text', content: '', typography: 'caption', color: 'contrastPrimary',
      },
      values: [{
        xValue: '26.11.2020', yValue: 12999, state: 'default', description: '1-я публикация на ',
      }, {
        xValue: '25.12.2020', yValue: 12050, state: 'default', description: '2-я публикация на ',
      },

        {
          xValue: '25.12.2020', yValue: 12150, state: 'default', description: '2-я публикация на ',
        },
        {
          xValue: '25.12.2020', yValue: 12250, state: 'default', description: '2-я публикация на ',
        },
        {
          xValue: '25.12.2020', yValue: 12350, state: 'default', description: '2-я публикация на ',
        },
        {
          xValue: '25.12.2020', yValue: 12450, state: 'default', description: '2-я публикация на ',
        },
        {
          xValue: '25.12.2020', yValue: 11050, state: 'default', description: '2-я публикация на ',
        },
        {
          xValue: '25.12.2020', yValue: 11950, state: 'default', description: '2-я публикация на ',
        },
        {
          xValue: '25.12.2020', yValue: 11850, state: 'default', description: '2-я публикация на ',
        },
        {
          xValue: '25.12.2020', yValue: 11750, state: 'default', description: '2-я публикация на ',
        },
        {
          xValue: '25.12.2020', yValue: 11650, state: 'default', description: '2-я публикация на ',
        },
        {
          xValue: '25.12.2020', yValue: 11550, state: 'default', description: '2-я публикация на ',
        },
        {
          xValue: '25.12.2020', yValue: 12250, state: 'default', description: '2-я публикация на ',
        },
        {
          xValue: '25.12.2020', yValue: 12350, state: 'default', description: '2-я публикация на ',
        },
        {
          xValue: '25.12.2020', yValue: 12450, state: 'default', description: '2-я публикация на ',
        },
        {
          xValue: '25.12.2020', yValue: 11050, state: 'default', description: '2-я публикация на ',
        },
        {
          xValue: '25.12.2020', yValue: 11950, state: 'default', description: '2-я публикация на ',
        },
        {
          xValue: '25.12.2020', yValue: 11850, state: 'default', description: '2-я публикация на ',
        },
        {
          xValue: '25.12.2020', yValue: 11750, state: 'default', description: '2-я публикация на ',
        },
      ],
    },
  },
  action: 'logAnalytic',
  analytic: {},
};
</script>

<template>
  <main>
    <LineChart :template-data="testData" />
  </main>
</template>
