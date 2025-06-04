import { getHolidaysOf } from 'japanese-holidays';

const [holidayMap, setHolidayMap] = useState<Record<string, string>>({});
useEffect(() => {
  const currentYear = new Date().getFullYear(); // 或固定年份 2025
  const holidays = getHolidaysOf(currentYear);
  const formattedHolidays = holidays.reduce((acc, holiday) => {
    const date = new Date(currentYear, holiday.month - 1, holiday.date);
    // 使用 FullCalendar 的日期格式化（必须与日历内部一致）
    const dateKey = date.toISOString().slice(0, 10); // "YYYY-MM-DD"
    acc[dateKey] = holiday.name;
    return acc;
  }, {} as Record<string, string>);
  
  // console.log("节假日数据:", formattedHolidays);
  setHolidayMap(formattedHolidays);
}, []);

datesSet={(arg) => {
    setTimeout(() => {

      const currentView = arg.view.type; // 获取当前视图类型

    // 如果是日视图，不修改背景色
    if (currentView === 'resourceTimelineDay') return;

      document.querySelectorAll('.fc-timeline-slot[data-date]').forEach(slot => {
        const date = slot.getAttribute('data-date');
        
        // 清除所有节假日标记
        const existingMarks = slot.querySelectorAll('.holiday-label');
        existingMarks.forEach(el => el.remove());
        
        if (date && holidayMap[date]) {
           slot.style.backgroundColor = 'pink';
          // 找到第二行的容器
          const weekdayEl = slot.querySelector('.fc-timeline-slot-cushion');
          if (weekdayEl) {
            // 创建节假日名称元素
            const holidayEl = document.createElement('div');
            holidayEl.className = 'holiday-label';
            
            // 处理长名称
            const holidayName = holidayMap[date].length > 5 
              ? holidayMap[date].substring(0, 5) + '...' 
              : holidayMap[date];
            holidayEl.textContent = holidayName;
            holidayEl.title = holidayMap[date]; // 悬浮提示完整名称
            
            // 插入到星期显示之后
            weekdayEl.parentNode?.insertBefore(
              holidayEl,
              weekdayEl.nextSibling
            );
          }
        }
      });
    }, 100);
  }}




  /* 周末样式 */
 .fc-day-sun, .fc-day-sat {
  background-color: pink !important;
 }

.fc-timeline-slot-frame > div:last-child{
  display: flex;
  flex-direction: column; 
}

/* 第二行容器 */
/* .fc-timeline-slot-cushion  > div:last-child {
  font-size: 0.85em !important;
  display: inline-block !important;
  margin-right: 4px;
} */

/* 节假日名称样式 */
.holiday-label {
  display: inline-block;
  color: #d32f2f;
  font-weight: bold;
  font-size: 0.85em;
  margin-left: 4px;
  max-width: 60px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  vertical-align: top;
}

/* 单元格整体调整 */
.fc-timeline-slot {
  min-height: 45px;
  padding: 2px 0;
  line-height: 1.3;
}
