// ✅ React Frontend: Calendar.tsx
import React, { useState, useRef, useEffect } from 'react';
import FullCalendar from '@fullcalendar/react';
import resourceTimelinePlugin from '@fullcalendar/resource-timeline';
import interactionPlugin from '@fullcalendar/interaction';
import EventEdit from './EventEditor';
import '../styles/calendar.css';
 // ゴミ箱アイコンを読み込む
import { useEvents } from '../contexts/EventContext';
// import { useSettings } from '../contexts/SettingsContext';
type ResourceType = 'room' | 'group';
const Calendar: React.FC = () => {
    const [resourceType, setResourceType] = useState<ResourceType>('room');
    const [resources, setResources] = useState<{ id: string; title: string }[]>([]);
    const fetchData = async (type: ResourceType) => {
        const data = mockData[type];
        setResources(data.resources);
        // バックエンドと繋ぐ場合
        /*
        const res = await fetch(`/calendar-data?type=${type}`);
        const data = await res.json();
        setResources(data.resources);
        setEvents(data.events);
        */
    };
// 状態管理：すべてのイベント、選択されたイベント、編集パネルの表示状態を制御
    const { events, setEvents, publish } = useEvents();
// 現在選択されているイベントの情報
    const [selectedEventInfo, setSelectedEventInfo] = useState<{
        eventObj: any; // FullCalendar のネイティブイベントオブジェクト
        plainObject: any; // 通常のオブジェクト形式
    } | null>(null);
    // 編集パネルの表示・非表示を制御
    const [showEdit, setShowEdit] = useState(false);
    // ゴミ箱DOM要素の参照を取得
    // ゴミ箱にドラッグされたときのホバー状態を制御
    // const { classes } = useSettings();
    // FullCalendar の resource 形式に変換
    // const resources = classes.map(cls => ({
    //     id: String(cls.id),
    //     title: `Group ${cls.name}`, // "Group X"
    // }));
    // イベントクリック時
 const handleEventClick = (clickInfo: any) => {
    console.log("クリックされたイベントのデータを確認する:", clickInfo.event);
    // 取得したデータを状態にセットする
    setSelectedEventInfo({
        eventObj: clickInfo.event,
        plainObject: {
            id: clickInfo.event.id,
            title: clickInfo.event.title,
            start: clickInfo.event.start,
            end: clickInfo.event.end,
            // resourceId: clickInfo.event.getResources()[0]?.id,
            backgroundColor: clickInfo.event.backgroundColor,
            borderColor: clickInfo.event.borderColor || clickInfo.event.backgroundColor,
            // カスタム属性（FullCalendar標準でないもの）。上記は標準プロパティ
            extendedProps: {
                department: clickInfo.event.extendedProps?.department,
                // 初期値は空。設定データを取得してからここを更新する必要がある
                instructor: clickInfo.event.extendedProps?.instructor || '',
                duration: clickInfo.event.extendedProps?.duration || '',
                location: clickInfo.event.extendedProps?.location || ''
            },
            el: clickInfo.el
        }
    });
    // クリックでポップアップを表示
    setShowEdit(true);
};
// ドロップで新しいイベントを作成する関数
const handleEventReceive = (info: any) => {
    console.log("ドロップ後のイベントデータを確認する:", info);
    const droppedResId = info.event.getResources()[0]?.id;
    let start = info.event.start;
    let end = info.event.end;
    // --- 時刻処理ロジック ---
    // start/endが「時刻なし」（日付のみ）の場合、デフォルトで09:00～18:00に設定
    if (start && 
        start.getHours() === 0 && 
        start.getMinutes() === 0 && 
        start.getSeconds() === 0) {
        // 日付のみ（時刻が00:00:00）の場合、09:00に設定
        start = new Date(start);
        start.setHours(9, 0, 0, 0); // 09:00
        end = new Date(start);
        end.setHours(18, 0, 0, 0); // 18:00
    } 
    else if (!start) {
        // start自体がない場合、現在の日付09:00～18:00を設定
        start = new Date();
        start.setHours(9, 0, 0, 0);
        end = new Date(start);
        end.setHours(18, 0, 0, 0);
    } 
    else if (!end) {
        // startのみ、endがない場合、startから9時間後をendに設定
        end = new Date(start);
        end.setHours(start.getHours() + 9);
    }
// 新しいイベントオブジェクトを作成
    const newEvent = {
        id: String(Date.now()), // 一意なIDを生成
        title: info.event.title,
        start: start,    // 開始日時（必ず時刻を含む）
        end: end,        // 終了日時（必ず時刻を含む）
        resourceId: droppedResId, // リソース（クラス）ID,
        backgroundColor: info.event.backgroundColor,
        borderColor: info.event.backgroundColor,
        extendedProps: {
            department: info.event.extendedProps?.department || '',
            instructor: info.event.extendedProps?.instructor || '',
            duration: info.event.extendedProps?.duration || '',
            location: info.event.extendedProps?.location || ''
        }
    };
    // イベントリストに追加
    setEvents(prev => [...prev, newEvent]);
};
// イベントのプロパティを更新。フォームが変更されたらここも更新する必要がある
const handleUpdateEvent = (updatedEvent: any) => {
    if (!selectedEventInfo) return;
    const { eventObj } = selectedEventInfo;
    eventObj.setProp('title', updatedEvent.title);
    eventObj.setExtendedProp('department', updatedEvent.extendedProps?.department);
    eventObj.setExtendedProp('instructor', updatedEvent.extendedProps?.instructor);
    eventObj.setExtendedProp('duration', updatedEvent.extendedProps?.duration);
    eventObj.setExtendedProp('location', updatedEvent.extendedProps?.location);
 // ローカルの events ステートを更新する
    setEvents(prevEvents =>
        prevEvents.map(event =>
            event.id === updatedEvent.id
                ? {
                    ...event,
                    title: updatedEvent.title,
                    extendedProps: {
                        ...event.extendedProps,
                        ...updatedEvent.extendedProps
                    }
                }
                : event
        )
    );
    // ポップアップを閉じる
    setShowEdit(false);
};
    setIsDraggingOver(true);
};
// イベントのドラッグが終了したときの処理関数
// setIsDraggingOver(true);
const target= info.jsEvent.target;
if(trashBinRef.current?.contains(target)){
setEventToDelete(info.event);
setShowDeleteConfirm(true);
}
setIsDraggingOver(false)
};
// イベントの削除を確認する
const confirmDelete=()=>{
  if(eventToDelete){
    eventToDelete.remove();
    setEvents(prev =>prev.filter(e =>e.id !== eventToDelete.id));
  }
  setShowDeleteConfirm(false);
  setEventToDelete(null);
  setIsDraggingOver(false);
};
// 削除をキャンセルする
// フオ-ムがすべて入カされているか確認する
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
        console.log(selectedEventInfo.plainObject.extendedProps);
        if (!isComplete) {
            toast.warning('未入力項目があります', {
                position: "top-center",
                autoClose: 3000,
            });
        } else {
            handleUpdateEvent(selectedEventInfo.plainObject); // 调用更新函数
            toast.success('更新が完了しました', {
                position: "top-center",
                autoClose: 2000,
            });
        }
    };
 const handleEventResize = (info: any) => {
//Handle event resize logic here if needed
    };
    useEffect(() => {
        fetchData(resourceType);
    }, [resourceType]);
    const handleGroupSwitch = () => {
        setResourceType(prev => prev === 'room' ? 'group' : 'room');
    };
 return (
    <div className="calendar-container">
      {/* ゴミ箱エリア */}
      <ToastContainer
      position="top-center"
      autoClose={3000}
      hideProgressBar
      newestOnTop
      closeOnClick
      pauseOnFocusLoss
      draggable
      pauseOnHover
    />
      <div className="calendar-wrapper">
        <FullCalendar
          locale="ja"
          locales={[jaLocale]}
          schedulerLicenseKey="CC-Attribution-NonCommercial-NoDerivatives"
          plugins={[resourceTimelinePlugin, interactionPlugin]}
          initialView="resourceTimelineDay"
           eventResize={handleEventResize}
          editable={true}
          droppable={true}
          firstDay={1}
          events={events}
          resourceAreaWidth="20%"
          resources={resources}
          eventMaxStack={2}
          resourceAreaHeaderContent={resourceType === 'room' ? '会場' : 'クラス'}
          slotMinTime="09:00:00"
          slotMaxTime="18:00:00"
          eventReceive={handleEventReceive}
          eventClick={handleEventClick}
          height="100%"
          headerToolbar={{
              left: 'updateBtn uploadBtn prev,next validationBtn viewBtn',  
              center: 'title',
              right: 'resourceTimelineDay,resourceTimelineWeek,resourceTimelineMonth'
            }}
            customButtons={{
              viewBtn: {  
                text: 'Change',
                click: handleGroupSwitch,
              },
              updateBtn: {
                text: '更新',
                click: handleSaveClick,
              },
              validationBtn: {
                text: '▲',
                click: handleSaveClick,
              },
              uploadBtn: {  // 补充缺失的配置
                text: '公開',
                click: () => {
                publish();
                toast.success('公開されました', {
                  position: "top-center",
                  autoClose: 2000,  
                  hideProgressBar: true,
                  closeOnClick: true,
                  pauseOnHover: true,
                  draggable: true,
                  progress: undefined
                });
              }
            }
          }}
        views={{
          resourceTimelineDay: {  
            buttonText: '日'
          },
          resourceTimelineMonth: {
            buttonText: '月'
          },
          resourceTimelineWeek: {
            buttonText: '週',
            dayHeaderFormat: {
              weekday: 'short',
              day: 'numeric'
            },
            dayHeaders: true,
            type: 'resourceTimeline',  
            duration: { weeks: 1 },
            slotDuration: { days: 1 },
            slotLabelFormat: [{
              weekday: 'short',
              day: 'numeric'
            }
          ]
          }
          }}
eventContent={(arg) => ({
  html: `
    <div class="fc-event-container">
      <div class="fc-event-main">${arg.event.title}</div>  
    </div>
  `
})}
/>
      {/*イベント編集パネル */}
      {showEdit && selectedEventInfo &&(
      <EventEdit
      event={selectedEventInfo.plainObject}
      onclose={()=>setShowEdit(false)}
      onSave={handleUpdateEvent}
      resources={resources}
        />
      )}
      </div>
    </div>
  );
};
export default Calendar;



// ✅ Java Backend Code (Spring Boot)

// ✅ ScheduleDraft.java
@Entity
@Table(name = "schedules_draft")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class ScheduleDraft {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String groupsName;
    private String departmentsName;
    private String lecturersName;
    private String curriculumsName;
    private String roomsName;
    private String categoriesName;
    private String backgroundColor;

    private LocalDateTime startDatetime;
    private LocalDateTime endDatetime;
    private LocalDateTime createdDatetime;
}

// ✅ ScheduleDraftRepository.java
public interface ScheduleDraftRepository extends JpaRepository<ScheduleDraft, Long> {}

// ✅ ScheduleDraftController.java
@RestController
@RequestMapping("/api/schedules/draft")
@CrossOrigin
public class ScheduleDraftController {
    private final ScheduleDraftRepository repository;

    public ScheduleDraftController(ScheduleDraftRepository repository) {
        this.repository = repository;
    }

    @GetMapping
    public List<ScheduleDraft> getAll() {
        return repository.findAll();
    }

    @DeleteMapping("/all")
    public void deleteAll() {
        repository.deleteAll();
    }

    @PostMapping("/bulk")
    public void saveAll(@RequestBody List<ScheduleDraft> drafts) {
        repository.saveAll(drafts);
    }
}

