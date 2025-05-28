import React, { useState, useEffect, useRef } from 'react';
import FullCalendar from '@fullcalendar/react';
import resourceTimelinePlugin from '@fullcalendar/resource-timeline';
import interactionPlugin from '@fullcalendar/interaction';

const MyCalendar = () => {
  const [events, setEvents] = useState([
    {
      id: '1',
      title: '数学课',
      start: '2024-06-01T10:00:00',
      end: '2024-06-01T12:00:00',
      resourceId: 'roomA',
    },
  ]);
  const [copyKey, setCopyKey] = useState(false);
  const calendarRef = useRef(null);

  // 监听 shift 键
  useEffect(() => {
    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.shiftKey) setCopyKey(true);
    };
    const handleKeyUp = () => {
      setCopyKey(false);
    };
    window.addEventListener('keydown', handleKeyDown);
    window.addEventListener('keyup', handleKeyUp);
    return () => {
      window.removeEventListener('keydown', handleKeyDown);
      window.removeEventListener('keyup', handleKeyUp);
    };
  }, []);

  const handleEventDrop = (info: any) => {
    const { event } = info;

    if (copyKey) {
      // 复制一个新事件
      const newEvent = {
        id: String(Date.now()), // 简单生成唯一 id
        title: event.title,
        start: event.startStr,
        end: event.endStr,
        resourceId: event.getResources()[0]?.id,
      };

      setEvents(prev => [...prev, newEvent]);

      // 还原原事件的位置
      info.revert();
    } else {
      // 正常拖动修改事件时间
      const updated = events.map(e =>
        e.id === event.id
          ? { ...e, start: event.startStr, end: event.endStr }
          : e
      );
      setEvents(updated);
    }
  };

  return (
    <FullCalendar
      ref={calendarRef}
      plugins={[resourceTimelinePlugin, interactionPlugin]}
      initialView="resourceTimelineDay"
      editable={true}
      events={events}
      resources={[
        { id: 'roomA', title: 'A 教室' },
        { id: 'roomB', title: 'B 教室' },
      ]}
      eventDrop={handleEventDrop}
    />
  );
};

export default MyCalendar;
