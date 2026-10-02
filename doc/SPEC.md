# 翻訳データの構造メモ

Alabaster Dawn のローカライズデータの構造と、ゲーム更新時の作業手順のメモ。
調査時点のゲームバージョン：2026-10-02 アップデート（changelog `02.10.2026`）。

## ディレクトリ構成（ゲーム本体）

```
{Steam}\steamapps\common\Alabaster Dawn\terra\data\locale\
├── original-texts.json      英語原文（全言語共通のマスター）
├── de_DE\ / fr_FR\ / zh_CN\ 他言語の翻訳
└── ja_JP\
    ├── ja_JP.json           言語設定
    ├── fonts\cc-japanese.ttf 公式同梱の日本語フォント（2026-10-02 版から）
    └── chunks\
        ├── database.json
        ├── lang.json
        ├── maps.json
        ├── maps.proto.json
        ├── maps.start.json
        └── remaining.json
```

このリポジトリの `terra\` は、ゲームの `terra\` に上書きコピーする前提の構成になっている。
フォントは公式がゲーム本体に同梱しているため、リポジトリには含めていない。

## original-texts.json（英語原文）

```json
{"entries": {"<セクション名>": [[ID, "英文", "コンテキスト", ""], ...], ...}}
```

- セクション名は `database.items`、`lang.gui`、`maps.start.center.center-06` など、データファイルのパスに対応する。
- 1件は `[ID(int), 英文, コンテキスト, 予備]` の4要素。
- 3番目のコンテキストはヒントとして使える。
  - `[item-id] [name]` や `[description]`：データ上の項目名
  - `[TALK Juno>surprised]`：会話の話者と表情。口調を合わせるときの手がかりになる。
- 2026-10-02 時点で143セクション、7,192件。
- **注意**：アップデートで、既存IDの英文が書き換わることがある（タグの削除、文の追加、仮テキストの置き換えなど）。
  ID が残っていても、訳が古くなっている場合がある。

## chunks\*.json（翻訳）

```json
{"entries": {"<セクション名>": [[ID, "訳文"], ...], ...}}
```

- 1件は `[ID, 訳文]` の2要素。セクション名と ID で原文と対応づける。
- 訳が無いエントリは英語にフォールバックする。
- セクションの格納先は、ゲーム側が「チャンク名がパスの先頭に一致するか」で決める。
  長いチャンク名から順に照合し、どれにも一致しなければ `remaining`。

| チャンク | 含まれるセクション |
|---|---|
| database.json | `database.*` |
| lang.json | `lang.*` |
| maps.start.json | `maps.start.*` |
| maps.proto.json | `maps.proto.*` |
| maps.json | それ以外の `maps.*` |
| remaining.json | `chars.*` / `cinematics.*` / `destructs.*` / `enemies.*` / `players.*` など |

- テキスト内のマークアップは原文と同じ形で残すこと。
  - `{c:3}…{c}`：色
  - `{.}` / `{..}`：ウェイト
  - `{v:…}`：変数参照
  - `{r:…}`：テキスト参照
  - `{i:…}`：アイコン
  - `{l:…}`、`{s:5}`
  - `|DESC|` / `|VAL|` / `|GEMS|` などのプレースホルダー
- 「…」はゲーム側のライブ翻訳処理で「...」に置換される（`fixUnicodeIssues`）。
  公式訳も「...」を使っているので、そちらに統一している。

## ja_JP.json（言語設定）

ゲーム側の `Language` クラス（`terra\dist\bundle.js`）が読み込む。

| キー | 意味 |
|---|---|
| name | 言語名 |
| noBitmapFont | true でビットマップフォントを使わない（日本語は必須） |
| systemFonts | 使うフォントの指定。`data/locale/ja_JP/fonts/<fontPath>` から読み込む |
| linebreakAnywhere / linebreakNotBefore / linebreakNotAfter / linebreakNotBetween | 改行と禁則処理の設定 |
| swapCommaDots | 数値の , と . を入れ替える |
| fixedMsgWidth | メッセージ幅を固定する |
| **debug** | **true だと製品版の言語選択に出てこない** |

### 日本語が選べなくなった件（2026-10-02 アップデート）

- この版から、公式の作業途中の日本語訳が `ja_JP` として同梱された。収録は約1,300件。
  Steam の更新で、このリポジトリのデータが公式版に上書きされた。
- 公式版の `ja_JP.json` は `"debug": true` になっている。
- `Locale.isActive()` は `!lang.debug || XG_GAME_DEBUG || XG_TEST_VERSION` を返す。
  そのため製品版では、`debug: true` の言語は言語一覧に出ない。fr_FR も同じ理由で非表示になっている。
- 対策として、このリポジトリの `ja_JP.json` は `"debug": false` を明示し、公式の `systemFonts`（cc-japanese.ttf）を取り込んでいる。
  **ゲームの更新で再び上書きされたら、リポジトリのデータを上書きし直す必要がある。**

## 翻訳方針

1. **公式訳が最優先。** ゲーム同梱の `ja_JP` に訳があるIDは、公式訳をそのまま使う。
   ただし空白だけのプレースホルダー（`" "`）は「公式訳なし」として扱う。
2. 公式訳が無いIDはこのリポジトリで翻訳する。用語・人名・口調は公式訳に合わせる。
   - 例：Weapons＝神器、Gem＝宝玉、Divine Arts＝ディヴァインアーツ、Life＝ライフ、Nyx＝ニクス、weave＝封じる、Chosen＝選士
   - 例：ジュノの一人称は「わたし」
3. 原文から消えたIDの訳は削除せずに残している。ゲーム動作に影響はない。他言語のデータにも同じように残っている。

## ゲーム更新時の手順

1. ゲーム側 `locale\ja_JP` の状態を確認する。
   公式訳が増えていないか、`ja_JP.json` が上書きされていないかを見る。
2. ゲーム側 `original-texts.json` と、このリポジトリの `chunks\*.json` を「セクション名＋ID」で突き合わせ、次の3つを洗い出す。
   - 未訳ID
   - 公式訳で置き換えるID
   - 英文が変わった既存ID
   英文の変化は、原文と訳文の `{…}` タグの組み合わせが一致しないもの、改行数が大きく違うもの、訳が「？？？」のままのものを目安に見つける。
3. 公式訳を上書きし、未訳と英文が変わった分を翻訳する。
4. 用語の統一、会話の前後関係と話者ごとの口調を確認する。
5. 確認ポイント
   - 全エントリで `{…}` タグが原文と一致すること
   - JSON として正しく読めること
   - `ja_JP.json` の `debug` が false であること

## 用語集（公式訳に無い語・統一した語）

公式訳にある語は常にそちらを優先する。以下は 2026-10-02 の更新時に決めた統一訳。


| English | 統一訳 | 備考 |
|---|---|---|
| Aspect (単独) | 属相 | 公式 |
| Aspect of X / X Element | Xの相 | 公式 "クライオの相" に準拠 (相神・属相神は使わない) |
| Weapon(s) | 神器 | 公式 |
| Gem | 宝玉 | 公式 (宝石 は使わない) |
| Gem Forge | 宝玉熔炉 | |
| Tiran Sol | ティラン・ソル | 原文に Terra（母神テラ）と Tiran の両方が出るため区別する。テラ・ソル は使わない |
| Tiran Nua | ティラン・ヌア | 同上 |
| Allmother Terra | 母神テラ | |
| Terryn(s) | テリン / テリン人 | |
| Age of the Firstborn | 始原の世代 | |
| X of the Firstborn (人物) | 始原の神選X | 旧訳準拠 (例: 始原の神選ヴァシア) |
| Firstborn (人々) | 始原の子ら | |
| Age of the Secondborn | 第二の世代 | |
| Secondborn (人々) | 第二の子ら | |
| Godless | 神なき者 | 無神者 は使わない |
| Godless Uprising | 神なき者の蜂起 | |
| House of Progress | 進歩の館 | 進歩院 は使わない |
| Order of the Allmother | 母神教団 | |
| Rebirth Council | 再生評議会 | |
| Marquin | マーキン | マーギン は使わない |
| Nerva | ネルヴァ | |
| Phos | フォス | |
| Vasia | ヴァシア | 公式 |
| Chosen | 選士 | 公式 |
| Valor / Wise | 勇士 / 賢士 | 公式 |
| Scala Moor | スカラの沼 | スカラ湿原 は使わない |
| Hall of Trials | 試練の間 | |
| Dojo | 道場 | |
| Somu's Dream | ソムの夢 | 公式 |
| The Dream (ローグライクモード) | 夢遊 | |
| Rest/Battle/Trial/Boss/Shop/Blessing Room | 休息の部屋 / 戦闘の部屋 / 試練の部屋 / ボスの部屋 / ショップの部屋 / 祝福の部屋 | 房間・〜ルーム は使わない |
| Blessing | 祝福 | |
| Perk | パーク | |
| Artificer | 神匠 | |
| Divine Charge / DC | ディヴァインチャージ / ＤＣ | 公式。神聖チャージ は使わない |
| Divine Arts | ディヴァインアーツ | 公式 |
| Daring Joust | デアリングジョスト | 宝玉名はカタカナ (公式 ブラッドアイ 準拠) |
| Brittle/Drain/Burn/Freeze/Chill | 脆弱/衰弱/燃焼/凍結/冷気 | |
| Deflect | 減衰 | |
| Essence | 精髄 | 公式 |
| Healing Bulb | 癒しのつぼみ | 公式 |
| Slightly/Mildly/Moderately/Greatly/Vastly increases | わずかに/少し/ある程度/大きく/非常に大きく アップする | 公式は Greatly=大きく〜アップする のみ。他は旧訳の段階表現に合わせる |
| Juno o'Lira / Valor o'Lira | ジュノ・オ＝リラ / 勇士オ＝リラ | 公式 |
| Penterson / Toko / Milana / Nima / Orlanda / Lyhamn | ペンタソン / トーコ / ミラナ / ニマ / オルランダ / リュハムン | 公式 |
| Remis Rock | レミスの大岩 | 公式の台詞に準拠 |
| Triumvirate | 三頭執政 | |
| Sol Gate Bridge / Sun's Gaze / Koro Valley | ソル門橋 / 陽の眼差し / コロの谷 | |
| ellipsis | ... | 公式準拠（「……」は使わない） |

### 口調

- ジュノ：一人称は「わたし」。くだけた若い口調（〜だよ／〜なの／〜かな）。父母は「父さん／母さん」
- キャベツ：落ち着いた口調（〜だ／〜だろう／〜かな／〜だね）。ジュノを「君」と呼ぶ
- フィリア：くだけた少女の口調
- エシュリン：男性（〜だ／〜だろ）
- 若い頃のダーモン：「お前」
- そのほかの話者も、公式訳の台詞サンプルに合わせる
