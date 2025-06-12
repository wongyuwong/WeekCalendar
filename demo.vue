<template>
  <view>
    <WeekCalendar 
        yearMonth="2025-06"
        theme="blue"
        @backevent="weekCalendarChange"
        @monthChange="onMonthChange"
      />
  </view>
</template>

<script lang="ts" setup>

// 引入周选择日历组件
import WeekCalendar from './WeekCalendar.vue'

// 组件注册
defineOptions({
  components: {
    WeekCalendar
  }
})

const onMonthChange = (e: any) => {
  console.log('月份变化:', e)
}
const weekCalendarChange = (e: any) => {
  console.log('周选择器选择:', e)
}

</script>

<style lang="scss" scoped>

</style>
