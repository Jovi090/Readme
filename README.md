✅ 目标：在 TypeScript + React + FullCalendar 中自动加载日本の祝日，并将其显示为背景事件（淡红色）

=========================
【前提依赖】
=========================
安装 FullCalendar：
npm install @fullcalendar/react @fullcalendar/core @fullcalendar/resource-timeline

=========================
【必要的 import】
=========================
import { useEffect, useState } from 'react';
import FullCalendar from '@fullcalendar/react';
import resourceTimelinePlugin from '@fullcalendar/resource-timeline';

import '@fullcalendar/core/main.css';
import '@fullcalendar/resource-timeline/main.css';

=========================
【类型定义】
=========================
type CalendarEvent = {
  start: string;
  end: string;
  title?: string;
  display?: string;
  classNames?: string[];
};

=========================
【祝日 State】
=========================
const [holidayEvents, setHolidayEvents] = useState<CalendarEvent[]>([]);

=========================
【祝日 API 获取函数】
=========================
const fetchJapaneseHolidays = async (): Promise<{ [date: string]: string }> => {
  const res = await fetch('https://holidays-jp.github.io/api/v1/date.json');
  return await res.json();
};

=========================
【转换为 FullCalendar 事件】
=========================
const convertToBackgroundEvents = (holidays: { [date: string]: string }): CalendarEvent[] => {
  return Object.entries(holidays).map(([date, name]) => ({
    start: date,
    end: date,
    title: name,
    display: 'background',
    classNames: ['fc-holiday-bg'],
  }));
};

=========================
【在 useEffect 中加载】
=========================
useEffect(() => {
  fetchJapaneseHolidays()
    .then(convertToBackgroundEvents)
    .then(setHolidayEvents);
}, []);

=========================
【在 <FullCalendar /> 中使用】
=========================
<FullCalendar
  plugins={[resourceTimelinePlugin]}
  initialView="resourceTimelineMonth"
  events={holidayEvents}
/>

=========================
【添加 CSS（淡红背景）】
=========================
.fc-holiday-bg {
  background-color: rgba(255, 150, 150, 0.2);
}

=========================
【总结】
=========================
- ✅ 无需 API Key，数据源公开免费
- ✅ 适合前端项目直接使用
- ✅ 支持显示祝日名称（作为 tooltip）
- ✅ 可结合 slotLabelContent/dayCellContent 等扩展更多祝日功能
