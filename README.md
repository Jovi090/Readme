===== frontend/ViewPage.tsx =====
import React, { useState } from 'react';
import FullCalendar from '@fullcalendar/react';
import resourceTimelinePlugin from '@fullcalendar/resource-timeline';
import interactionPlugin from '@fullcalendar/interaction';
import './styles/viewPage.css';
import { ToastContainer } from 'react-toastify';
import 'react-toastify/dist/ReactToastify.css';
import jaLocale from '@fullcalendar/core/locales/ja';

import { useEvents } from '../contexts/EventContext';
import { useSettings } from '../contexts/SettingsContext';

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
    const { publishedEvents, isPublished } = useEvents();
    const { classes } = useSettings();
    const [detailEvent, setDetailEvent] = useState<any | null>(null);

    const resources = classes.map(cls => ({
        id: String(cls.id),
        title: cls.name,
    }));

    if (!isPublished) {
        return (
            <div className="view-page-calendar-container">
                <div className="view-page-calendar-wrapper unpublished-center">
                    <div className="not-published-card">
                        <div className="not-published-icon">
                            <svg width="40" height="40" fill="none" viewBox="0 0 24 24">
                                <path fill="#6366F1" d="M12 2a10 10 0 1 0 10 10A10.011 10.011 0 0 0 12 22m0 17a1.5 1.5 0 1 1" />
                            </svg>
                        </div>
                        <div className="not-published-title">
                            まだ公開されていません
                        </div>
                        <div className="not-published-message">
                            編集画面で「公開」ボタンを押すと、<br />
                            ここにカリキュラムが表示されます。
                        </div>
                    </div>
                </div>
            </div>
        );
    }

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
                    events={publishedEvents}
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

===== backend/entity/SchedulePublished.java =====
package simplex.bn25.centmerci.server.entity;

import jakarta.persistence.*;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;
import java.time.LocalDateTime;

@Entity
@Table(name = "schedules_published")
@Data
@NoArgsConstructor
@AllArgsConstructor 
public class SchedulePublished {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id; 

    @Column(name = "groups_name")
    private String groupsName; 

    @Column(name = "departments_name")
    private String departmentsName; 

    @Column(name = "lecturers_name")
    private String lecturersName; 

    @Column(name = "curriculums_name", nullable = false) 
    private String curriculumsName;

    @Column(name = "rooms_name")
    private String roomsName; 

    @Column(name = "categories_name", nullable = false)
    private String categoriesName; 

    @Column(name = "background_color", nullable = false)
    private String backgroundColor; 

    @Column(name = "start_datetime", nullable = false) 
    private LocalDateTime startDateTime; 

    @Column(name = "end_datetime", nullable = false) 
    private LocalDateTime endDateTime; 

    @Column(name = "created_datetime", nullable = false, updatable = false) 
    private LocalDateTime createdDateTime = LocalDateTime.now();
}

===== backend/dto/SchedulePublishedDTO.java =====
package simplex.bn25.centmerci.server.dto;

import lombok.Data;
import java.time.LocalDateTime;

@Data
public class SchedulePublishedDTO {
    private Long id;
    private String groupsName;
    private String departmentsName;
    private String lecturersName;
    private String curriculumsName;
    private String roomsName;
    private String categoriesName;
    private String backgroundColor;
    private LocalDateTime startDateTime;
    private LocalDateTime endDateTime;
}

===== backend/repository/SchedulePublishedRepository.java =====
package simplex.bn25.centmerci.server.repository;

import org.springframework.data.jpa.repository.JpaRepository;
import simplex.bn25.centmerci.server.entity.SchedulePublished;

public interface SchedulePublishedRepository extends JpaRepository<SchedulePublished, Long> {
}

===== backend/service/SchedulePublishedService.java =====
package simplex.bn25.centmerci.server.service;

import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import simplex.bn25.centmerci.server.dto.SchedulePublishedDTO;
import simplex.bn25.centmerci.server.entity.SchedulePublished;
import simplex.bn25.centmerci.server.repository.SchedulePublishedRepository;

import java.util.List;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
public class SchedulePublishedService {

    private final SchedulePublishedRepository repository;

    public List<SchedulePublishedDTO> getAllSchedules() {
        return repository.findAll().stream().map(this::toDTO).collect(Collectors.toList());
    }

    private SchedulePublishedDTO toDTO(SchedulePublished entity) {
        SchedulePublishedDTO dto = new SchedulePublishedDTO();
        dto.setId(entity.getId());
        dto.setGroupsName(entity.getGroupsName());
        dto.setDepartmentsName(entity.getDepartmentsName());
        dto.setLecturersName(entity.getLecturersName());
        dto.setCurriculumsName(entity.getCurriculumsName());
        dto.setRoomsName(entity.getRoomsName());
        dto.setCategoriesName(entity.getCategoriesName());
        dto.setBackgroundColor(entity.getBackgroundColor());
        dto.setStartDateTime(entity.getStartDateTime());
        dto.setEndDateTime(entity.getEndDateTime());
        return dto;
    }
}

===== backend/controller/SchedulePublishedController.java =====
package simplex.bn25.centmerci.server.controller;

import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.*;
import simplex.bn25.centmerci.server.dto.SchedulePublishedDTO;
import simplex.bn25.centmerci.server.service.SchedulePublishedService;

import java.util.List;

@RestController
@RequestMapping("/api/schedules")
@RequiredArgsConstructor
@CrossOrigin(origins = "*")
public class SchedulePublishedController {

    private final SchedulePublishedService service;

    @GetMapping("/published")
    public List<SchedulePublishedDTO> getPublishedSchedules() {
        return service.getAllSchedules();
    }
}

