import React, { useState, useEffect } from 'react';
import FullCalendar from '@fullcalendar/react';
import resourceTimelinePlugin from '@fullcalendar/resource-timeline';

const mockData = {
  room: {
    resources: [
      { id: 'roomA', title: 'A 教室' },
      { id: 'roomB', title: 'B 教室' },
    ],
    events: [
      {
        id: '1',
        title: '数学课',
        start: '2024-06-01T10:00:00',
        end: '2024-06-01T12:00:00',
        resourceId: 'roomA',
      },
      {
        id: '2',
        title: '英语课',
        start: '2024-06-01T13:00:00',
        end: '2024-06-01T15:00:00',
        resourceId: 'roomB',
      },
    ],
  },
  group: {
    resources: [
      { id: 'group1', title: '一年级一班' },
      { id: 'group2', title: '一年级二班' },
    ],
    events: [
      {
        id: '1',
        title: '数学课',
        start: '2024-06-01T10:00:00',
        end: '2024-06-01T12:00:00',
        resourceId: 'group1',
      },
      {
        id: '2',
        title: '英语课',
        start: '2024-06-01T13:00:00',
        end: '2024-06-01T15:00:00',
        resourceId: 'group2',
      },
    ],
  },
};

const MyCalendar = () => {
  const [resourceType, setResourceType] = useState('room');
  const [resources, setResources] = useState([]);
  const [events, setEvents] = useState([]);

  const fetchData = async (type) => {
    // 模拟后端数据
    const data = mockData[type];
    setResources(data.resources);
    setEvents(data.events);

    // ✅ 若接入后端，替换为以下代码即可：
    /*
    const res = await fetch(`/calendar-data?type=${type}`);
    const data = await res.json();
    setResources(data.resources);
    setEvents(data.events);
    */
  };

  useEffect(() => {
    fetchData(resourceType);
  }, [resourceType]);

  const toggleResourceType = () => {
    setResourceType(prev => (prev === 'room' ? 'group' : 'room'));
  };

  return (
    <FullCalendar
      plugins={[resourceTimelinePlugin]}
      initialView="resourceTimelineDay"
      headerToolbar={{
        left: 'prev,next today toggleResource',
        center: 'title',
        right: 'resourceTimelineDay,resourceTimelineWeek',
      }}
      customButtons={{
        toggleResource: {
          text: '切换资源类型',
          click: toggleResourceType,
        },
      }}
      resources={resources}
      events={events}
      resourceAreaHeaderContent={resourceType === 'room' ? '教室' : '班级'}
      height="auto"
    />
  );
};

export default MyCalendar;
