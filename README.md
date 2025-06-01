<FullCalendar
  plugins={[timeGridPlugin, interactionPlugin]}
  initialView="timeGridWeek"
  editable={true}
  selectable={true}
  slotMinTime="08:00:00"
  slotMaxTime="22:00:00"

  eventAllow={({ start, end }) => {
    const startHour = start.getHours() + start.getMinutes() / 60;
    const endHour = (end?.getHours() ?? 22) + (end?.getMinutes() ?? 0) / 60;
    return startHour >= 8 && endHour <= 22;
  }}
/>
