# CLAUDE.md — タスクトラッカー開発ガイド

## 1. アーキテクチャ・技術スタック

### ファイル構成
- **単一HTMLファイル構成**（`index.html` のみ）
- CSS は `<style>` タグにインライン記述
- JavaScript は `<script type="module">` タグにインライン記述
- 外部ファイル・ビルドツールは使用しない

### 使用ライブラリ・CDN
| ライブラリ | バージョン | 用途 |
|---|---|---|
| Tailwind CSS | CDN（Play CDN） | UIスタイリング全般 |
| Firebase App | 12.12.1 | Firebase初期化 |
| Firebase Firestore | 12.12.1 | データ永続化（クラウド） |
| Firebase Auth | 12.12.1 | Google認証 |

```html
<script src="https://cdn.tailwindcss.com"></script>
```
```javascript
import { initializeApp } from 'https://www.gstatic.com/firebasejs/12.12.1/firebase-app.js';
import { getFirestore, doc, setDoc, getDoc, onSnapshot } from 'https://www.gstatic.com/firebasejs/12.12.1/firebase-firestore.js';
import { getAuth, GoogleAuthProvider, signInWithRedirect, getRedirectResult, signOut, onAuthStateChanged } from 'https://www.gstatic.com/firebasejs/12.12.1/firebase-auth.js';
```

### Firebase の使い方

**認証**
- Google アカウントによるログイン（`signInWithRedirect` 使用）
- iOS Safari / ホーム画面PWA対応のため `signInWithPopup` ではなく `signInWithRedirect` を採用
- `onAuthStateChanged` で認証状態を監視し、ログイン前はログイン画面を表示

**Firestore**
- ログイン後、`onSnapshot` でリアルタイム同期
- データは常にFirestoreに保存（ローカルのみの保存モードなし）
- localStorage はキャッシュ兼オフラインバッファとして併用

**Firestoreセキュリティルール**
```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId}/{document=**} {
      allow read, write: if request.auth != null
                         && request.auth.uid == userId;
    }
  }
}
```

### データ構造

**Firestoreパス**
```
users/{uid}/todo-tracker/main
```

**ドキュメント構造**
```javascript
{
  tasks: Task[],
  categories: string[]
}
```

**Taskオブジェクト**
```javascript
{
  id:        string,      // Date.now().toString()
  name:      string,      // タスク名
  category:  string,      // カテゴリ名（categoriesの要素）
  priority:  3 | 2 | 1 | null,  // 高・中・低・未設定
  deadline:  string | null,     // "YYYY-MM-DD" または null
  progress:  number,      // 0〜100（%）、デフォルト0
  memo:      string,      // メモ、デフォルト""
  createdAt: string       // "YYYY-MM-DD"
}
```

**localStorageキー**
```javascript
'todo-tracker-tasks'           // Task[] のJSONキャッシュ
'todo-tracker-categories'      // string[] のJSONキャッシュ
'todo-tracker-form-collapsed'  // フォーム折りたたみ状態（"true"/"false"）
```

---

## 2. デザイン方針

### カラーパレット（Tailwindのグレー系スレートをベースに使用）
| 用途 | 値 |
|---|---|
| ページ背景 | `bg-gray-100`（#f3f4f6） |
| カード・セクション背景 | `bg-white` |
| ボーダー | `border-gray-200`（#e5e7eb） |
| プライマリテキスト | `text-gray-800` / `text-gray-700` |
| セカンダリテキスト | `text-gray-500` / `text-gray-400` |
| ミュート・プレースホルダー | `text-gray-300` |
| アクション（ボタン・フォーカス） | `slate-700`（#334155）/ `slate-400` |
| アクティブタブ下線 | `border-slate-700` |

### 優先度バッジカラー
| 優先度 | クラス |
|---|---|
| 3（高） | `bg-red-50 text-red-600 border border-red-100` |
| 2（中） | `bg-yellow-50 text-yellow-600 border border-yellow-100` |
| 1（低） | `bg-blue-50 text-blue-600 border border-blue-100` |

### プログレスバーカラー（締切・進捗共通の色思想）
| 状態 | カラーコード |
|---|---|
| 余裕あり / 完了 | `#22c55e`（green） |
| やや余裕あり | `#84cc16`（lime） |
| 注意 | `#f97316`（orange） |
| 危険 / 超過 | `#ef4444`（red） |
| 進捗0〜29% | `#94a3b8`（slate） |
| 進捗30〜59% | `#f97316`（orange） |
| 進捗60〜99% | `#3b82f6`（blue） |
| 進捗100% | `#22c55e`（green） |

### 基本スタイル（カスタムCSS）
```css
body { font-family: 'Segoe UI', system-ui, -apple-system, sans-serif; }
input, select, textarea { font-size: 16px !important; } /* iOS自動ズーム防止 */
```

### 主要コンポーネントクラス

| クラス名 | 用途 |
|---|---|
| `.icon-btn` | 28×28px の円形アイコンボタン（編集・削除など） |
| `.progress-track` | プログレスバーの背景トラック（height: 4px） |
| `.progress-fill` | プログレスバーの塗り部分（width/color をインラインで指定） |
| `.overdue-pulse` | 期限超過時のフェードアニメーション |
| `.task-row` | テーブル行（hover時に`#f9fafb`） |
| `.pencil-icon` | ホバー時に表示される鉛筆アイコン（通常は`opacity:0`） |
| `.editable-cell` | クリックで編集モードに入るセル |
| `.tab-btn` | カテゴリタブボタン |

### レイアウト
- 最大幅: `max-w-5xl`（1024px）、中央揃え
- セクション間: `space-y-4`
- カード: `bg-white rounded-2xl shadow-sm border border-gray-200`
- モーダル背景: `rgba(0,0,0,0.2)` + `backdrop-filter:blur(4px)`

### モバイル対応
- テーブルは `overflow-x-auto` + `min-width:680px` で横スクロール
- `.pencil-icon` はモバイルで常時 `opacity:0.4`（ホバー不可のため）
- フォームはモバイルで縦積みレイアウト
- iOS ホーム画面アプリ（PWA）として動作可能

---

## 3. コーディング規則

### 言語
- **UIテキスト・ラベル・コメント・`alert`・`confirm`**: 日本語
- **変数名・関数名**: 英語（キャメルケース）
- **コードコメント**: セクション区切りに `// ─── セクション名 ───` 形式を使用

### 状態管理
- グローバル変数でシンプルに管理（Reactなどのフレームワーク不使用）
- `<script type="module">` 内にスコープを閉じ、必要な関数のみ `window` に公開

```javascript
Object.assign(window, {
  functionName, anotherFunction, // onclick属性から呼ぶものだけ公開
});
```

### DOM更新の方針
- 状態変更後は `innerHTML` を使って丸ごと再描画（仮想DOMなし）
- テーブル行・タブ・カテゴリリストはすべて `innerHTML` で再構築
- インライン編集中は対象セルだけ編集UI、それ以外は通常表示で再描画

### HTMLエスケープ
- テンプレートリテラルでHTML生成する際は必ず `esc()` を通す

```javascript
function esc(s) {
  return String(s).replace(/&/g,'&amp;').replace(/</g,'&lt;')
    .replace(/>/g,'&gt;').replace(/"/g,'&quot;').replace(/'/g,'&#39;');
}
```

### 非同期処理
- `async/await` を使用
- Firebase Auth操作は `try/catch` でエラーをキャッチし `alert` で通知
- Firestoreの `setDoc` はfire-and-forget（awaitしない）

```javascript
async function signInWithGoogle() {
  try {
    await signInWithRedirect(auth, provider);
  } catch(e) {
    console.error('サインインエラー:', e);
    alert('サインインに失敗しました。もう一度お試しください。');
  }
}
```

### エラーハンドリング
- ユーザー操作起点（サインイン・サインアウト）: `try/catch` + `alert`
- バックグラウンド処理（`getRedirectResult`など）: `.catch(e => console.error(...))`
- `localStorage` の読み込み: `try/catch` で握りつぶし（データがなくても動作継続）

### バリデーション
- 必須フィールド（タスク名・カテゴリ名）は入力時に確認
- 不正時は対象inputに一時的に `ring-2 ring-red-400` を付与（1.5秒後に解除）
- 破壊的操作（削除）は `confirm()` で確認

### 日付の扱い
- 保存形式: `"YYYY-MM-DD"`（文字列）
- タイムゾーン問題を避けるため `new Date(str + 'T00:00:00')` でローカル時刻として解析
- 今日の日付: `new Date().toISOString().split('T')[0]`

---

## 4. ファイル構成の方針

### 単一ファイル構成
外部依存（バンドラー・node_modules）なし。GitHub Pages に `index.html` 一枚でデプロイ。

```
todo-tracker/
├── index.html       # アプリ全体
├── logo.png         # ヘッダー絵文字の代わりにiOSアイコンとして使用
└── .github/
    └── workflows/
        └── deploy.yml   # main push → GitHub Pages 自動デプロイ
```

### GitHub Pages デプロイ
- ブランチ: `main`
- Source: GitHub Actions（`deploy.yml`）
- `actions/upload-pages-artifact` + `actions/deploy-pages` を使用

### 新機能を追加する際の手順
1. `index.html` の `// ─── State ───` セクションに必要な変数を追加
2. Taskオブジェクトに新フィールドを追加する場合は `addTask()` のデフォルト値も更新
3. テーブルに列を追加する場合は `<colgroup>` の `<col>` も追加
4. 新しい関数を `onclick` から呼ぶ場合は `Object.assign(window, {...})` に追加
5. `saveData()` を呼べばlocalStorage + Firestoreに自動保存される

### Firebase プロジェクト情報
- プロジェクトID: `todo-tracker-25501`
- 認証: Google Sign-In（`signInWithRedirect`）
- データベース: Cloud Firestore
- ホスティング: GitHub Pages（Firebaseホスティングは不使用）
- 承認済みドメイン: `ryoxsakai.github.io`（要確認: 追加済みか）
