
======== 后端 – 原始课程 (Curriculum) ========

1. Curriculum.java
-------------------
package com.yourapp.model;

import javax.persistence.*;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;
import java.time.LocalDateTime;

@Entity
@Table(name = "curriculums")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Curriculum {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;
    private String description;

    // 可按需扩展字段，如 duration, category 等
}


2. CurriculumRepository.java
-------------------
package com.yourapp.repository;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import com.yourapp.model.Curriculum;

@Repository
public interface CurriculumRepository extends JpaRepository<Curriculum, Long> {
    // 默认 CRUD 方法
}


3. CurriculumService.java
-------------------
package com.yourapp.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import com.yourapp.model.Curriculum;
import com.yourapp.repository.CurriculumRepository;
import java.util.List;

@Service
public class CurriculumService {

    @Autowired
    private CurriculumRepository repo;

    /** 获取所有课程模板 */
    public List<Curriculum> getAllCurriculums() {
        return repo.findAll();
    }

    /** 创建新课程模板 */
    public Curriculum createCurriculum(Curriculum curriculum) {
        return repo.save(curriculum);
    }

    /** 更新课程模板 */
    public Curriculum updateCurriculum(Long id, Curriculum curriculum) {
        curriculum.setId(id);
        return repo.save(curriculum);
    }

    /** 删除课程模板 */
    public void deleteCurriculum(Long id) {
        repo.deleteById(id);
    }
}


4. CurriculumController.java
-------------------
package com.yourapp.controller;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;
import com.yourapp.model.Curriculum;
import com.yourapp.service.CurriculumService;
import java.util.List;

@RestController
@RequestMapping("/api/curriculums")
@CrossOrigin(origins = "http://localhost:3030")
public class CurriculumController {

    @Autowired
    private CurriculumService service;

    /** 获取所有课程模板 */
    @GetMapping
    public List<Curriculum> getCurriculums() {
        return service.getAllCurriculums();
    }

    /** 创建课程模板 */
    @PostMapping
    public Curriculum createCurriculum(@RequestBody Curriculum curriculum) {
        return service.createCurriculum(curriculum);
    }

    /** 更新课程模板 */
    @PutMapping("/{id}")
    public Curriculum updateCurriculum(@PathVariable Long id, @RequestBody Curriculum curriculum) {
        return service.updateCurriculum(id, curriculum);
    }

    /** 删除课程模板 */
    @DeleteMapping("/{id}")
    public void deleteCurriculum(@PathVariable Long id) {
        service.deleteCurriculum(id);
    }
}


======== 前端 – React + Axios (Curriculum API) ========

1. api.ts
-------------------
import axios from 'axios';

const API = axios.create({ baseURL: 'http://localhost:8080/api/curriculums' });

// 原始课程 (Curriculum) 相关接口
export const getCurriculums = () => API.get('/');
export const createCurriculum = (curriculum) => API.post('/', curriculum);
export const updateCurriculum = (id, curriculum) => API.put(`/${id}`, curriculum);
export const deleteCurriculum = (id) => API.delete(`/${id}`);


2. React 组件示例 (加载左侧课程栏)
-------------------
import React, { useEffect, useState } from 'react';
import { getCurriculums } from './api';

export default function CurriculumList() {
  const [items, setItems] = useState([]);

  useEffect(() => {
    getCurriculums().then(res => {
      setItems(res.data);
    });
  }, []);

  return (
    <div className="curriculum-list">
      {items.map(cur => (
        <div
          key={cur.id}
          className="fc-event"
          data-curriculum={JSON.stringify(cur)}
        >
          {cur.title}
        </div>
      ))}
    </div>
  );
}




======== 后端（Spring Boot + JPA + PostgreSQL） ========

1. application.yml
-------------------
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/calendar_db
    username: postgres
    password: your_password
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    database-platform: org.hibernate.dialect.PostgreSQLDialect
  jackson:
    serialization:
      write-dates-as-timestamps: false
server:
  port: 8080


2. EventDraft.java
-------------------
@Entity
@Table(name = "event_draft")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class EventDraft {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;
    private LocalDateTime startTime;
    private LocalDateTime endTime;
    private String description;
    private boolean allDay;
    private Long originEventId;
}


3. EventPublished.java
-------------------
@Entity
@Table(name = "event_published")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class EventPublished {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;
    private LocalDateTime startTime;
    private LocalDateTime endTime;
    private String description;
    private boolean allDay;
    private Long publishedBy;
    private LocalDateTime publishedAt = LocalDateTime.now();
}


4. EventDraftRepository.java
-------------------
@Repository
public interface EventDraftRepository extends JpaRepository<EventDraft, Long> {
}


5. EventPublishedRepository.java
-------------------
@Repository
public interface EventPublishedRepository extends JpaRepository<EventPublished, Long> {
}


6. EventService.java
-------------------
@Service
public class EventService {

    @Autowired
    private EventDraftRepository draftRepo;

    @Autowired
    private EventPublishedRepository pubRepo;

    public List<EventDraft> getDrafts() {
        return draftRepo.findAll();
    }

    public EventDraft saveDraft(EventDraft draft) {
        return draftRepo.save(draft);
    }

    public void deleteDraft(Long id) {
        draftRepo.deleteById(id);
    }

    public List<EventPublished> getPublished() {
        return pubRepo.findAll();
    }

    public void publishDrafts() {
        List<EventDraft> drafts = draftRepo.findAll();
        List<EventPublished> published = drafts.stream().map(d -> {
            EventPublished p = new EventPublished();
            p.setTitle(d.getTitle());
            p.setStartTime(d.getStartTime());
            p.setEndTime(d.getEndTime());
            p.setDescription(d.getDescription());
            p.setAllDay(d.isAllDay());
            p.setPublishedBy(1L); // 模拟用户
            p.setPublishedAt(LocalDateTime.now());
            return p;
        }).toList();
        pubRepo.saveAll(published);
    }
}


7. EventController.java
-------------------
@RestController
@RequestMapping("/api/events")
@CrossOrigin(origins = "http://localhost:3000")
public class EventController {

    @Autowired
    private EventService service;

    @GetMapping("/drafts")
    public List<EventDraft> getDrafts() {
        return service.getDrafts();
    }

    @PostMapping("/drafts")
    public EventDraft createDraft(@RequestBody EventDraft draft) {
        return service.saveDraft(draft);
    }

    @PutMapping("/drafts/{id}")
    public EventDraft updateDraft(@PathVariable Long id, @RequestBody EventDraft draft) {
        draft.setId(id);
        return service.saveDraft(draft);
    }

    @DeleteMapping("/drafts/{id}")
    public void deleteDraft(@PathVariable Long id) {
        service.deleteDraft(id);
    }

    @PostMapping("/publish")
    public void publish() {
        service.publishDrafts();
    }

    @GetMapping("/published")
    public List<EventPublished> getPublished() {
        return service.getPublished();
    }
}


======== 前端（React + Axios + FullCalendar） ========

1. api.ts
-------------------
import axios from 'axios';

const API = axios.create({ baseURL: 'http://localhost:8080/api/events' });

export const getDrafts = () => API.get('/drafts');
export const createDraft = (event) => API.post('/drafts', event);
export const updateDraft = (id, event) => API.put(`/drafts/${id}`, event);
export const deleteDraft = (id) => API.delete(`/drafts/${id}`);
export const publishEvents = () => API.post('/publish');
export const getPublished = () => API.get('/published');


2. useCalendar.ts (伪代码)
-------------------
useEffect(() => {
  getDrafts().then(res => setEvents(res.data));
}, []);

const onEventDrop = (info) => {
  const updated = {
    ...info.event.extendedProps,
    title: info.event.title,
    startTime: info.event.start.toISOString(),
    endTime: info.event.end.toISOString()
  };
  updateDraft(info.event.id, updated);
};

const handlePublish = () => {
  publishEvents().then(() => alert("Published!"));
};

FullCalendar 事件钩子绑定到 onEventDrop，按钮绑定 handlePublish 即可。

