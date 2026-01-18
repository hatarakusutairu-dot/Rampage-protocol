# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

RAMPAGE PROTOCOL is a browser-based 3D multiplayer cooperative game where players work together to stop a rampaging giant robot. The entire application is contained in a single `index.html` file.

## Technology Stack

- **Three.js (r128)**: 3D rendering via CDN
- **Supabase**: Real-time multiplayer synchronization and database
- **Vanilla JavaScript**: No build tools or frameworks

## Development

Open `index.html` directly in a browser. No build step required.

## Architecture

### Game Flow
1. Title screen → Team creation/join → Lobby (role selection) → 3D Game

### Multiplayer System
- Teams identified by 4-digit codes
- Supabase tables: `teams`, `players`, `game_states`, `npc_states`
- Real-time sync via Supabase Realtime subscriptions (`subscribeToTeam()`)

### 3D World (Three.js)
- First-person camera with WASD movement + mouse look
- Robot with animated parts (`lArmGrp`, `rArmGrp`, `headMesh`)
- Interactive objects: 3 control panels (A, B, main), 6 NPCs

### Game State
Global `G` object holds all game state:
- `time`, `rampage`, `score`, `rescued`, `recovery`
- `panelA`, `panelB`, `panelMain` (puzzle progression)

### Core Functions
- `initThreeJS()`: Scene setup, robot creation, NPC/panel placement
- `animate()`: Main game loop (movement, interaction, rampage escalation)
- `processInteraction()` / `completeInteraction()`: E-key hold mechanics
- `startGame()` / `endGame()`: Game lifecycle

### Role System
Three roles: Commander (指揮官), Rescue (レスキュー), Scout (情報員) - stored in `players.role`

---

## Common Issues and Solutions (学んだ教訓)

### 1. マルチプレイヤー同期の順序問題

**問題**: リーダーがゲーム開始時にDBを更新する前に、他プレイヤーがデータを読み込もうとして失敗する

**解決策**:
- データを先に保存してから、ステータス更新で他プレイヤーに通知する
- 例: パネルコード生成 → DB保存 → 待機(300ms) → チームステータスを'playing'に更新

```javascript
// ❌ 悪い例
await db.from('teams').update({ status: 'playing' });
generatePanelCodes();
await db.from('game_states').update({ panel_codes: ... });

// ✅ 良い例
generatePanelCodes();
await db.from('game_states').update({ panel_codes: ... });
await new Promise(r => setTimeout(r, 300)); // DB反映待ち
await db.from('teams').update({ status: 'playing' });
```

### 2. 変数のnull参照問題

**問題**: モーダルを閉じた後に変数がnullになり、その後の処理で参照エラー

**解決策**: 変数を使う処理の前に、別の変数に保存しておく

```javascript
// ❌ 悪い例
closeCodeInputPanel(); // currentCodePanel = null になる
unlockPanel(currentCodePanel); // null が渡される

// ✅ 良い例
var panelToUnlock = currentCodePanel;
closeCodeInputPanel();
unlockPanel(panelToUnlock);
```

### 3. ポインターロックとモーダルの競合

**問題**: モーダル内のボタンをクリックしても、ポインターロックが再取得されてしまう

**解決策**:
- モーダル内のクリックで`e.stopPropagation()`を呼ぶ
- documentのクリックハンドラでモーダルが開いているかチェック

```javascript
document.getElementById('modal').addEventListener('click', function(e) {
  e.stopPropagation();
});

document.addEventListener('click', function(e) {
  var modal = document.getElementById('modal');
  if (modal && modal.style.display === 'flex') return;
  // ポインターロック取得処理
});
```

### 4. マルチプレイヤーでのヒント同期

**問題**: 各プレイヤーが独自にヒントをシャッフルして、異なるヒントが表示される

**解決策**:
- シード付き乱数を使用（teamIdやpanelCodesをシードにする）
- ヒントをキャッシュして毎回再生成しない
- チームヒント共有システムで取得したヒントをDB経由で共有

```javascript
function seededRandom(seed) {
  var x = Math.sin(seed) * 10000;
  return x - Math.floor(x);
}

// シード生成（全員同じ値になる）
var seed = 0;
for (var c = 0; c < teamId.length; c++) {
  seed += teamId.charCodeAt(c) * (c + 1);
}
```

### 5. NPC状態の同期不足

**問題**: 他プレイヤーが救助したNPCが、自分の画面では倒れたまま/座ったまま

**解決策**: NPC状態変更時に必要なプロパティをすべて更新する

```javascript
// 他プレイヤーの救助を受信した時
if (ns.is_rescued && !npc.rescued) {
  npc.rescued = true;
  npc.hintObtained = true;
  playRescueEffect(npc); // エフェクトも再生
  if (npc.npcType === 'standing') {
    npc.canWalk = true; // 歩行可能フラグも設定
  }
}
```

### 6. Supabaseリアルタイム購読の信頼性

**問題**: リアルタイム購読が遅延したり、通知が届かない場合がある

**解決策**: ポーリングをバックアップとして併用する

```javascript
// リアルタイム購読
subscription = db.channel('lobby-' + teamId)
  .on('postgres_changes', { ... }, callback)
  .subscribe();

// バックアップポーリング（2秒ごと）
setInterval(function() {
  refreshLobbyData();
}, 2000);
```

### 7. 初期化関数の呼び出し忘れ

**問題**: 関数を定義したが、呼び出しを忘れて機能しない

**解決策**:
- 新しい初期化関数を追加したら、startGame()やshowLobby()などの適切な場所で呼び出す
- 呼び出し箇所をコメントで明記

```javascript
// initShuffledHints() - startGame()内でpanelCodes設定後に呼び出す必要あり
```

### 8. Edit toolの「unexpectedly modified」エラー

**問題**: Edit toolでファイル編集時に「File has been unexpectedly modified」エラー

**解決策**: Node.jsスクリプトを使用して編集する

```bash
node << 'SCRIPT'
const fs = require('fs');
let content = fs.readFileSync('index.html', 'utf8');
content = content.replace(oldCode, newCode);
fs.writeFileSync('index.html', content, 'utf8');
SCRIPT
```

### 9. 非リーダーがゲーム開始できない問題

**問題**: リーダーがゲーム開始ボタンを押しても、他のチームメンバーが3D空間に入れない

**原因**:
- Supabaseリアルタイム購読が遅延または失敗している
- `startGame()`が複数回呼ばれて競合状態になる
- ポーリング間隔が長すぎてステータス変更を検知できない

**解決策**:

1. 重複呼び出し防止ガードを追加:
```javascript
var gameStarting = false;

async function startGame() {
  if (gameStarting || G.playing) {
    console.log('startGame blocked: already starting or playing');
    return;
  }
  gameStarting = true;

  try {
    // ... ゲーム開始処理
  } catch (e) {
    gameStarting = false; // エラー時はリセット
    throw e;
  }
}
```

2. ポーリング間隔を短くする（2秒→1秒）:
```javascript
lobbyRefreshInterval = setInterval(function() {
  refreshLobbyData();
}, 1000); // 2000から1000に短縮
```

3. ステータスチェックにログを追加してデバッグしやすくする:
```javascript
if (teamCheck.data) {
  console.log('Team status check:', teamCheck.data.status);
  if (teamCheck.data.status === 'playing') {
    console.log('Starting game via polling...');
    startGame();
  }
}
```

### 10. 非同期関数の重複実行問題

**問題**: 同じ非同期関数が複数回同時に呼ばれて予期しない動作をする

**解決策**: フラグ変数でガードする

```javascript
var isProcessing = false;

async function doSomething() {
  if (isProcessing) return;
  isProcessing = true;

  try {
    await someAsyncOperation();
  } finally {
    isProcessing = false;
  }
}
```

### 11. Supabase `.single()` による400エラー

**問題**: Supabaseの`.single()`メソッドを使ったクエリが400 Bad Requestを返す

**原因**:
- `.single()`はレコードが0件または2件以上の場合に400エラーを返す
- マルチプレイヤー同期で、レコードがまだ作成されていない/まだ更新されていない場合に発生

**解決策**: `.single()`を使わず、通常の配列として取得する

```javascript
// ❌ 悪い例 - レコードがない場合に400エラー
var result = await db.from('game_states')
  .select('panel_codes')
  .eq('team_id', teamId)
  .single();
if (result.data) { ... }

// ✅ 良い例 - 配列として取得
var result = await db.from('game_states')
  .select('panel_codes')
  .eq('team_id', teamId);
if (result.data && result.data.length > 0) {
  var record = result.data[0];
  // ...
}
```

**注意**: `insert().select().single()`は新規作成直後なので問題ない。問題になるのは既存レコードを検索する場合。

### 12. Supabase updateが失敗してもエラーにならない問題

**問題**: Supabaseの`update()`はレコードが存在しない場合、エラーを返さず空の結果を返す

**解決策**: update後に`.select()`を追加し、結果をチェックしてupsertパターンを使用

```javascript
// ❌ 悪い例 - updateが失敗してもわからない
await db.from('game_states').update({
  panel_codes: JSON.stringify(panelCodes)
}).eq('team_id', teamId);

// ✅ 良い例 - 結果をチェックしてレコードがなければinsert
var result = await db.from('game_states').update({
  panel_codes: JSON.stringify(panelCodes)
}).eq('team_id', teamId).select();

if (!result.data || result.data.length === 0) {
  // レコードが存在しない場合は作成
  await db.from('game_states').insert({
    team_id: teamId,
    panel_codes: JSON.stringify(panelCodes)
  });
}
```

### 13. パネルコード同期の最終解決策（シード生成）

**問題**: データベースを使ったパネルコード同期が400エラーになる

**最終解決策**: DB同期を使わず、team_idをシードにして全員が同じコードを生成

```javascript
// シード付き乱数で全員が同じコードを生成
generatePanelCodes(myData.teamId);
```

**メリット**: DB保存/取得不要、400エラーなし、必ず同じコード

---

## Testing Checklist

マルチプレイヤー機能をテストする際のチェックリスト:

1. [ ] パネルコードが全員同じか（コンソールで`Panel A/B/Main:`を確認）
2. [ ] ヒントが全員同じ内容か
3. [ ] チームヒントパネルに全員のヒントが表示されるか
4. [ ] 救助したNPCが他プレイヤーの画面でも立ち上がるか
5. [ ] 制御盤の正解が全員で一致するか
6. [ ] ゲーム開始時に全員が3D空間に入れるか

### 14. 制御盤解除状態が他プレイヤーに同期されない問題

**問題**: プレイヤーAが制御盤を解除しても、プレイヤーBの画面では未解除のまま

**原因**: `syncGameStateToServer()`がリーダーのみ実行可能だった

**解決策**: 全プレイヤーが制御盤解除を同期できる専用関数`syncPanelUnlock()`を追加

### 15. リアルタイム購読が機能しない場合のバックアップ

**問題**: Supabase Realtimeの購読が遅延または失敗し、ゲーム状態が同期されない

**解決策**: 非リーダーはポーリングでゲーム状態を定期的に取得

```javascript
// 非リーダー用ポーリング（1秒間隔）
if (!myData.isLeader) {
  setInterval(pollGameState, 1000);
}

async function pollGameState() {
  var gs = await db.from('game_states').select('*').eq('team_id', myData.teamId);
  if (gs.data && gs.data.length > 0) {
    syncFromGameState(gs.data[0]);
  }
}
```
