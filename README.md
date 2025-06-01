目标：使用日本の祝日 API（无认证）将祝日显示为 FullCalendar 中的淡红色背景块

=========================
【使用的 API 来源】
=========================
提供者：https://holidays-jp.github.io/
API 地址：https://holidays-jp.github.io/api/v1/date.json

特点：
- 无需 API Key
- 免费公开
- 返回格式简单（JSON）
- 数据包含祝日名称和日期

=========================
【步骤 1】在组件中定义请求函数
=========================
const fetchJapaneseHolidays = async (): Promise<{ [date: string]: string }> => {
  const res = await fetch('https://holidays-jp.github.io/api/v1/date.json');
  return await res.json();
};

=========================
【步骤 2】转换为背景事件（FullCalendar 用）
=========================
const convertToBackgroundEvents = (holidays: { [date: string]: string }) => {
  return Object.entries(holidays).map(([date, name]) => ({
    start: date,
    end: date,
    title: name,
    display: 'background',
    classNames: ['fc-holiday-bg']
  }));
};

=========================
【步骤 3】加载祝日事件（useEffect 中）
=========================
const [holidayEvents, setHolidayEvents] = useState([]);

useEffect(() => {
  fetchJapaneseHolidays()
    .then(convertToBackgroundEvents)
    .then(setHolidayEvents);
}, []);

=========================
【步骤 4】在 <FullCalendar /> 中使用
=========================
<FullCalendar
  ...
  events={[...yourOtherEvents, ...holidayEvents]}
/>

如无其他事件：
events={holidayEvents}

=========================
【步骤 5】添加祝日背景样式
=========================
.fc-holiday-bg {
  background-color: rgba(255, 150, 150, 0.2);
}

=========================
✅ 示例返回数据
=========================
{
  "2025-01-01": "元日",
  "2025-01-13": "成人の日",
  "2025-02-11": "建国記念の日",
  ...
}

=========================
【总结】
=========================
- ✅ 无需申请 API Key
- ✅ 即调即用，适合浏览器前端直接请求
- ✅ 稳定可靠，适合生产项目使用
- ✅ 可轻松扩展为 Tooltip、节假日标记等功能
