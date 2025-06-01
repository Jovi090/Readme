import React from 'react';
import FullCalendar from '@fullcalendar/react';
import timeGridPlugin from '@fullcalendar/timegrid';
import interactionPlugin, {
  DateSelectArg,
  EventDropArg,
  EventResizeDoneArg,
} from '@fullcalendar/interaction';
import { EventApi } from '@fullcalendar/core';

const CalendarComponent: React.FC = () => {
  // 限制事件时间：必须在 08:00 - 22:00
  const isWithinAllowedHours = (start: Date, end?: Date | null): boolean => {
    const startHour = start.getHours() + start.getMinutes() / 60;
    const endDate = end ?? new Date(start.getTime() + 60 * 60 * 1000); // 默认持续 1 小时
    const endHour = endDate.getHours() + endDate.getMinutes() / 60;

    return startHour >= 8 && endHour <= 22;
  };

  // 拖动事件时限制
  const handleEventDrop = (info: EventDropArg) => {
    if (!isWithinAllowedHours(info.event.start!, info.event.end)) {
      alert('事件必须在 08:00 - 22:00 之间');
      info.revert();
    }
  };

  // 拉伸事件时限制
  const handleEventResize = (info: EventResizeDoneArg) => {
    if (!isWithinAllowedHours(info.event.start!, info.event.end)) {
      alert('事件必须在 08:00 - 22:00 之间');
      info.revert();
    }
  };

  // 新建事件时限制
  const handleSelectAllow = ({ start, end }: DateSelectArg): boolean => {
    return isWithinAllowedHours(start, end);
  };

  return (
    <FullCalendar
      plugins={[timeGridPlugin, interactionPlugin]}
      initialView="timeGridWeek"
      editable={true}
      selectable={true}
      slotMinTime="08:00:00"
      slotMaxTime="22:00:00"
      selectAllow={handleSelectAllow}
      eventDrop={handleEventDrop}
      eventResize={handleEventResize}
      events={[]}
    />
  );
};

export default CalendarComponent;
