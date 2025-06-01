import FullCalendar from '@fullcalendar/react';
import timeGridPlugin from '@fullcalendar/timegrid';
import interactionPlugin from '@fullcalendar/interaction';

<FullCalendar
  plugins={[timeGridPlugin, interactionPlugin]}
  initialView="timeGridWeek"
  editable={true}
  selectable={true}
  slotMinTime="08:00:00"
  slotMaxTime="22:00:00"

  // 限制拖拽或调整的时间
  eventAllow={({ start, end }) => {
    const startHour = start.getHours() + start.getMinutes() / 60;
    const actualEnd = end ?? new Date(start.getTime() + 60 * 60 * 1000);
    const endHour = actualEnd.getHours() + actualEnd.getMinutes() / 60;
    return startHour >= 8 && endHour <= 22;
  }}

  // 限制新建事件（画选区）
  selectAllow={({ start, end }) => {
    const startHour = start.getHours() + start.getMinutes() / 60;
    const endHour = end.getHours() + end.getMinutes() / 60;
    return startHour >= 8 && endHour <= 22;
  }}

  events={[]}
/>
