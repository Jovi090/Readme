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
