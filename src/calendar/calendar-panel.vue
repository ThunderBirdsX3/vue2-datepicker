<template>
  <div
    :class="[
      'mx-calendar',
      `mx-calendar-panel-${panel}`,
      { 'mx-calendar-week-mode': type === 'week' },
    ]"
  >
    <div class="mx-calendar-header">
      <button
        v-show="showIconDoubleArrow"
        class="mx-btn mx-btn-text mx-btn-icon-double-left"
        @click="handleIconDoubleLeftClick"
      >
        <i class="mx-icon-double-left"></i>
      </button>
      <button
        v-show="showIconArrow"
        class="mx-btn mx-btn-text mx-btn-icon-left"
        @click="handleIconLeftClick"
      >
        <i class="mx-icon-left"></i>
      </button>
      <button
        v-show="showIconDoubleArrow"
        class="mx-btn mx-btn-text mx-btn-icon-double-right"
        @click="handleIconDoubleRightClick"
      >
        <i class="mx-icon-double-right"></i>
      </button>
      <button
        v-show="showIconArrow"
        class="mx-btn mx-btn-text mx-btn-icon-right"
        @click="handleIconRightClick"
      >
        <i class="mx-icon-right"></i>
      </button>
      <span class="mx-calendar-header-label">
        <span v-if="panel === 'year'">{{ yearHeader }}</span>
        <button
          v-else-if="panel === 'month'"
          class="mx-btn mx-btn-text"
          @click="handelPanelChange('year')"
        >
          {{ monthHeader }}
        </button>
        <template v-else-if="panel === 'date'">
          <button
            v-for="item in dateHeader"
            :key="item.panel"
            :class="`mx-btn mx-btn-text mx-btn-current-${item.panel}`"
            @click="handelPanelChange(item.panel)"
          >
            {{ item.label }}
          </button>
        </template>
      </span>
    </div>
    <div class="mx-calendar-content">
      <table-year
        v-show="panel === 'year'"
        :decade="calendarDecade"
        :get-cell-classes="getYearClasses"
        @select="handleSelectYear"
      ></table-year>
      <table-month
        v-if="type !== 'year'"
        v-show="panel === 'month'"
        :get-cell-classes="getMonthClasses"
        @select="handleSelectMonth"
      ></table-month>
      <table-date
        v-if="type !== 'year' && type !== 'month'"
        v-show="panel === 'date'"
        :calendar-year="calendarYear"
        :calendar-month="calendarMonth"
        :title-format="titleFormat"
        :show-week-number="typeof showWeekNumber === 'boolean' ? showWeekNumber : type === 'week'"
        :get-cell-classes="getDateClasses"
        :get-row-classes="getWeekState"
        @select="handleSelectDate"
      ></table-date>
    </div>
  </div>
</template>

<script>
import {
  subMonths,
  addMonths,
  subYears,
  addYears,
  setMonth,
  setYear,
  startOfYear,
  startOfMonth,
  startOfDay,
} from 'date-fns';
import localeMixin from '../mixin/locale';
import formatMixin from '../mixin/format';
import { getValidDate, isValidDate } from '../util/date';
import TableDate from './table-date';
import TableMonth from './table-month';
import TableYear from './table-year';

export default {
  name: 'CalendarPanel',
  components: {
    TableDate,
    TableMonth,
    TableYear,
  },
  mixins: [localeMixin, formatMixin],
  props: {
    value: {},
    defaultValue: {
      validator: value => isValidDate(value),
      default() {
        const date = new Date();
        date.setHours(0, 0, 0, 0);
        return date;
      },
    },
    disabledDate: {
      type: Function,
      default: () => false,
    },
    type: {
      type: String,
      default: 'date',
    },
    getClasses: {
      type: Function,
      default: () => [],
    },
    showWeekNumber: {
      type: Boolean,
      default: undefined,
    },
    titleFormat: {
      type: String,
      default: 'YYYY-MM-DD',
    },
    calendar: Date,
  },
  data() {
    const panels = ['date', 'year', 'month'];
    const panel = panels.indexOf(this.type) !== -1 ? this.type : 'date';
    return {
      panel,
      innerCalendar: null,
    };
  },
  computed: {
    innerValue() {
      const value = Array.isArray(this.value) ? this.value : [this.value];
      const map = {
        year: startOfYear,
        month: startOfMonth,
        date: startOfDay,
      };
      const start = map[this.type] || map.date;
      return value.filter(isValidDate).map(v => start(v));
    },
    calendarYear() {
      return this.innerCalendar.getFullYear();
    },
    calendarMonth() {
      return this.innerCalendar.getMonth();
    },
    calendarDecade() {
      return Math.floor(this.calendarYear / 10) * 10;
    },
    showIconDoubleArrow() {
      return this.panel === 'date' || this.panel === 'month' || this.panel === 'year';
    },
    showIconArrow() {
      return this.panel === 'date';
    },
    yearHeader() {
      return `${this.calendarDecade} ~ ${this.calendarDecade + 9}`;
    },
    monthHeader() {
      return this.calendarYear;
    },
    dateHeader() {
      const monthBeforeYear = this.t('monthBeforeYear');
      const yearFormat = this.t('yearFormat');
      const monthFormat = this.t('monthFormat') || 'MMM';
      const yearLabel = {
        panel: 'year',
        label: this.formatDate(this.innerCalendar, yearFormat),
      };
      const monthLabel = {
        panel: 'month',
        label: this.formatDate(this.innerCalendar, monthFormat),
      };
      return monthBeforeYear ? [monthLabel, yearLabel] : [yearLabel, monthLabel];
    },
  },
  watch: {
    value: {
      immediate: true,
      handler: 'initCalendar',
    },
    calendar: {
      handler: 'initCalendar',
    },
    defaultValue: {
      handler: 'initCalendar',
    },
  },
  methods: {
    initCalendar() {
      let calendarDate = this.calendar;
      if (!isValidDate(calendarDate)) {
        calendarDate = getValidDate(this.innerValue[0], this.defaultValue);
      }
      this.innerCalendar = calendarDate;
    },
    emitDate(date, type) {
      if (!this.disabledDate(new Date(date))) {
        this.$emit('select', date, type);
      }
    },
    updateCalendar(date) {
      this.innerCalendar = date;
      this.$emit('update:calendar', date);
    },
    handelPanelChange(panel) {
      this.panel = panel;
    },
    handleIconLeftClick() {
      const nextCalendar = subMonths(this.innerCalendar, 1);
      this.updateCalendar(nextCalendar);
    },
    handleIconRightClick() {
      const nextCalendar = addMonths(this.innerCalendar, 1);
      this.updateCalendar(nextCalendar);
    },
    handleIconDoubleLeftClick() {
      const nextCalendar = subYears(this.innerCalendar, this.panel === 'year' ? 10 : 1);
      this.updateCalendar(nextCalendar);
    },
    handleIconDoubleRightClick() {
      const nextCalendar = addYears(this.innerCalendar, this.panel === 'year' ? 10 : 1);
      this.updateCalendar(nextCalendar);
    },
    handleSelectYear(year) {
      if (this.type === 'year') {
        const date = this.getCellDate(year, 'year');
        this.emitDate(date, 'year');
      } else {
        const nextCalendar = setYear(this.innerCalendar, year);
        this.updateCalendar(nextCalendar);
        this.handelPanelChange('month');
      }
    },
    handleSelectMonth(month) {
      if (this.type === 'month') {
        const date = this.getCellDate(month, 'month');
        this.emitDate(date, 'month');
      } else {
        const nextCalendar = setMonth(this.innerCalendar, month);
        this.updateCalendar(nextCalendar);
        this.handelPanelChange('date');
      }
    },
    handleSelectDate(day) {
      const date = this.getCellDate(day, 'date');
      this.emitDate(date, this.type === 'week' ? 'week' : 'date');
    },
    getCellDate(value, type) {
      if (type === 'year') {
        return new Date(value, 0);
      }
      if (type === 'month') {
        return new Date(this.calendarYear, value);
      }
      return new Date(this.calendarYear, this.calendarMonth, value);
    },
    getDateClasses(day) {
      const cellDate = this.getCellDate(day, 'date');
      const notCurrentMonth = cellDate.getMonth() !== this.calendarMonth;
      const classes = [];
      if (cellDate.getTime() === new Date().setHours(0, 0, 0, 0)) {
        classes.push('today');
      }
      if (notCurrentMonth) {
        classes.push('not-current-month');
      }
      const state = this.getStateClass(cellDate);
      if (!(state === 'active' && notCurrentMonth)) {
        classes.push(state);
      }
      return classes.concat(this.getClasses(cellDate, this.innerValue, classes.join(' ')));
    },
    getMonthClasses(month) {
      if (this.type !== 'month') {
        return this.calendarMonth === month ? 'active' : '';
      }
      const classes = [];
      const cellDate = this.getCellDate(month, 'month');
      classes.push(this.getStateClass(cellDate));
      return classes.concat(this.getClasses(cellDate, this.innerValue, classes.join(' ')));
    },
    getYearClasses(year) {
      if (this.type !== 'year') {
        return this.calendarYear === year ? 'active' : '';
      }
      const classes = [];
      const cellDate = this.getCellDate(year, 'year');
      classes.push(this.getStateClass(cellDate));
      return classes.concat(this.getClasses(cellDate, this.innerValue, classes.join(' ')));
    },
    getStateClass(cellDate) {
      if (this.disabledDate(new Date(cellDate))) {
        return 'disabled';
      }
      if (this.innerValue.some(v => v.getTime() === cellDate.getTime())) {
        return 'active';
      }
      return '';
    },
    getWeekState(row) {
      if (this.type !== 'week') return '';
      const start = this.getCellDate(row[0].day, 'date').getTime();
      const end = this.getCellDate(row[6].day, 'date').getTime();
      const active = this.innerValue.some(v => {
        const time = v.getTime();
        return time >= start && time <= end;
      });
      return active ? 'mx-active-week' : '';
    },
  },
};
</script>