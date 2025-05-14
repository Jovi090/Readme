React グループ開発用テンプレート構築ガイド（空ページ＋ナビゲーション付き）

一、開発環境の準備
---------------------
1. Node.js と npm のインストール：
   https://nodejs.org/ から最新版をインストールしてください。

2. Vite CLI（オプション）をインストール：
   pnpm install -g create-vite

3. React プロジェクトの作成：
   pnpm create vite my-app --template react
   cd my-app
   pnpm install

4. ルーティング用ライブラリのインストール（react-router-dom）：
   pnpm add react-router-dom

二、ディレクトリ構成（おすすめ）
---------------------
src/
├── components/
│   ├── TopNavBar.jsx
│   └── Layout.jsx
├── pages/
│   ├── EditorPage.jsx
│   ├── SettingPage.jsx
│   ├── UserPage.jsx
│   └── Unauthorized.jsx（任意）
├── App.jsx
└── main.jsx

三、TopNavBar コンポーネント（TopNavBar.jsx）
---------------------
import React from 'react';
import { Link } from 'react-router-dom';
import './TopNavBar.css';

const TopNavBar = () => {
  return (
    <div className="top-nav">
      <div className="nav-left">
        <span className="brand">scheduler</span>
        <Link to="/editor">editor</Link>
        <Link to="/setting">setting</Link>
        <Link to="/user">user</Link>
      </div>
      <div className="nav-right">
        <Link to="/logout">logout</Link>
      </div>
    </div>
  );
};

export default TopNavBar;

TopNavBar.css スタイル：
---------------------
.top-nav {
  background-color: #204080;
  color: white;
  display: flex;
  justify-content: space-between;
  align-items: center;
  height: 40px;
  padding: 0 20px;
  font-family: sans-serif;
}

.nav-left {
  display: flex;
  align-items: center;
  gap: 20px;
  margin-left: 30px;
}

.nav-right {
  margin-right: 10px;
}

a {
  color: white;
  text-decoration: none;
  font-size: 14px;
}

a:hover {
  text-decoration: underline;
}

.brand {
  font-weight: bold;
  margin-right: 20px;
}

四、Layout コンポーネント（Layout.jsx）
---------------------
import React from 'react';
import { Outlet } from 'react-router-dom';
import TopNavBar from './TopNavBar';

const Layout = () => {
  return (
    <div>
      <TopNavBar />
      <div style={{ paddingTop: '50px' }}>
        <Outlet />
      </div>
    </div>
  );
};

export default Layout;

五、App のルーティング設定（App.jsx）
---------------------
import React from 'react';
import { BrowserRouter as Router, Routes, Route } from 'react-router-dom';
import Layout from './components/Layout';
import EditorPage from './pages/EditorPage';
import SettingPage from './pages/SettingPage';
import UserPage from './pages/UserPage';

function App() {
  return (
    <Router>
      <Routes>
        <Route path="/" element={<Layout />}>
          <Route path="editor" element={<EditorPage />} />
          <Route path="setting" element={<SettingPage />} />
          <Route path="user" element={<UserPage />} />
          <Route path="logout" element={<div>Logging out...</div>} />
        </Route>
      </Routes>
    </Router>
  );
}

export default App;

六、各ページの基本構成（EditorPage.jsx など）
---------------------
import React from 'react';

const EditorPage = () => {
  return (
    <div>
      <h2>Editor Page</h2>
      {/* 担当者がこの中にページロジックを記述 */}
    </div>
  );
};

export default EditorPage;

※SettingPage.jsx や UserPage.jsx も同様に構成します。

七、チーム開発のためのおすすめ協力方法
---------------------
- 各メンバーが自分の担当ページファイルを開発（EditorPage, SettingPage, UserPage）
- チームリーダーが構造と共通コンポーネントを準備
- Git（ブランチ開発）や VS Code Live Share で効率的に共同作業


八、FullCalendar を使ってリソースタイムライン＋ドラッグ可能なイベントパネルを追加（CSS スタイル付き）
---------------------

1. FullCalendar のパッケージをインストール：

pnpm add @fullcalendar/core @fullcalendar/react @fullcalendar/interaction @fullcalendar/resource @fullcalendar/resource-timeline

2. ドラッグ可能なイベントパネルの作成（DraggableEvents.jsx）：

```jsx
// src/components/DraggableEvents.jsx
import React, { useEffect } from 'react';
import { Draggable } from '@fullcalendar/interaction';
import '../styles/draggable-events.css'; // CSS をインポート

const DraggableEvents = () => {
  useEffect(() => {
    let containerEl = document.getElementById('external-events');
    new Draggable(containerEl, {
      itemSelector: '.fc-event',
      eventData: function (eventEl) {
        return {
          title: eventEl.innerText.trim()
        };
      }
    });
  }, []);

  return (
    <div id="external-events">
      <p><strong>イベントをカレンダーにドラッグ</strong></p>
      <div className="fc-event">会議A</div>
      <div className="fc-event">研修B</div>
    </div>
  );
};

export default DraggableEvents;
```

3. ドラッグ用 CSS（styles/draggable-events.css）：

```css
#external-events {
  padding: 10px;
  width: 200px;
  background: #eee;
  border-radius: 4px;
}

#external-events .fc-event {
  margin: 10px 0;
  padding: 5px;
  background: #3788d8;
  color: white;
  cursor: pointer;
  border-radius: 3px;
}
```

4. リソースカレンダーの作成（Calendar.jsx）：

```jsx
// src/components/Calendar.jsx
import React from 'react';
import FullCalendar from '@fullcalendar/react';
import resourceTimelinePlugin from '@fullcalendar/resource-timeline';
import interactionPlugin from '@fullcalendar/interaction';
import '../styles/calendar.css'; // CSS をインポート

const Calendar = () => {
  return (
    <div className="calendar-wrapper">
      <FullCalendar
        schedulerLicenseKey="GPL-My-Project-Is-Open-Source"
        plugins={[resourceTimelinePlugin, interactionPlugin]}
        initialView="resourceTimelineDay"
        editable={true}
        droppable={true}
        resources={[
          { id: 'a', title: 'リソースA' },
          { id: 'b', title: 'リソースB' }
        ]}
        events={[]}
        eventReceive={(info) => {
          console.log('外部イベント受信:', info.event);
        }}
        height="100%"
      />
    </div>
  );
};

export default Calendar;
```

5. カレンダー用 CSS（styles/calendar.css）：

```css
.calendar-wrapper {
  background: #fff;
  padding: 10px;
  border-radius: 4px;
  border: 1px solid #ddd;
  height: 600px;
}
```

6. EditorPage.jsx に両方組み込み、レイアウト CSS 使用：

```jsx
// src/pages/EditorPage.jsx
import React from 'react';
import Calendar from '../components/Calendar';
import DraggableEvents from '../components/DraggableEvents';
import '../styles/editor-page.css'; // CSS をインポート

const EditorPage = () => {
  return (
    <div className="editor-page">
      <div className="editor-sidebar">
        <DraggableEvents />
      </div>
      <div className="editor-calendar-area">
        <Calendar />
      </div>
    </div>
  );
};

export default EditorPage;
```

7. EditorPage 用 CSS（styles/editor-page.css）：

```css
.editor-page {
  display: flex;
  padding: 20px;
  gap: 20px;
}

.editor-sidebar {
  width: 220px;
  background: #f4f4f4;
  padding: 10px;
  border-radius: 4px;
}

.editor-calendar-area {
  flex-grow: 1;
  background: #fff;
  padding: 10px;
  border-radius: 4px;
  border: 1px solid #ddd;
}
```

このようにすれば、FullCalendar のすべての UI 部品が整理された CSS によって一括でスタイリングされ、拡張性と可読性の高い構成になります。





React グループ開発用テンプレート構築ガイド（空ページ＋ナビゲーション付き）

一、開発環境の準備
---------------------
1. Node.js と npm のインストール：
   https://nodejs.org/ から最新版をインストールしてください。

2. Vite CLI（オプション）をインストール：
   pnpm install -g create-vite

3. React プロジェクトの作成：
   pnpm create vite my-app --template react
   cd my-app
   pnpm install

4. ルーティング用ライブラリのインストール（react-router-dom）：
   pnpm add react-router-dom

二、ディレクトリ構成（おすすめ）
---------------------
src/
├── components/
│   ├── TopNavBar.jsx
│   └── Layout.jsx
├── pages/
│   ├── EditorPage.jsx
│   ├── SettingPage.jsx
│   ├── UserPage.jsx
│   └── Unauthorized.jsx（任意）
├── App.jsx
└── main.jsx

三、TopNavBar コンポーネント（TopNavBar.jsx）
---------------------
import React from 'react';
import { Link } from 'react-router-dom';
import './TopNavBar.css';

const TopNavBar = () => {
  return (
    <div className="top-nav">
      <div className="nav-left">
        <span className="brand">scheduler</span>
        <Link to="/editor">editor</Link>
        <Link to="/setting">setting</Link>
        <Link to="/user">user</Link>
      </div>
      <div className="nav-right">
        <Link to="/logout">logout</Link>
      </div>
    </div>
  );
};

export default TopNavBar;

TopNavBar.css スタイル：
---------------------
.top-nav {
  background-color: #204080;
  color: white;
  display: flex;
  justify-content: space-between;
  align-items: center;
  height: 40px;
  padding: 0 20px;
  font-family: sans-serif;
}

.nav-left {
  display: flex;
  align-items: center;
  gap: 20px;
  margin-left: 30px;
}

.nav-right {
  margin-right: 10px;
}

a {
  color: white;
  text-decoration: none;
  font-size: 14px;
}

a:hover {
  text-decoration: underline;
}

.brand {
  font-weight: bold;
  margin-right: 20px;
}

四、Layout コンポーネント（Layout.jsx）
---------------------
import React from 'react';
import { Outlet } from 'react-router-dom';
import TopNavBar from './TopNavBar';

const Layout = () => {
  return (
    <div>
      <TopNavBar />
      <div style={{ paddingTop: '50px' }}>
        <Outlet />
      </div>
    </div>
  );
};

export default Layout;

五、App のルーティング設定（App.jsx）
---------------------
import React from 'react';
import { BrowserRouter as Router, Routes, Route } from 'react-router-dom';
import Layout from './components/Layout';
import EditorPage from './pages/EditorPage';
import SettingPage from './pages/SettingPage';
import UserPage from './pages/UserPage';

function App() {
  return (
    <Router>
      <Routes>
        <Route path="/" element={<Layout />}>
          <Route path="editor" element={<EditorPage />} />
          <Route path="setting" element={<SettingPage />} />
          <Route path="user" element={<UserPage />} />
          <Route path="logout" element={<div>Logging out...</div>} />
        </Route>
      </Routes>
    </Router>
  );
}

export default App;

六、各ページの基本構成（EditorPage.jsx など）
---------------------
import React from 'react';

const EditorPage = () => {
  return (
    <div>
      <h2>Editor Page</h2>
      {/* 担当者がこの中にページロジックを記述 */}
    </div>
  );
};

export default EditorPage;

※SettingPage.jsx や UserPage.jsx も同様に構成します。

七、チーム開発のためのおすすめ協力方法
---------------------
- 各メンバーが自分の担当ページファイルを開発（EditorPage, SettingPage, UserPage）
- チームリーダーが構造と共通コンポーネントを準備
- Git（ブランチ開発）や VS Code Live Share で効率的に共同作業
