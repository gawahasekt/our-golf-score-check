# ゴルフスコア アプリ — 設計ドキュメント

---

## 1. 要件定義

### 1-1. プロジェクト概要

| 項目 | 内容 |
|---|---|
| アプリ名 | ゴルフスコア |
| 形態 | シングルHTMLファイル（ビルド不要） |
| 対象 | スマートフォン優先、PCブラウザでも動作 |
| ホスティング | Netlify（静的ファイル配信） |
| リアルタイムDB | Firebase Realtime Database |

---

### 1-2. 機能要件

#### FR-01 セッション管理
- アプリ起動時に UUID v4 のセッションIDを自動生成し、URLパラメータ `?session=<UUID>` として付与する
- 既存セッションIDはURLパラメータ → localStorage の順に優先して読み込む
- 「URLをコピー」ボタンで現在のセッションURLをクリップボードにコピーできる
- 「新しいラウンド」ボタンで新セッションを生成し、過去データは元URLから引き続き参照できる

#### FR-02 コース管理
- コース名・ホール数・各ホールPARをFirebase共有DB（`courses/`）に保存できる
- コース名の部分一致検索で保存済みコースを絞り込み表示できる（最大8件）
- 「適用」ボタンでコースデータを現在のセッションに反映できる
- 「×」ボタンで保存済みコースを削除できる
- コースデータは全ユーザーで共有される

#### FR-03 ホール数・PAR設定
- 9ホール / 18ホールを選択できる
- 各ホールのPARを個別に設定できる（PAR 3〜5）
- 18ホール時はOUT（1〜9番）/ IN（10〜18番）に分けて表示する
- 設定変更は即時Firebase & localStorageに保存される

#### FR-04 メンバー管理
- メンバー名（最大10文字）を登録できる
- メンバーを削除するとそのメンバーのスコアも連動して削除される
- メンバーIDはUnixタイムスタンプ（`Date.now()`）で生成する

#### FR-05 スコア入力（カウンター）
- 対象メンバー・対象ホールをタブ/丸ボタンで選択できる
- 「＋」「−」ボタンで打数を増減できる（最小0 = 未入力扱い）
- 数字部分タップでキーボード直接入力モードに切り替えられる（0〜20の範囲）
- カウンターの初期表示値はそのホールのPAR
- 「＋」「−」ボタン操作時点で即座に保存される
- 「決定 ›」ボタンで次のホールへ移動（未入力の場合はPARとして記録）
- 入力済みホールは丸ボタンに緑ドットが表示される

#### FR-06 スコア表示
- ホール×メンバーの打数とPAR差（+/−）を一覧表示する
- 18ホール時はOUT/IN小計行を表示する
- 合計行はスコア合計とPAR差を表示する
- スコアランク（Eagle以上 / Birdie / Par / Bogey / Double Bogey / +3以上）を色で区別する

#### FR-07 スコアラベル表示
| 打数 / PAR差 | ラベル |
|---|---|
| 1打でホールイン | Hole-in-one! |
| PAR比 −4以下 | Condor |
| PAR比 −3 | Albatross |
| PAR比 −2 | Eagle |
| PAR比 −1 | Birdie |
| PAR比 ±0 | Par |
| PAR比 +1 | Bogey |
| PAR比 +2 | Double Bogey |
| PAR比 +3以上 | （表示なし） |

#### FR-08 データ管理
- 「スコアのみクリア」：スコアデータのみ削除、メンバー・PAR設定は保持
- 「全データをリセット」：メンバー・スコア・設定をすべて削除

#### FR-09 CSV出力
- スコア表をCSV形式でダウンロードできる
- ファイル名：`golf-score-YYYYMMDD.csv`
- UTF-8 BOM付き（日本語Excel対応）
- ホール別スコア・OUT/IN小計（18H時）・合計を含む

#### FR-10 リアルタイム同期
- 同じセッションURLを開いた複数端末間でデータをリアルタイムに同期する
- Firebase `onValue` でデータ変更を監視し、自端末の書き込みエコーはスキップする

#### FR-11 オフライン対応
- Firebase接続が切れていてもlocalStorageに保存し、入力を継続できる
- 再接続時にFirebaseへ自動同期される
- ヘッダーの接続ドット（●）でオンライン/オフライン状態を視覚的に確認できる

---

### 1-3. 非機能要件

| 分類 | 要件 |
|---|---|
| 対応端末 | スマートフォン（iOS/Android）・PCブラウザ |
| 最大幅 | 480px（中央寄せ） |
| ダブルタップズーム | 無効（`touch-action: manipulation`） |
| セキュリティ | Firebase Security Rules により `sessions/$sessionId` と `courses` のみ読み書き許可 |
| セッション推測困難性 | UUID128ビットにより総当たり攻撃を実質不可能にする |
| インストール | 不要（ブラウザで直接動作） |

---

## 2. 画面設計

### 2-1. 画面一覧

```mermaid
block-beta
  columns 1
  header["⛳ ゴルフスコア  ● [接続状態]  [?]"]
  tabs["① メンバー　｜　② カウント　｜　③ スコア"]
  block:screens:3
    s1["① メンバー画面"]
    s2["② カウント画面"]
    s3["③ スコア画面"]
  end
  modal["ヘルプモーダル（オーバーレイ）"]
```

---

### 2-2. ① メンバー画面 レイアウト

```mermaid
block-beta
  columns 1
  A["📋 ラウンド共有カード\n─────────────────\nセッションURL表示\n[URLをコピー]　[新しいラウンド]"]
  B["🏌️ コース選択カード\n─────────────────\n検索テキストボックス\nコースリスト（名前 ホール数 合計PAR）\n  [適用] [×]\n─────────────────\n現在の設定を保存\n[コース名入力] [保存]"]
  C["🕳️ ホール数カード\n─────────────────\n[9ホール]　[18ホール]"]
  D["📐 PAR設定カード\n─────────────────\nOUT(1〜9番)  1 2 3 4 5 6 7 8 9\n             ▼ ▼ ▼ ▼ ▼ ▼ ▼ ▼ ▼\nIN(10〜18番) 10 ... 18"]
  E["👥 メンバー登録カード\n─────────────────\n[名前入力] [追加]\nメンバー一覧\n  名前　[削除]"]
  F["[スコアのみクリア]　[全データをリセット]"]
```

---

### 2-3. ② カウント画面 レイアウト

```mermaid
block-byzantine
  columns 1
  A["👥 メンバー選択カード\n─────────────────\n[田中] [山田] [鈴木] ...  （横スクロール）"]
  B["🕳️ ホール選択カード\n─────────────────\n●1 ●2 ●3 ... ●9  （横スクロール・入力済みは緑ドット）"]
  C["🎯 カウンターカード\n─────────────────\n　田中　3番ホール\n─────────────────\n[−]　　　5　　　[＋]\n　　　　PAR 4\n　　　　Birdie\n─────────────────\n[　　決定　›　　]"]
```

---

### 2-4. ③ スコア画面 レイアウト

```mermaid
block-beta
  columns 1
  A["📊 スコアテーブルカード\n─────────────────────────────────\n H  │ PAR │ 田中 │ 山田 │ 鈴木\n──┼─────┼──────┼──────┼──────\n 1  │  4  │ 5 +1 │ 3 -1 │ 4 ±0\n 2  │  3  │ 2 -1 │  —   │ 3 ±0\n ... \n OUT│ 36  │  38  │  —   │  36\n ... \n合計│ 72  │  74  │  —   │  72"]
  B["[CSVダウンロード]"]
```

---

## 3. 画面遷移図

```mermaid
stateDiagram-v2
    [*] --> 初期化 : アプリ起動
    初期化 --> メンバー画面 : セッション確立\nlocalStorage読込

    state メンバー画面 {
        [*] --> メンバー一覧
        メンバー一覧 --> メンバー一覧 : メンバー追加/削除\nPAR変更\nホール数変更\nコース適用/保存/削除
        メンバー一覧 --> 確認ダイアログ : スコアのみクリア\n全データリセット\n新しいラウンド\nコース削除
        確認ダイアログ --> メンバー一覧 : OK / キャンセル
    }

    state カウント画面 {
        [*] --> カウンター
        カウンター --> カウンター : ＋/− ボタン\nメンバー切替\nホール切替\n数字直接入力
        カウンター --> カウンター : 決定 › （次ホールへ）
    }

    state スコア画面 {
        [*] --> スコア一覧
        スコア一覧 --> CSV保存 : CSVダウンロード
        CSV保存 --> スコア一覧
    }

    メンバー画面 --> カウント画面 : ② タブをタップ
    メンバー画面 --> スコア画面  : ③ タブをタップ
    カウント画面 --> メンバー画面 : ① タブをタップ
    カウント画面 --> スコア画面  : ③ タブをタップ
    スコア画面  --> メンバー画面 : ① タブをタップ
    スコア画面  --> カウント画面 : ② タブをタップ

    メンバー画面 --> ヘルプモーダル : ? ボタン
    カウント画面 --> ヘルプモーダル : ? ボタン
    スコア画面  --> ヘルプモーダル : ? ボタン
    ヘルプモーダル --> メンバー画面 : × / 背景タップ（元の画面に戻る）
    ヘルプモーダル --> カウント画面 : × / 背景タップ
    ヘルプモーダル --> スコア画面  : × / 背景タップ
```

---

## 4. 処理フロー

### 4-1. アプリ初期化フロー

```mermaid
flowchart TD
    Start([アプリ起動]) --> LS[localStorageからG読込]
    LS --> P1{URLパラメータに\nsession=?}
    P1 -->|あり| UseURL[URLのsessionIdを使用]
    P1 -->|なし| P2{localStorageに\ngolf_sid?}
    P2 -->|あり| UseLS[localStorageのsessionIdを使用]
    P2 -->|なし| NewID[crypto.randomUUID()で生成]
    UseURL --> UpdateURL[URL & localStorageを更新]
    UseLS  --> UpdateURL
    NewID  --> UpdateURL
    UpdateURL --> ShowURL[セッションURL表示]
    ShowURL --> StartSync[Firebase onValueリスナー登録]
    StartSync --> LoadCourses[コースDB読込]
    LoadCourses --> BuildUI[UI構築\nbuildParInputs / buildMemberList]
    BuildUI --> End([準備完了])
```

---

### 4-2. スコア入力フロー（カウンター）

```mermaid
flowchart TD
    Start([カウント画面を開く]) --> HasMember{メンバー登録済み?}
    HasMember -->|なし| ShowMsg[案内メッセージ表示]
    HasMember -->|あり| SelectMember[メンバー選択\nctrMid = 選択ID]
    SelectMember --> SelectHole[ホール選択\nctrHole = 選択番号]
    SelectHole --> ShowCounter[カウンター表示\n初期値 = pars&#91;ctrHole&#93;]

    ShowCounter --> PlusMinus{＋/−ボタン操作}
    PlusMinus --> Calc[打数 = 現在値 ± 1\n最小0 = 未入力扱い]
    Calc --> Save1[Firebase & localStorage保存]
    Save1 --> Dot[ホールドットを緑に更新]
    Dot --> ShowDiff[スコアラベル表示]
    ShowDiff --> PlusMinus

    ShowCounter --> TapNum{数字タップ}
    TapNum --> EditMode[直接入力モード\nキーボード表示]
    EditMode --> Blur[フォーカスアウト / Enter]
    Blur --> ClampVal[値クランプ 0〜20]
    ClampVal --> Save2[Firebase & localStorage保存]
    Save2 --> ShowDiff

    ShowDiff --> Confirm{決定 › タップ}
    Confirm --> Recorded{スコア記録済み?}
    Recorded -->|なし| SetPar[PAR値を記録]
    Recorded -->|あり| NextHole{最終ホール?}
    SetPar --> NextHole
    NextHole -->|No| MoveNext[ctrHole++]
    NextHole -->|Yes| Stay[現在ホールに留まる]
    MoveNext --> SelectHole
    Stay --> ShowCounter
```

---

### 4-3. Firebase リアルタイム同期フロー

```mermaid
sequenceDiagram
    participant A as 端末A
    participant FB as Firebase RTDB
    participant B as 端末B

    Note over A,B: 同じ ?session=UUID でアクセス

    A->>FB: onValue(sessions/UUID) 登録
    B->>FB: onValue(sessions/UUID) 登録

    A->>A: stroke(+1) 実行
    A->>FB: set(sessions/UUID, G) 書き込み
    FB-->>A: onValue イベント発火
    A->>A: JSON比較: incoming === G?\n→ 同一のためスキップ
    FB-->>B: onValue イベント発火
    B->>B: JSON比較: incoming ≠ G\n→ G を更新 & refreshUI()
    B->>B: localStorage にも保存

    Note over A,B: オフライン時

    A->>A: stroke(+1) 実行
    A->>A: localStorage に保存
    A--xFB: Firebase write 失敗（キュー）
    A->>A: 再接続検知
    A->>FB: キューの書き込みを送信
    FB-->>B: onValue イベント発火
    B->>B: G を更新 & refreshUI()
```

---

### 4-4. コース保存・適用フロー

```mermaid
flowchart TD
    subgraph 保存
        S1([「保存」ボタンタップ]) --> S2{コース名入力済み?}
        S2 -->|なし| S3[フォーカス移動]
        S2 -->|あり| S4[push courses/ に書き込み\nname, holes, pars, savedAt]
        S4 --> S5[入力欄クリア\n完了アラート]
        S4 --> S6[onValue により全端末のリストが自動更新]
    end

    subgraph 適用
        A1([「適用」ボタンタップ]) --> A2[確認ダイアログ]
        A2 -->|キャンセル| A3[何もしない]
        A2 -->|OK| A4[G.holes, G.pars を上書き]
        A4 --> A5[save: Firebase & localStorage]
        A5 --> A6[buildParInputs 再描画]
    end

    subgraph 削除
        D1([「×」ボタンタップ]) --> D2[確認ダイアログ]
        D2 -->|キャンセル| D3[何もしない]
        D2 -->|OK| D4[remove courses/id]
        D4 --> D5[onValue により全端末のリストが自動更新]
    end
```

---

### 4-5. CSV出力フロー

```mermaid
flowchart TD
    Start([CSVダウンロード]) --> Check{メンバー登録済み?}
    Check -->|なし| Alert[アラート表示]
    Check -->|あり| Header["ヘッダー行生成\n[ホール, PAR, 名前1, 名前2, ...]"]
    Header --> Rows["ホール行生成（1〜n）\n打数が未入力は空文字"]
    Rows --> Sub18{18ホール?}
    Sub18 -->|Yes| SubRows[OUT / IN 小計行追加]
    Sub18 -->|No| Total
    SubRows --> Total[合計行追加]
    Total --> BOM[BOM付きUTF-8でBlob生成]
    BOM --> DL["<a>要素クリック\ngolf-score-YYYYMMDD.csv"]
    DL --> End([ダウンロード完了])
```

---

### 4-6. データ状態管理

```mermaid
stateDiagram-v2
    direction LR
    [*] --> 未保存 : アプリ初期状態

    state "セッションデータ (G)" as G {
        未保存 --> 保存済み : save() 呼び出し
        保存済み --> 保存済み : save() 呼び出し（更新）
        保存済み --> リセット済み : resetAll() / clearScores()
        リセット済み --> 保存済み : save() 呼び出し
    }

    state "保存先" as Dest {
        Firebase : sessions/&#123;sessionId&#125;
        LS : localStorage\ngolfApp_v1
    }

    保存済み --> Firebase : set() 非同期
    保存済み --> LS : setItem() 同期
```

---

## 5. データ構造

### 5-1. セッションデータ（Firebase `sessions/{sessionId}` & localStorage `golfApp_v1`）

```json
{
  "holes": 9,
  "pars": [4, 3, 4, 5, 4, 4, 3, 4, 5],
  "members": [
    { "id": 1715000000000, "name": "田中" },
    { "id": 1715000000001, "name": "山田" }
  ],
  "scores": {
    "1715000000000_0": 5,
    "1715000000000_1": 3,
    "1715000000001_0": 4
  }
}
```

> キー形式：`{memberId}_{holeIndex}`（0始まり）

---

### 5-2. コースデータ（Firebase `courses/{pushId}`）

```json
{
  "name": "○○カントリークラブ",
  "holes": 18,
  "pars": [4, 3, 5, 4, 4, 3, 4, 5, 4, 4, 3, 4, 5, 4, 4, 3, 4, 5],
  "savedAt": 1715000000000
}
```

---

## 6. Firebase セキュリティルール

```json
{
  "rules": {
    "sessions": {
      "$sessionId": {
        ".read": true,
        ".write": true
      }
    },
    "courses": {
      ".read": true,
      ".write": true
    },
    "$other": {
      ".read": false,
      ".write": false
    }
  }
}
```

---

## 7. 技術スタック

```mermaid
graph LR
    Browser["ブラウザ\n(iOS Safari / Android Chrome / PC)"]
    HTML["index.html\nHTML + CSS + JS\n(シングルファイル)"]
    Firebase["Firebase\nRealtime Database"]
    Netlify["Netlify\n(静的ホスティング)"]
    LS["localStorage\n(オフラインキャッシュ)"]

    Browser -->|アクセス| Netlify
    Netlify -->|配信| HTML
    HTML -->|ES module import| Firebase
    HTML -->|読み書き| LS
    Firebase -->|onValue リアルタイム同期| HTML
```
