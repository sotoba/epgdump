# epgdump

MPEG-2 TS（ISDB 等）に含まれる SI（主に SDT / EIT / NIT）を読み、番組表（EPG）またはチャンネルメタデータを **XML**（xmltv.dtd 風）、**CSV**、**JSON** で出力するコマンドラインツールです。

バージョンはビルド時に [CMakeLists.txt](CMakeLists.txt) の `serial` と一致し、`epgdump` を引数なしで実行したときの `VERSION` 表示でも確認できます。

## 由来

- **起源**: BonTest 由来の epgdump をベースにしたコードが含まれます。著作権・ライセンスは後述のとおりです。
- **Linux / xmltv 向け改造（xmltv-epg）**: Solaris 版 recfriio に含まれる epgdump の Linux 改造を元に、xmltv 用 XML を出力する流れでした。タイトル内の特定パターンをサブタイトルへ分離する処理は [eit.c](eit.c)（`subtitle_cnv_str` 等）にあります。
- **fork 元**: [Piro77/epgdump](https://github.com/Piro77/epgdump)（地上デジ・BS/CS 向けの改良版。`/BS` などのチャンネル種別引数の廃止、CSV/JSON、イベント `check` / `wait` など）。
- **本リポジトリ**: [sotoba/epgdump](https://github.com/sotoba/epgdump)。上記を引き継ぎ、チャンネル専用モード・地上波の送信形態（フルセグ/1セグ）出力・SDT の扱い改善などを追加しています。

以前の説明文の全文は git 履歴内の `readme.txt` から参照できます。

## ライセンス

BonTest 由来部分を含むため、従来どおり **GPL に従います**（ソース改変・再配布時は GPL の要件に従ってください）。オリジナル README にあった BonTest / FAAD2 に関する記述の趣旨はそのままです。詳細は歴史的な `readme.txt` や各上流のライセンス表記を参照してください。

## ビルド

- **CMake**（2.8 以上想定）
- **iconv**（`FindIconv` で検出）

例:

```sh
cmake -B build -S .
cmake --build build
```

生成された実行ファイルは `build/epgdump`（環境によってパスは異なります）。`cmake --install` で `bin` へインストール可能です。

## 使い方

入力の `<tsFile>` および出力の `<outfile>` に `-` を指定すると、それぞれ標準入力・標準出力になります。

### 番組表（EPG）の出力

| 形式 | コマンド |
|------|----------|
| XML（既定） | `epgdump <tsFile> <outfile>` |
| CSV | `epgdump csv <tsFile> <outfile>` |
| JSON | `epgdump json <tsFile> <outfile>` |

第一引数が `csv` / `json` 以外の 2 引数形式は、XML 出力として扱われます。

### チャンネル一覧のみ（EIT を待たない）

NIT（PID 0x10）と SDT（0x11）だけを最大約 **15 秒**（ソース内 `CHANNEL_SCAN_SEC`）読み、**SDT に現れる全サービス**の名前・ID 等を出力します。EIT スケジュールの有無は問いません（通常の XML/JSON/CSV では EIT スケジュールがあるサービスが中心となります）。

| 形式 | コマンド |
|------|----------|
| XML（チャンネル要素のみ） | `epgdump channels <tsFile> <outfile>` |
| JSON | `epgdump channels json <tsFile> <outfile>` |

### チャンネル一覧（CSV・EIT ありサービスのみ）

`csvc` は **EIT スケジュールがあるサービス**に限定したチャンネル一覧を CSV 風に出します（従来どおり）。

```text
epgdump csvc <tsFile> <outfile>
```

### デバイスからのイベント確認・待機

チューナデバイスなどから TS を直接読み、特定イベントの有無確認や開始待ちを行います。

```text
epgdump check <device> <sid> <eventid> <eventtime>
epgdump wait <device> <sid> <eventid> <maxwaitsec>
```

- `check`: 指定サービス ID・イベント ID が「次のイベント」に含まれるか。終了コード 0/1。取得不能など含め最長約 10 秒程度。
- `wait`: イベント開始を待つ。成功 0、失敗 1。

`check` の第 5 引数（イベント開始時刻）は [util.c](util.c) の `str2timet` が解釈する形式に限ります（19 文字以上）:

- `YYYY/MM/DD HH:MM:SS`
- `YYYY-MM-DDTHH:MM:SS`

### ヘルプ

引数を足りない状態で実行すると Usage が表示されます（`csvc` は実装されていますが、この Usage 一覧には出ません）。

## 出力形式の注意

- XML は xmltv 風ですが **独自拡張**があり、他製品の epgdump と **そのまま置き換えには使えません**。
- チャンネル ID は `id="<接頭辞>_<service_id>"` 形式です（`getBSCSGR` + サービス ID）。接頭辞は `original_network_id` および TSID から決まり、地上波は `GR` + リモコンキー ID（例: `GR2_23122`）、衛星は `B`/`C` ベースの略号（例: `BS_237`）などになります。詳細は [xmldata.c](xmldata.c) の `getBSCSGR` / `getTSID2BSCS` を参照してください。

## この fork の追加・変更（概要）

- **`channels` / `channels json`**: NIT+SDT のみで短時間スキャンし、番組情報なしでサービス一覧を出力。
- **地上波のフルセグ / 1セグ**: NIT の TS 情報記述子（ARIB TR-B14 の `transmission_type_info`）をサービス ID に対応付け、XML/JSON のチャンネル情報に `transmission_type` および `full_segment`（値 0x01 / 0x03 のとき）を付与。
- **JSON の `transmission_type`（実装メモ）**: 番組表 `epgdump json` では `name` が既に `,` で終わるため、`,"transmission_type"` をそのまま挟むと **二重カンマ**になる。`channels json` では直前のキーが `,` 無しで終わる場合がある。**`fprint_terrestrial_transmission_json(..., leading_comma)`** で両者を分ける。また `transmission_type` ブロックの末尾に `,` が無いと **`"programs"` と連結**して JSON が壊れるため、`transmission_type_info != 0` のときだけ `programs` 直前に `,` を補う。
- **SDT の複数セクション**: 同一サービスが再通知された場合に名前・ONID・TSID を更新し、セクション分割された SDT でも取りこぼしにくくしています。
- **EIT とサービス一覧の整合（SDT 未到達チャンネル）**: 従来実装では、SDT でまだ登録されていない `service_id` の EIT を受け取ると打ち切り（`EIT_SDTNOTFOUND`）になっていました。本 fork では、その場合に **サービス用のエントリを新規作成してリストへ追加**し、続けて EIT を解釈します。地方局などで **SDT に載らない／後から載るチャンネルの EIT だけが先に流れる**ようなケースでも、番組表を取りこぼしにくくするための変更です。あわせて `check` 用の時刻差表示・JSON の開始/終了時刻・音声コンポーネント記述子の取り扱い（空でないときだけ `strdup`）など、`time_t` や文字列まわりの細かな修正を含みます。

そのほか、全出力形式への `version_number` / `original_network_id` の付与など、番組・チャンネルメタの出力拡張が入っています（該当コミットは `git log` を参照）。

Piro77 版から引き続き、読み込み時の seek 回避、BS/CS のテーブル不要化、BIT に基づく EIT 送出周期の考慮、複数ジャンル対応などの改良が含まれます。

## 謝辞

オリジナル・fork 元の作者、Solaris 版開発者、拡張ツール作者、◆N/E9PqspSk 氏、ARIB 資料、本リポジトリの fork 元 [Piro77/epgdump](https://github.com/Piro77/epgdump) に感謝します。
