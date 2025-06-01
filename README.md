✅ 目标：在 FullCalendar 中将日本の祝日显示为淡红色背景块（自动获取）

=========================
【步骤 1】获取 Google API Key
=========================
1. 打开 https://console.cloud.google.com/
2. 创建一个项目（如未创建）
3. 启用「Google Calendar API」
4. 前往“API 和服务” > “凭据”，创建 API Key
5. 获取到 API Key，例如：AIzaSyXXXX...

=========================
【步骤 2】准备祝日数据请求函数
=========================
在你的 React 组件文件顶部添加：

const JAPANESE_HOLIDAY_CALENDAR_ID = 'ja.japanese#holiday@group.v.calendar.google.com';
const API_KEY = 'YOUR_GOOGLE_API_KEY'; // ⛳️ 替换成你自己的 API Key

const fetchJapaneseHolidays = async (timeMin: string, timeMax: string) => {
  const url = `https://www.googleapis.com/calendar/v3/calendars/${encodeURIComponent(
    JAPANESE_HOLIDAY_CALENDAR_ID
  )}/events?key=${API_KEY}&timeMin=${timeMin}&timeMax=${timeMax}&singleEvents=true&orderBy=startTime`;

  const res = await fetch(url);
  const data = await res.json();

  return (data.items || []).map((item: any) => ({
    start: item.start.date,
    end: item.end.date,
    title: item.summary,
    display: 'background',
    classNames: ['fc-holiday-bg'],
  }));
};

=========================
【步骤 3】在组件中加载祝日事件
=========================
import { useEffect, useState } from 'react';

const [holidayEvents, setHolidayEvents] = useState([]);

useEffect(() => {
  const timeMin = '2025-01-01T00:00:00Z';
  const timeMax = '2027-01-01T00:00:00Z';

  fetchJapaneseHolidays(timeMin, timeMax).then(setHolidayEvents);
}, []);

=========================
【步骤 4】在 FullCalendar 中传入祝日事件
=========================
<FullCalendar
  ...
  events={[...yourOtherEvents, ...holidayEvents]}
/>

如无其他事件，可直接：
events={holidayEvents}

=========================
【步骤 5】添加淡红色背景样式
=========================
.fc-holiday-bg {
  background-color: rgba(255, 150, 150, 0.2);
}

=========================
✅ 可选扩展：
=========================
- 加载 weekend 背景颜色（如需我也可以提供）
- 祝日 tooltip 提示：title 字段已包含祝日名称
- 多语言支持或祝日图标扩展等
