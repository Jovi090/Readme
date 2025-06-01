views={{
  resourceTimelineMonth: {
    buttonText: '月',
    duration: { months: 13 },
    slotDuration: { days: 1 },
    slotLabelFormat: [  // 这个可以保留（结构备用），但实际由 slotLabelContent 控制
      { year: 'numeric', month: 'long' },
      { weekday: 'short', day: 'numeric' }
    ],
    slotLabelContent: (arg) => {
      const date = arg.date;

      if (arg.level === 0) {
        return {
          html: `<span>${date.getFullYear()}年${date.getMonth() + 1}月</span>`
        };
      }

      if (arg.level === 1) {
        const day = date.getDate();
        const weekday = ['日', '月', '火', '水', '木', '金', '土'][date.getDay()];
        return {
          html: `<span>${day}(${weekday})</span>`
        };
      }

      return {};
    }
  }
}}
