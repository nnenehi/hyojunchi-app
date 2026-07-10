# Power Apps アプリ作成ガイド（画面の手作り手順）

## なぜ自動生成のアプリは「機能しない」のか

Power Apps の「Excel から開始」で自動生成されるのは、選んだ **1テーブル分の汎用アプリ**
（一覧画面・詳細画面・編集フォームの3画面）だけです。
元のHTMLアプリのような「樹種を選ぶ → 径級ボタンをタップ → 1本記録」という専用UIは
自動では作られません。**Excelファイルはデータの器として使い、画面は以下の手順で手作りします。**

所要時間の目安は30〜60分です。数式はすべてコピペできます。

---

## 0. 事前準備

1. make.powerapps.com →「作成」→「**空のアプリ**」→「空のキャンバスアプリ」→ 形式は「**電話**」
2. 左端の「データ」アイコン →「データの追加」→ 接続から **OneDrive for Business**（または OneDrive）を選ぶ
3. `標準地調査_PowerApps用データ.xlsx` を選び、**5つのテーブルすべて**
   （調査地・毎木記録・樹種マスタ・径級マスタ・区分マスタ）にチェックを入れて接続する
4. 左のツリービューで「App」を選び、**OnStart** プロパティに次を貼り付ける:

```
Set(varSurvey, First(調査地));
Set(varSp, First(SortByColumns(樹種マスタ, "表示順")).樹種名);
Set(varKubun, "残す")
```

5. 「App」を右クリック →「OnStart を実行」を1回実行しておく

> **注意**: 数式内のテーブル名・列名はExcelのテーブル定義と一致している必要があります。
> Power Apps が列名を `'径級cm'` のように引用符付きで補完することがありますが、そのままで問題ありません。

---

## 1. 記録画面（メイン画面）

`Screen1` の名前を `scrRecord` に変えて、上から順に以下を配置します。

### 1-1. 調査地ラベル（画面上部）

- **ラベル** を挿入 → 名前 `lblSurvey`
- **Text**:
```
If(IsBlank(varSurvey), "調査地未設定 — 調査地画面で作成してください",
  varSurvey.山林名 & " / " & varSurvey.林班小班 & " / " & varSurvey.地点 &
  "（" & CountRows(Filter(毎木記録, 調査地ID = varSurvey.調査地ID)) & "本記録）")
```

### 1-2. 樹種チップ（横スクロールのギャラリー）

- **水平ギャラリー** を挿入 → 名前 `galSpecies`、高さ 60 程度
- **Items**: `SortByColumns(樹種マスタ, "表示順")`
- ギャラリー内に **ボタン** を1つ置き（テンプレート内）:
  - **Text**: `ThisItem.樹種名`
  - **OnSelect**: `Set(varSp, ThisItem.樹種名)`
  - **Fill**: `If(ThisItem.樹種名 = varSp, RGBA(45,90,61,1), White)`
  - **Color**: `If(ThisItem.樹種名 = varSp, White, RGBA(34,48,42,1))`
  - **BorderColor**: `RGBA(45,90,61,1)`、**BorderThickness**: 1

### 1-3. 残す / 伐倒トグル（ボタン2つ）

- **ボタン** `btnKeep`:
  - **Text**: `"残す"`
  - **OnSelect**: `Set(varKubun, "残す")`
  - **Fill**: `If(varKubun = "残す", RGBA(45,90,61,1), White)`
  - **Color**: `If(varKubun = "残す", White, Gray)`
- **ボタン** `btnCut`（隣に配置）:
  - **Text**: `"伐倒"`
  - **OnSelect**: `Set(varKubun, "伐倒")`
  - **Fill**: `If(varKubun = "伐倒", RGBA(179,58,58,1), White)`
  - **Color**: `If(varKubun = "伐倒", White, Gray)`

### 1-4. 備考入力

- **テキスト入力** を挿入 → 名前 `txtMemo`
- **Default**: `""`、**HintText**: `"備考（例: 二又、曲がり）"`

### 1-5. 径級ボタングリッド（5列ギャラリー）— アプリの心臓部

- **縦方向ギャラリー** を挿入 → 名前 `galDia`
- **Items**: `SortByColumns(径級マスタ, "表示順")`
- 右側の設定パネルで **折り返しの数（WrapCount）を 5** にする
- テンプレート内に **ボタン** `btnDia` を1つ置き、テンプレートいっぱいに広げる:
  - **Text**: `ThisItem.径級cm`
  - **OnSelect**:
```
If(IsBlank(varSurvey),
  Notify("先に調査地画面で標準地を作成してください", NotificationType.Warning),
  Patch(毎木記録, Defaults(毎木記録), {
    記録ID: GUID(),
    調査地ID: varSurvey.調査地ID,
    No: CountRows(Filter(毎木記録, 調査地ID = varSurvey.調査地ID)) + 1,
    樹種: varSp,
    径級cm: ThisItem.径級cm,
    区分: varKubun,
    備考: txtMemo.Text,
    記録日時: Now()
  });
  Reset(txtMemo);
  Notify(varSp & " " & ThisItem.径級cm & "cm を「" & varKubun & "」で記録",
    NotificationType.Success, 1500)
)
```

> `GUID()` がExcelの記録ID（テキスト列）に入らずエラーになる場合は `GUID()` を
> `Text(Now(), "yyyymmddhhmmss") & Text(Rand()*1000, "000")` に置き換えてください。

### 1-6. その他の径級（62cm以上）

- **テキスト入力** `txtCustomDia`（**Format**: 数値）と **ボタン** `btnCustomDia` を横に並べる
- `btnCustomDia` の **OnSelect** は 1-5 の数式の `ThisItem.径級cm` を
  `Round(Value(txtCustomDia.Text)/2, 0) * 2` に置き換えたもの＋最後に `Reset(txtCustomDia)`

### 1-7. 直前の取り消しボタン

- **ボタン** `btnUndo`:
  - **Text**: `"↩ 直前の記録を取り消す"`
  - **OnSelect**:
```
With({r: Last(SortByColumns(Filter(毎木記録, 調査地ID = varSurvey.調査地ID), "No"))},
  If(IsBlank(r),
    Notify("取り消せる記録がありません", NotificationType.Warning),
    Remove(毎木記録, r);
    Notify("取り消しました", NotificationType.Information, 1200)
  )
)
```

---

## 2. 調査地画面

新しい画面 `scrSurvey` を追加します。

### 2-1. 新規作成フォーム

- **テキスト入力** を3つ: `txtYama`（山林名）、`txtRinpan`（林班小班）、`txtChiten`（地点）
- **ボタン** `btnCreateSurvey`:
  - **Text**: `"＋ 新しい標準地を作成して開始"`
  - **OnSelect**:
```
Set(varSurvey,
  Patch(調査地, Defaults(調査地), {
    調査地ID: "S" & Text(Now(), "yymmddhhmmss"),
    山林名: txtYama.Text,
    林班小班: txtRinpan.Text,
    地点: txtChiten.Text,
    作成日: Today()
  })
);
Reset(txtYama); Reset(txtRinpan); Reset(txtChiten);
Navigate(scrRecord)
```

### 2-2. 保存済み標準地の切り替え

- **縦方向ギャラリー** `galSurveys` → **Items**: `調査地`
- テンプレート内のラベル **Text**:
  `ThisItem.山林名 & " / " & ThisItem.林班小班 & " / " & ThisItem.地点`
- ギャラリーの **OnSelect**（または切替ボタン）:
  `Set(varSurvey, ThisItem); Navigate(scrRecord)`

---

## 3. 一覧・修正画面

新しい画面 `scrList` を追加します。

- **縦方向ギャラリー** `galRecords`:
  - **Items**:
```
SortByColumns(Filter(毎木記録, 調査地ID = varSurvey.調査地ID), "No", SortOrder.Descending)
```
  - テンプレートにラベルを並べる: `ThisItem.No`、`ThisItem.径級cm & "cm"`、`ThisItem.樹種`、`ThisItem.区分`、`ThisItem.備考`
  - 区分ラベルの **Color**: `If(ThisItem.区分 = "伐倒", RGBA(179,58,58,1), RGBA(45,90,61,1))`
- 修正は **編集フォーム** を使うのが簡単です:
  - **編集フォーム** `frmEdit` を挿入 → **DataSource**: `毎木記録`、**Item**: `galRecords.Selected`
  - 表示フィールド: 径級cm / 樹種 / 区分 / 備考
  - 保存ボタンの **OnSelect**: `SubmitForm(frmEdit)`
  - 削除ボタンの **OnSelect**: `Remove(毎木記録, galRecords.Selected)`

---

## 4. 集計画面

新しい画面 `scrSummary` を追加します。

- **縦方向ギャラリー** `galSummary` → **Items**:
```
AddColumns(
  GroupBy(Filter(毎木記録, 調査地ID = varSurvey.調査地ID), 樹種, g),
  残す本数, CountRows(Filter(g, 区分 = "残す")),
  伐倒本数, CountRows(Filter(g, 区分 = "伐倒")),
  計, CountRows(g),
  間伐率, Round(CountRows(Filter(g, 区分 = "伐倒")) / CountRows(g) * 100, 1)
)
```
- テンプレートにラベル5つ: `ThisItem.樹種` / `ThisItem.残す本数` / `ThisItem.伐倒本数` / `ThisItem.計` / `ThisItem.間伐率 & "%"`
- 全体の間伐率ラベル（ギャラリーの外）:
```
With({t: Filter(毎木記録, 調査地ID = varSurvey.調査地ID)},
  "全体: 残" & CountRows(Filter(t, 区分="残す")) &
  " / 伐" & CountRows(Filter(t, 区分="伐倒")) &
  "　間伐率 " & If(CountRows(t)=0, "0",
    Text(Round(CountRows(Filter(t, 区分="伐倒")) / CountRows(t) * 100, 1))) & "%")
```

---

## 5. 画面下のタブナビゲーション

各画面の下端に **ボタン** を4つ横に並べます（全画面にコピペ）:

| Text | OnSelect |
|---|---|
| 📝 記録 | `Navigate(scrRecord)` |
| 📊 集計 | `Navigate(scrSummary)` |
| 📋 一覧 | `Navigate(scrList)` |
| 🌲 調査地 | `Navigate(scrSurvey)` |

---

## トラブルシューティング

| 症状 | 原因と対処 |
|---|---|
| 数式で「名前が認識されません」 | データ接続に5テーブル全部を追加したか確認。テーブル名は `毎木記録` のようにExcelのテーブル名そのまま |
| Patch してもExcelに行が増えない | Excelファイルを**自分がPCで開いたままにしていない**か確認（ロックされます）。また保存反映には数秒かかります |
| `__PowerAppsId__` という列が勝手に増えた | 正常です。Power Apps が行を識別するための列なので**削除しないでください** |
| 委任（delegation）の警告が出る | Excel接続では正常です。1調査地あたり数百本程度なら実害はありません（既定で500行、設定で2000行まで） |
| 山の圏外で保存できない | Excel接続はオンライン必須です。圏外運用が必要なら `SaveData`/`LoadData` の実装か Dataverse 移行が必要です |
| 記録IDにGUIDが入らずエラー | `GUID()` を `Text(Now(),"yyyymmddhhmmss") & Text(Rand()*1000,"000")` に変更 |
