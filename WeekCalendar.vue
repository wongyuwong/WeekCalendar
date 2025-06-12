<template>
  <div class="calendar-box" :class="[theme || '', line || '', size || '']">
    <div v-if="list.length === 0" class="tip">加载中</div>
    <div v-else class="wrap">
      <div class="month-navigation">
        <div class="month-nav-btn" @click="prevMonth">
          <text class="nav-icon">〈</text>
        </div>
        <div class="month-display">
          {{ displayYearMonth }}
        </div>
        <div class="month-nav-btn" @click="nextMonth">
          <text class="nav-icon">〉</text>
        </div>
      </div>
      
      <div class="week header">
        <div class="day">一</div>
        <div class="day">二</div>
        <div class="day">三</div>
        <div class="day">四</div>
        <div class="day">五</div>
        <div class="day saturday">六</div>
        <div class="day sunday">日</div>
      </div>
      <div 
        v-for="(week, index) in list" 
        :key="index" 
        class="week-box" 
        :class="{ 'on': rangeBg[index] }"
      >
        <div 
          class="week" 
          :class="{ 'select': index === selectLine }" 
          @click="weekTap(index)"
        >
          <div 
            v-for="(item, idx) in week" 
            :key="idx" 
            class="day" 
            :class="{ 
              'active': item.isToday, 
              'saturday': idx === 5, 
              'sunday': idx === 6 
            }"
          >
            <span :class="{ 'grey': !item.isCurrentMonth }">{{ item.title }}</span>
          </div>
        </div>
      </div>
      <div 
        class="highlight" 
        :style="{ 
          display: highlightStyle.display, 
          top: highlightStyle.top + 'px', 
          height: highlightStyle.height + 'px' 
        }"
      ></div>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, reactive, computed, watch, onMounted } from 'vue';
import type { PropType } from 'vue';
import moment from 'moment';

interface DayItem {
  title: string;
  value: string;
  isCurrentMonth: boolean;
  isToday: boolean;
}

interface HighlightStyle {
  display: string;
  top: number;
  height: number;
}

export default defineComponent({
  name: 'WeekCalendar',
  props: {
    yearMonth: {
      type: String,
      default: '',
    },
    year: {
      type: String,
      default: '',
    },
    month: {
      type: String,
      default: '',
    },
    range: {
      type: Array as PropType<string[]>,
      default: () => [],
    },
    theme: {
      type: String,
      default: '',
    },
    /* 是否有外部边框 line: border */
    line: {
      type: String,
      default: '',
    },
    /*  尺寸 size: big || small */
    size: {
      type: String,
      default: '',
    },
  },
  emits: ['backevent', 'monthChange'],
  setup(props, { emit }) {
    const load = ref(true);
    const that_year = ref('');
    const that_month = ref('');
    const today = ref<moment.Moment | null>(null);
    const list = ref<DayItem[][]>([]);
    const selectLine = ref(0);
    const rangeBg = ref<boolean[]>([]);
    const highlightStyle = reactive<HighlightStyle>({
      display: 'none',
      top: 0,
      height: 0,
    });
    
    // 添加月份显示计算属性
    const displayYearMonth = computed(() => {
      return `${that_year.value}年${that_month.value}月`;
    });
    
    // 上个月导航
    const prevMonth = () => {
      try {
        let year = parseInt(that_year.value || '0');
        let month = parseInt(that_month.value || '0');
        
        if (year === 0 || month === 0) {
          console.error('月份切换失败：年份或月份无效');
          return;
        }
        
        month--;
        if (month < 1) {
          month = 12;
          year--;
        }
        
        // 更新年月
        that_year.value = year.toString();
        that_month.value = month.toString().padStart(2, '0');
        
        // 通知父组件月份变化
        emit('monthChange', { year: that_year.value, month: that_month.value });
        
        // 使用setTimeout避免立即初始化导致的渲染问题
        setTimeout(() => {
          // 重新初始化日历
          init();
        }, 10);
      } catch (error) {
        console.error('切换上个月出错:', error);
      }
    };
    
    // 下个月导航
    const nextMonth = () => {
      try {
        let year = parseInt(that_year.value || '0');
        let month = parseInt(that_month.value || '0');
        
        if (year === 0 || month === 0) {
          console.error('月份切换失败：年份或月份无效');
          return;
        }
        
        month++;
        if (month > 12) {
          month = 1;
          year++;
        }
        
        // 更新年月
        that_year.value = year.toString();
        that_month.value = month.toString().padStart(2, '0');
        
        // 通知父组件月份变化
        emit('monthChange', { year: that_year.value, month: that_month.value });
        
        // 使用setTimeout避免立即初始化导致的渲染问题
        setTimeout(() => {
          // 重新初始化日历
          init();
        }, 10);
      } catch (error) {
        console.error('切换下个月出错:', error);
      }
    };
    
    // 监听yearMonth变化
    watch(() => props.yearMonth, (newVal, oldVal) => {
      try {
        if (!newVal) return;
        
        if (oldVal && newVal) {
          try {
            const n = moment(newVal).diff(moment(oldVal), 'month');
            if (n > 0) {
              selectLine.value = 0;
            } else if (n < 0) {
              // 设置为10后续渲染会自动选中当月最大周
              selectLine.value = 10;
            }
          } catch (error) {
            console.error('计算月份差异出错:', error);
          }
        }
        
        const parts = newVal.split('-');
        if (parts.length !== 2) {
          console.error('yearMonth格式错误:', newVal);
          return;
        }
        
        const [year, month] = parts;
        if (!year || !month) {
          console.error('yearMonth解析失败:', newVal);
          return;
        }
        
        that_year.value = year;
        that_month.value = month;
        
        // 使用setTimeout避免渲染问题
        setTimeout(() => {
          // 每次月份变化都需要重新初始化
          init();
        }, 10);
      } catch (error) {
        console.error('监听yearMonth变化出错:', error);
      }
    });
    
    // 监听year变化
    watch(() => props.year, (newVal) => {
      if (newVal) {
        that_year.value = newVal;
        // 每次年份变化都需要重新初始化
        init();
      }
    });
    
    // 监听month变化
    watch(() => props.month, (newVal) => {
      if (newVal) {
        that_month.value = newVal;
        // 每次月份变化都需要重新初始化
        init();
      }
    });
    
    // 监听range变化
    watch(() => props.range, (newVal) => {
      if (newVal && newVal.length !== 0) {
        setRangeBg();
      }
    }, { deep: true });
    
    // 点击周
    const weekTap = (index: number) => {
      selectLine.value = parseInt(String(index));
      backFn();
    };
    
    // 初始化
    const init = () => {
      try {
        // 检查初始化所需数据是否完整
        if (!that_year.value || !that_month.value) {
          console.error('初始化日历组件失败：年份或月份缺失');
          return;
        }
        
        // 防止初次加载多次渲染
        load.value = false;
        list.value = [];
        
        // 月份需要减一，否则会是次月
        const currentMonth = moment(new Date(parseInt(that_year.value), parseInt(that_month.value) - 1, 1));
        const startOfMonth = moment(currentMonth).startOf('month');
        const endOfMonth = moment(currentMonth).endOf('month');
        
        // 当月的天数
        const days = moment(currentMonth).daysInMonth();
        
        // 获取到本月第一天是星期几
        let week = moment(currentMonth).day();
        // 调整week，使周一为1，周日为7
        if (week === 0) week = 7;
        
        let tempList: DayItem[] = [];
        
        // 补上个月的日期
        for (let i = week - 1; i > 0; i--) {
          let before = moment(startOfMonth).subtract(i, 'days');
          tempList.push(getItem(before, false));
        }
        
        // 当月日期
        for (let i = 0; i < days; i++) {
          let after = moment(startOfMonth).add(i, 'days');
          tempList.push(getItem(after, true));
        }
        
        // 把list分组 一组7个
        let data = chunk(tempList, 7);
        
        // 获取到最后一周，如果最后一周数量不足7个，需要补下个月的数据
        if (data.length > 0) {
          let lastWeek = data[data.length - 1];
          let lastWeekLength = lastWeek.length;
          
          // 因为是从0开始 所以添加的天数需要加一
          for (let i = 0; i < 7 - lastWeekLength; i++) {
            let after = moment(endOfMonth).add(i + 1, 'days');
            lastWeek.push(getItem(after, false));
          }
        }
        
        // 更新列表数据
        list.value = data;
        
        // 如果当前选中的行大于可选行数，则选择最后一行
        if (data.length > 0 && parseInt(String(selectLine.value)) >= data.length) {
          selectLine.value = data.length - 1;
        }
        
        // 触发监听回调
        // backFn();
        
        // 如果配置了范围区域
        if (props.range.length !== 0) {
          setRangeBg();
        }
      } catch (error) {
        console.error('初始化日历组件出错:', error);
      }
    };
    
    // 设置范围背景
    const setRangeBg = () => {
      try {
        if (!props.range || props.range.length < 2 || list.value.length === 0) return;
        
        const [start, end] = props.range;
        const arr: boolean[] = new Array(list.value.length).fill(false);
        const moment_start = moment(start);
        const moment_end = moment(end);
        
        list.value.forEach((item, index) => {
          let hasInRange = false;
          for (let i = 0; i < item.length; i++) {
            const moment_that = moment(item[i].value);
            if (
              moment_that.diff(moment_start, 'days') >= 0 && 
              moment_that.diff(moment_end, 'days') <= 0
            ) {
              hasInRange = true;
              break;
            }
          }
          if (hasInRange) {
            arr[index] = true;
          }
        });
        
        rangeBg.value = arr;
        
        const lineSize = arr.filter(it => it).length;
        const line_start = arr.indexOf(true);
        let line_height = 38;
        
        if (props.size === "big") {
          line_height = 42;
        } else if (props.size === "small") {
          line_height = 32;
        }
        
        // 更新高亮样式
        highlightStyle.display = line_start === -1 ? "none" : "block";
        highlightStyle.top = line_height + line_start * line_height + (line_start * 2);
        highlightStyle.height = line_height * lineSize + (lineSize - 1) * 2;
      } catch (error) {
        console.error('设置范围背景出错:', error);
        // 出错时隐藏高亮样式
        highlightStyle.display = "none";
      }
    };
    
    // 返回选中的行
    const backFn = () => {
      // console.log('触发了监听返回')
      try {
        if (list.value.length === 0 || selectLine.value >= list.value.length) return;
        
        const data = list.value[selectLine.value].map(item => item.value);
        emit('backevent', { data });
      } catch (error) {
        console.error('返回选中周数据出错:', error);
      }
    };
    
    // 获取显示的日期对象
    const getItem = (date: moment.Moment, isCurrentMonth: boolean): DayItem => {
      try {
        const isToday = today.value ? date.isSame(today.value, 'day') : false;
        return {
          title: parseInt(date.format('DD')) + '',
          value: date.format('YYYY-MM-DD'),
          isCurrentMonth,
          isToday,
        };
      } catch (error) {
        console.error('创建日期项出错:', error);
        // 返回默认值，避免崩溃
        return {
          title: '',
          value: '',
          isCurrentMonth: false,
          isToday: false,
        };
      }
    };
    
    // 数组分割函数
    const chunk = (arr: any[], chunkSize: number) => {
      const result = [];
      for (let i = 0; i < arr.length; i += chunkSize) {
        result.push(arr.slice(i, i + chunkSize));
      }
      return result;
    };
    
    // 组件挂载时初始化
    onMounted(() => {
      const currentMoment = moment();
      today.value = currentMoment;
      
      // 如果没有读取到初始数据，则赋值当年当月
      if (!that_year.value && !that_month.value) {
        that_year.value = currentMoment.year().toString();
        // 此处获取到的月份需要加一
        that_month.value = (currentMoment.month() + 1).toString();
      }
      
      init();
    });
    
    return {
      list,
      selectLine,
      rangeBg,
      highlightStyle,
      weekTap,
      init,
      displayYearMonth,
      prevMonth,
      nextMonth,
    };
  },
});
</script>

<style scoped>
.calendar-box {
  font-weight: 600;
  font-size: 14px;
}

.wrap {
  line-height: 38px;
  border-radius: 6px;
  position: relative;
}

.wrap::after {
  content: " ";
  display: block;
  font-size: 0;
  height: 0;
  overflow: hidden;
  clear: both;
}

/* big small样式 start */
.big, .big .wrap {
  font-size: 16px;
  line-height: 42px;
}

.small, .small .wrap {
  font-size: 12px;
  line-height: 32px;
}
/* big small样式 end */

.wrap .week {
  display: flex;
  border-radius: 6px;
}

.wrap .week-box {
  border-radius: 6px;
  z-index: 10;
  position: relative;
  margin: 2px;
}

.wrap .week .day {
  flex: 1;
  text-align: center;
  color: #323232;
  font-weight: 400;
  border-radius: 6px;
}

.wrap .week .day:last-child, .wrap .week .sunday {
  border-right: none;
}

.wrap .header .day {
  color: #323232;
  font-weight: 800;
}

.wrap .week .active {
  position: relative;
}

.wrap .week .active::after {
  content: " ";
  display: block;
  position: absolute;
  top: 50%;
  left: 50%;
  background-color: rgba(235, 51, 51, 0.2);
  height: 32px;
  width: 32px;
  margin-top: -16px;
  margin-left: -16px;
  border-radius: 6px;
  z-index: 1;
  border: 1px solid #eb3333;
}

.wrap .week .active span {
  color: #333;
  z-index: 10;
  position: relative;
  font-weight: bold;
}

/* 周六 */
.wrap .week .saturday {
  color: #eb3333;
}

/* 周日 */
.wrap .week .sunday {
  color: #eb3333;
}

.wrap .header {
  background-color: #f4f5f8;
}

.wrap .week .day .grey {
  color: #bbb;
}

.wrap .select {
  background-color: #ffecea;
  border-radius: 6px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

.wrap .on .select {
  background-color: #ffecea;
}

/* 蓝色主题下的选中行样式 */
.blue .wrap .select {
  background-color: #e9f2fe;
}

.tip {
  text-align: center;
  color: #bbb;
  line-height: 50px;
}

.border .wrap {
  border: 1px solid #efefef;
}

.highlight {
  position: absolute;
  border: 2px solid #ffd9d5;
  top: 38px;
  left: 0;
  right: 0;
  z-index: 1;
  border-radius: 8px;
}

/* theme */
.blue .wrap .week .active::after {
  background-color: rgba(26, 137, 250, 0.2);
  border: 1px solid #1a89fa;
}

.blue .wrap .select {
  background-color: #e9f2fe;
}

.blue .wrap .on .select {
  background-color: #e9f2fe;
}

.blue .wrap .week .day .grey {
  color: #cacbce;
}

.blue .wrap .week .saturday {
  color: #1888f9;
}

.blue .wrap .week .sunday {
  color: #1888f9;
}

.blue .highlight {
  border-color: #e9f2fe;
}

/* big small */
.big .wrap .week .active::after {
  height: 36px;
  width: 36px;
  margin-top: -18px;
  margin-left: -18px;
}

.small .wrap .week .active::after {
  height: 28px;
  width: 28px;
  margin-top: -14px;
  margin-left: -14px;
}

/* 增强今天日期的可见性 */
.wrap .week .active {
  font-weight: bold;
}

/* 月份导航样式 */
.month-navigation {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px 0;
  margin-bottom: 10px;
  background-color: #f4f5f8;
  border-radius: 6px;
}

.month-nav-btn {
  width: 36px;
  height: 36px;
  display: flex;
  justify-content: center;
  align-items: center;
  cursor: pointer;
}

.nav-icon {
  font-size: 16px;
  font-weight: bold;
}

.month-display {
  font-size: 16px;
  font-weight: bold;
}
</style>
