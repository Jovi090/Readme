import React, { useState, useEffect } from 'react';
import FullCalendar from '@fullcalendar/react';
import resourceTimelinePlugin from '@fullcalendar/resource-timeline';
import interactionPlugin from '@fullcalendar/interaction';
import './styles/viewPage.css';
import { ToastContainer } from 'react-toastify';
import 'react-toastify/dist/ReactToastify.css';
import jaLocale from '@fullcalendar/core/locales/ja';
import axios from 'axios';

const EventDetailModal: React.FC<{ event: any; onClose: () => void }> = ({ event, onClose }) => {
    const ext = event.extendedProps || {};
    return (
        <div className="view-modal-overlay">
            <div className="view-modal-content">
                <h3>{event.title}</h3>
                <ul style={{ listStyle: 'none', paddingLeft: 0 }}>
                    <li><b>担当講師：</b>{ext.instructor || "未設定"}</li>
                    <li><b>所要日数：</b>{ext.duration || "未設定"}</li>
                    <li><b>会場：</b>{ext.location || "未設定"}</li>
                    <li><b>担当部署：</b>{ext.department || "未設定"}</li>
                </ul>
                <button className="view-modal-close-btn" onClick={onClose}>閉じる</button>
            </div>
        </div>
    );
};

const ViewPage: React.FC = () => {
    const [events, setEvents] = useState<any[]>([]);
    const [resources, setResources] = useState<any[]>([]);
    const [detailEvent, setDetailEvent] = useState<any | null>(null);
    // const [isPublished, setIsPublished] = useState<boolean>(true); // 公開判定は現在無効化

    useEffect(() => {
        const fetchData = async () => {
            try {
                const res = await axios.get('http://localhost:8080/api/schedules/published');
                const raw = res.data;

                const eventsData = raw.map((item: any) => ({
                    id: item.id,
                    title: item.curriculumsName,
                    start: item.startDateTime,
                    end: item.endDateTime,
                    resourceId: item.groupsName,
                    extendedProps: {
                        instructor: item.lecturersName,
                        department: item.departmentsName,
                        location: item.roomsName,
                        duration: "N/A"
                    }
                }));

                const uniqueGroups = Array.from(new Set(raw.map((item: any) => item.groupsName)));
                const resourceData = uniqueGroups.map((name: string) => ({
                    id: name,
                    title: name
                }));

                setEvents(eventsData);
                setResources(resourceData);
            } catch (err) {
                console.error("データ取得に失敗しました", err);
                // setIsPublished(false); // 公開判定無効化
            }
        };
        fetchData();
    }, []);

    return (
        <div className="view-page-calendar-container">
            <ToastContainer position="top-center" autoClose={3000} hideProgressBar newestOnTop closeOnClick pauseOnFocusLoss draggable pauseOnHover />
            <div className="view-page-calendar-wrapper">
                <FullCalendar
                    locale="ja"
                    locales={[jaLocale]}
                    schedulerLicenseKey="CC-Attribution-NonCommercial-NoDerivatives"
                    plugins={[resourceTimelinePlugin, interactionPlugin]}
                    initialView="resourceTimelineDay"
                    editable={false}
                    droppable={false}
                    resources={resources}
                    events={events}
                    resourceAreaWidth={"26%"}
                    eventMaxStack={2}
                    firstDay={1}
                    resourceAreaHeaderContent={"クラス"}
                    slotMinTime={"09:00:00"}
                    slotMaxTime={"18:00:00"}
                    height="auto"
                    headerToolbar={{
                        left: 'prev,next',
                        center: 'title',
                        right: 'resourceTimelineDay,resourceTimelineWeek,resourceTimelineMonth'
                    }}
                    views={{
                        resourceTimelineDay: { buttonText: '日' },
                        resourceTimelineMonth: { buttonText: '月' },
                        resourceTimelineWeek: {
                            buttonText: '週',
                            dayHeaderFormat: { weekday: 'short', day: 'numeric' },
                            dayHeaders: true,
                            type: 'resourceTimeline',
                            duration: { weeks: 1 },
                            slotDuration: { days: 1 },
                            slotLabelFormat: [{ weekday: 'short', day: 'numeric' }]
                        }
                    }}
                    eventContent={(arg) => ({
                        html: `<div class="fc-event-container"><div class="fc-event-main">${arg.event.title}</div></div>`
                    })}
                    eventClick={(info) => {
                        setDetailEvent({
                            title: info.event.title,
                            extendedProps: info.event.extendedProps
                        });
                    }}
                />
                {detailEvent && (
                    <EventDetailModal
                        event={detailEvent}
                        onClose={() => setDetailEvent(null)}
                    />
                )}
            </div>
        </div>
    );
};

export default ViewPage;
