===== frontend/Calendar.tsx =====
import React, { useState, useRef, useEffect } from 'react';
import FullCalendar from '@fullcalendar/react';
import resourceTimelinePlugin from '@fullcalendar/resource-timeline';
import interactionPlugin from '@fullcalendar/interaction';
import EventEdit from './EventEditor';
import '../styles/calendar.css';
import { FaTrashAlt } from 'react-icons/fa';
import { ToastContainer, toast } from 'react-toastify';
import 'react-toastify/dist/ReactToastify.css';
import jaLocale from '@fullcalendar/core/locales/ja';
import { useEvents } from '../contexts/EventContext';
import { fetchDrafts, saveDraft, deleteDraft, publishDrafts } from '../api/draftApi';

type ResourceType = 'room' | 'group';

const Calendar: React.FC = () => {
    const [resourceType, setResourceType] = useState<ResourceType>('group');
    const [resources, setResources] = useState<{ id: string; title: string }[]>([]);
    const { events, setEvents } = useEvents(); // 状態管理フック
    const [selectedEventInfo, setSelectedEventInfo] = useState<{
        eventObj: any;
        plainObject: any;
    } | null>(null);
    const [showEdit, setShowEdit] = useState(false);
    const trashBinRef = useRef<HTMLDivElement>(null);
    const [showDeleteConfirm, setShowDeleteConfirm] = useState(false);
    const [eventToDelete, setEventToDelete] = useState<any>(null);
    const [isDraggingOver, setIsDraggingOver] = useState(false);

    // 初期データ読み込み（DBから草稿データを取得）
    useEffect(() => {
        const fetchData = async () => {
            try {
                const draftData = await fetchDrafts();
                const convertedEvents = draftData.map((item: any) => ({
                    id: item.id,
                    title: item.curriculumsName,
                    start: item.startDateTime,
                    end: item.endDateTime,
                    resourceId: item.groupsName,
                    backgroundColor: item.backgroundColor,
                    borderColor: item.backgroundColor,
                    extendedProps: {
                        department: item.departmentsName,
                        instructor: item.lecturersName,
                        duration: '',
                        location: item.roomsName
                    }
                }));
                setEvents(convertedEvents);

                const uniqueGroups = Array.from(new Set(draftData.map((e: any) => e.groupsName)));
                setResources(uniqueGroups.map(name => ({ id: name, title: name })));
            } catch (err) {
                toast.error("データ取得に失敗しました");
            }
        };
        fetchData();
    }, []);

    const handleEventClick = (clickInfo: any) => {
        setSelectedEventInfo({
            eventObj: clickInfo.event,
            plainObject: {
                id: clickInfo.event.id,
                title: clickInfo.event.title,
                start: clickInfo.event.start,
                end: clickInfo.event.end,
                backgroundColor: clickInfo.event.backgroundColor,
                borderColor: clickInfo.event.borderColor || clickInfo.event.backgroundColor,
                extendedProps: {
                    department: clickInfo.event.extendedProps?.department,
                    instructor: clickInfo.event.extendedProps?.instructor || '',
                    duration: clickInfo.event.extendedProps?.duration || '',
                    location: clickInfo.event.extendedProps?.location || ''
                },
                el: clickInfo.el
            }
        });
        setShowEdit(true);
    };

    const handleUpdateEvent = async (updatedEvent: any) => {
        if (!selectedEventInfo) return;
        const { eventObj } = selectedEventInfo;

        eventObj.setProp('title', updatedEvent.title);
        eventObj.setExtendedProp('department', updatedEvent.extendedProps?.department);
        eventObj.setExtendedProp('instructor', updatedEvent.extendedProps?.instructor);
        eventObj.setExtendedProp('duration', updatedEvent.extendedProps?.duration);
        eventObj.setExtendedProp('location', updatedEvent.extendedProps?.location);

        setEvents(prev =>
            prev.map(e => e.id === updatedEvent.id ? {
                ...e,
                title: updatedEvent.title,
                extendedProps: {
                    ...e.extendedProps,
                    ...updatedEvent.extendedProps
                }
            } : e)
        );

        await saveDraft({
            id: updatedEvent.id,
            curriculumsName: updatedEvent.title,
            startDateTime: updatedEvent.start,
            endDateTime: updatedEvent.end,
            groupsName: updatedEvent.resourceId,
            backgroundColor: updatedEvent.backgroundColor,
            departmentsName: updatedEvent.extendedProps?.department,
            lecturersName: updatedEvent.extendedProps?.instructor,
            roomsName: updatedEvent.extendedProps?.location,
            categoriesName: '一時カテゴリ'
        });

        setShowEdit(false);
        toast.success("保存しました");
    };

    const handleEventDragStop = (info: any) => {
        const target = info.jsEvent.target;
        if (trashBinRef.current?.contains(target)) {
            setEventToDelete(info.event);
            setShowDeleteConfirm(true);
        }
        setIsDraggingOver(false);
    };

    const confirmDelete = async () => {
        if (eventToDelete) {
            eventToDelete.remove();
            setEvents(prev => prev.filter(e => e.id !== eventToDelete.id));
            await deleteDraft(Number(eventToDelete.id));
        }
        setShowDeleteConfirm(false);
        setEventToDelete(null);
        setIsDraggingOver(false);
    };

    const cancelDelete = () => {
        setShowDeleteConfirm(false);
        setEventToDelete(null);
        setIsDraggingOver(false);
    };

    const checkFormComplete = (eventData: any) => {
        const props = eventData.extendedProps || {};
        return (
            (eventData.instructor || props.instructor) &&
            (eventData.duration || props.duration) &&
            (eventData.location || props.location)
        );
    };

    const handleSaveClick = () => {
        if (!selectedEventInfo) return;
        const isComplete = checkFormComplete(selectedEventInfo.plainObject.extendedProps);
        if (!isComplete) {
            toast.warning('未入力項目があります');
        } else {
            handleUpdateEvent(selectedEventInfo.plainObject);
        }
    };

    const handleGroupSwitch = () => {
        setResourceType(prev => prev === 'room' ? 'group' : 'room');
    };

    const handlePublish = async () => {
        try {
            await publishDrafts();
            toast.success("公開しました");
        } catch {
            toast.error("公開に失敗しました");
        }
    };

    return (
        <div className="calendar-container">
            <div ref={trashBinRef} className={`trash-bin ${isDraggingOver ? 'drag-over' : ''}`}>
                <FaTrashAlt className="trash-icon" />
                <span>削除するにはここにドラッグ</span>
            </div>

            <ToastContainer position="top-center" autoClose={3000} hideProgressBar />

            <div className="calendar-wrapper">
                <FullCalendar
                    locale="ja"
                    locales={[jaLocale]}
                    schedulerLicenseKey="CC-Attribution-NonCommercial-NoDerivatives"
                    plugins={[resourceTimelinePlugin, interactionPlugin]}
                    initialView="resourceTimelineDay"
                    editable={true}
                    droppable={true}
                    eventDragStop={handleEventDragStop}
                    firstDay={1}
                    events={events}
                    resources={resources}
                    resourceAreaWidth="20%"
                    resourceAreaHeaderContent={resourceType === 'room' ? '会場' : 'クラス'}
                    slotMinTime="09:00:00"
                    slotMaxTime="18:00:00"
                    eventClick={handleEventClick}
                    height="100%"
                    headerToolbar={{
                        left: 'updateBtn uploadBtn prev,next validationBtn viewBtn',
                        center: 'title',
                        right: 'resourceTimelineDay,resourceTimelineWeek,resourceTimelineMonth'
                    }}
                    customButtons={{
                        viewBtn: {
                            text: '切替',
                            click: handleGroupSwitch
                        },
                        updateBtn: {
                            text: '保存',
                            click: handleSaveClick
                        },
                        validationBtn: {
                            text: '▲',
                            click: handleSaveClick
                        },
                        uploadBtn: {
                            text: '公開',
                            click: handlePublish
                        }
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
                />
                {showDeleteConfirm && (
                    <div className="custom-confirm-modal">
                        <div className="confirm-modal-content">
                            <h3>イベントの削除</h3>
                            <p>このイベントを削除してもよろしいですか？</p>
                            <div className="confirm-buttons">
                                <button onClick={confirmDelete}>はい</button>
                                <button onClick={cancelDelete}>いいえ</button>
                            </div>
                        </div>
                    </div>
                )}
                {showEdit && selectedEventInfo && (
                    <EventEdit
                        event={selectedEventInfo.plainObject}
                        onclose={() => setShowEdit(false)}
                        onSave={handleUpdateEvent}
                        resources={resources}
                    />
                )}
            </div>
        </div>
    );
};

export default Calendar;

===== frontend/api/draftApi.ts =====
import axios from 'axios';

// 草稿一覧を取得
export const fetchDrafts = async () => {
    const res = await axios.get('/api/schedules/draft');
    return res.data;
};

// 草稿を保存
export const saveDraft = async (draft: any) => {
    const res = await axios.post('/api/schedules/draft', draft);
    return res.data;
};

// 草稿を削除
export const deleteDraft = async (id: number) => {
    await axios.delete(`/api/schedules/draft/${id}`);
};

// 草稿を公開
export const publishDrafts = async () => {
    await axios.post('/api/schedules/draft/publish');
};

===== frontend/contexts/EventContext.tsx =====
import React, { createContext, useContext, useState } from 'react';

// イベント状態用の型定義
interface EventContextType {
    events: any[];
    setEvents: React.Dispatch<React.SetStateAction<any[]>>;
}

const EventContext = createContext<EventContextType | undefined>(undefined);

// Provider
export const EventProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
    const [events, setEvents] = useState<any[]>([]);
    return (
        <EventContext.Provider value={{ events, setEvents }}>
            {children}
        </EventContext.Provider>
    );
};

// フックとして使用
export const useEvents = () => {
    const context = useContext(EventContext);
    if (!context) {
        throw new Error('useEvents must be used within an EventProvider');
    }
    return context;
};

===== backend/controller/ScheduleDraftController.java =====
package simplex.bn25.centmerci.server.controller;

import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.*;
import simplex.bn25.centmerci.server.entity.ScheduleDraft;
import simplex.bn25.centmerci.server.service.ScheduleDraftService;

import java.util.List;

@RestController
@RequestMapping("/api/schedules/draft")
@RequiredArgsConstructor
@CrossOrigin(origins = "*")
public class ScheduleDraftController {

    private final ScheduleDraftService draftService;

    @GetMapping
    public List<ScheduleDraft> getAllDrafts() {
        return draftService.getDrafts();
    }

    @PostMapping
    public ScheduleDraft saveDraft(@RequestBody ScheduleDraft draft) {
        return draftService.saveDraft(draft);
    }

    @DeleteMapping("/{id}")
    public void deleteDraft(@PathVariable Long id) {
        draftService.deleteDraft(id);
    }

    @PostMapping("/publish")
    public void publish() {
        draftService.publishDrafts();
    }
}

===== backend/service/ScheduleDraftService.java =====
package simplex.bn25.centmerci.server.service;

import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import simplex.bn25.centmerci.server.entity.ScheduleDraft;
import simplex.bn25.centmerci.server.entity.SchedulePublished;
import simplex.bn25.centmerci.server.repository.ScheduleDraftRepository;
import simplex.bn25.centmerci.server.repository.SchedulePublishedRepository;

import java.time.LocalDateTime;
import java.util.List;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
public class ScheduleDraftService {

    private final ScheduleDraftRepository draftRepo;
    private final SchedulePublishedRepository publishedRepo;

    public List<ScheduleDraft> getDrafts() {
        return draftRepo.findAll();
    }

    public ScheduleDraft saveDraft(ScheduleDraft draft) {
        return draftRepo.save(draft);
    }

    public void deleteDraft(Long id) {
        draftRepo.deleteById(id);
    }

    public void publishDrafts() {
        List<ScheduleDraft> drafts = draftRepo.findAll();

        for (ScheduleDraft draft : drafts) {
            if (draft.getCurriculumsName() == null || draft.getStartDateTime() == null || draft.getEndDateTime() == null) {
                throw new IllegalStateException("未入力の項目があります。公開できません。");
            }
        }

        publishedRepo.deleteAll();

        List<SchedulePublished> published = drafts.stream().map(d -> {
            SchedulePublished p = new SchedulePublished();
            p.setGroupsName(d.getGroupsName());
            p.setDepartmentsName(d.getDepartmentsName());
            p.setLecturersName(d.getLecturersName());
            p.setCurriculumsName(d.getCurriculumsName());
            p.setRoomsName(d.getRoomsName());
            p.setCategoriesName(d.getCategoriesName());
            p.setBackgroundColor(d.getBackgroundColor());
            p.setStartDateTime(d.getStartDateTime());
            p.setEndDateTime(d.getEndDateTime());
            p.setCreatedDateTime(LocalDateTime.now());
            return p;
        }).collect(Collectors.toList());

        publishedRepo.saveAll(published);
    }
}

===== backend/repository/ScheduleDraftRepository.java =====
package simplex.bn25.centmerci.server.repository;

import org.springframework.data.jpa.repository.JpaRepository;
import simplex.bn25.centmerci.server.entity.ScheduleDraft;

public interface ScheduleDraftRepository extends JpaRepository<ScheduleDraft, Long> {
}

===== backend/entity/ScheduleDraft.java =====
package simplex.bn25.centmerci.server.entity;

import jakarta.persistence.*;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;
import java.time.LocalDateTime;

@Entity
@Table(name = "schedule_draft")
@Data
@NoArgsConstructor
@AllArgsConstructor 
public class ScheduleDraft {

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

