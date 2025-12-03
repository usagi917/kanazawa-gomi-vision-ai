# 金沢市ごみ分別AI

写真からごみの種類を推定し、金沢市の収集区分にあわせて分別方法と収集日ルールを案内するシンプルなWebアプリです。ブラウザだけで動作し、スマホでも利用できます。

## できること
- 写真をアップロードまたは撮影してAI判定（プレビュー・リセット付き）
- 分別区分（例: 燃やすごみ / 燃やさないごみ / 資源）を表示
- 町名を選ぶと、その地域の収集日ルールを表示（未選択時は案内カードで設定を促す）
- 注意事項（分解が必要など）を合わせて提示


## 必要なもの
- モダンブラウザ（Chrome / Edge / Safari など）
- Google Apps Script で公開した Web API のURL（`GAS_API_URL`）
- 金沢市の分別辞書と収集カレンダーを格納したスプレッドシート

## フロントエンドのセットアップ（`index.html`）
1. `index.html` をエディタで開き、スクリプト冒頭の `GAS_API_URL` を自分のGASデプロイURLに置き換えます。
2. 保存するだけで準備完了です。ビルド工程や依存パッケージは不要です。

### 動作フロー（フロント）
- 起動時に `GET GAS_API_URL` を叩き、`dictionary` と `calendar` を取得してセレクトボックスを生成。
- 画像を選択すると Base64 に変換し、`POST GAS_API_URL`（Content-Type: `text/plain`）で `{ image, mimeType }` を送信。
- レスポンスの `item_name` を辞書に突合し、分別区分・注意書きを表示。地域選択がある場合はカレンダーのルール（`burn` / `nonburn` / `resource`）を表示。
- 地域未選択時は案内カードを表示し、設定を促します。

### GAS のデプロイ手順
1. Google スプレッドシートを用意し、下記2シートを作成。
   - `dictionary_raw`: ヘッダー行の後に「品目名(A) / 区分(C) / 注意書き(D)」。B列は未使用。
   - `calendar`: ヘッダー行の後に「地域名(A) / 燃やすごみ(J) / 燃やさないごみ(K) / 資源(L) / ビン(M)」。フロントでは `burn` `nonburn` `resource` を参照（`bottle`は現状未使用）。
2. スクリプト プロパティを設定。
   - `SHEET_ID`: スプレッドシートのID（URLの `/d/` と `/edit` の間）。
   - `GEMINI_KEY`: Gemini APIキー（Google AI Studio の API key）。
3. 下記「GASバックエンドコード」を Google Apps Script に貼り付けて保存。
4. デプロイ → 新しいデプロイ → 種類「ウェブアプリ」 → アクセス権「全員」または「全員（匿名ユーザー）」で公開し、発行された URL を `GAS_API_URL` に設定。

### GAS バックエンドコード
フロントが呼び出す API を提供するサンプルです。画像(Base64)を Gemini に送り、品目名を返します。

```javascript
// --- GASコード (バックエンド) ---

const CONFIG = {
  SHEET_ID: PropertiesService.getScriptProperties().getProperty('SHEET_ID'),
  API_KEY: PropertiesService.getScriptProperties().getProperty('GEMINI_KEY'),
  DICT_NAME: 'dictionary_raw', // 辞書シート名
  CAL_NAME: 'calendar',        // カレンダーシート名
  MODEL_NAME: 'models/gemini-2.5-flash'
};

function doGet(e) {
  const data = { dictionary: getDictionaryData(), calendar: getCalendarData() };
  return createOutput(data);
}

function doPost(e) {
  try {
    const params = JSON.parse(e.postData.contents); // text/plain をパース
    const imageBase64 = params.image;
    const mimeType = params.mimeType;

    const dict = getDictionaryData();
    const itemsList = dict.map(d => d.name).join(', ');
    const prompt = `
    この画像のゴミが、以下の「品目リスト」のどれに該当するか特定してください。
    [品目リスト]
    ${itemsList.substring(0, 30000)}
    [条件]
    最も近い item_name をリストから一つ選ぶこと。
    出力はJSON形式のみ: { "item_name": "選んだ品目名" }
    `;

    const result = callGeminiAPI(imageBase64, mimeType, prompt);
    return createOutput(result);

  } catch (err) {
    return createOutput({ error: err.message });
  }
}

function callGeminiAPI(base64, mime, prompt) {
  const url = `https://generativelanguage.googleapis.com/v1beta/${CONFIG.MODEL_NAME}:generateContent?key=${CONFIG.API_KEY}`;
  const payload = {
    contents: [{ parts: [{ text: prompt }, { inline_data: { mime_type: mime, data: base64 } }] }],
    generationConfig: { response_mime_type: "application/json" }
  };
  const res = UrlFetchApp.fetch(url, { method: 'post', contentType: 'application/json', payload: JSON.stringify(payload), muteHttpExceptions: true });
  const json = JSON.parse(res.getContentText());
  if (json.error) throw new Error(json.error.message);
  const text = json.candidates[0].content.parts[0].text;
  return JSON.parse(text.replace(/```json|```/g, '').trim());
}

function createOutput(data) {
  return ContentService.createTextOutput(JSON.stringify(data)).setMimeType(ContentService.MimeType.JSON);
}

function getDictionaryData() {
  const sheet = SpreadsheetApp.openById(CONFIG.SHEET_ID).getSheetByName(CONFIG.DICT_NAME);
  if (!sheet) return [];
  const rows = sheet.getDataRange().getValues();
  rows.shift();
  return rows.map(r => ({ name: r[0], type: r[2], note: r[3] })).filter(d => d.name);
}

function getCalendarData() {
  const sheet = SpreadsheetApp.openById(CONFIG.SHEET_ID).getSheetByName(CONFIG.CAL_NAME);
  if (!sheet) return [];
  const rows = sheet.getDataRange().getValues();
  rows.shift();
  return rows.map(r => ({ area: r[0], rules: { burn: r[9], nonburn: r[10], resource: r[11], bottle: r[12] } })).filter(d => d.area);
}
```

### GAS 側の期待レスポンス
- **GET `GAS_API_URL`** で初期データを返す
  - 例: `{ "dictionary": [{"name": "ペットボトル", "type": "資源", "note": "キャップとラベルを外す" }], "calendar": [{"area": "◯◯町", "rules": {"burn": "毎週火・金", "nonburn": "第2水曜", "resource": "第1・3木曜"}}] }`
- **POST `GAS_API_URL`** に画像をBase64(JSON)で送信すると、`{"item_name": "ペットボトル"}` のように判定結果を返す

### 画像送信フォーマット（POST）
```json
{
  "image": "<BASE64本体>",
  "mimeType": "image/jpeg"
}
```

## 使い方
1. ブラウザで `index.html` を開く（ダブルクリックでも可。CORSを避けたい場合は簡易サーバーで開くと安心）。
2. 「地域を選択」で町名を選ぶ（スキップ可）。
3. 「ゴミの写真を撮影」で画像を撮るか選ぶ。
4. 「AIで判定する」を押すと分別区分が表示され、地域を選んでいれば収集日ルールも表示されます。

