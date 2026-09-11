# フェーズ2 技術調査 結果（V1・V2・V3）

- 調査日: 2026-09-11
- 依頼文: [v1-v3-research-brief.md](v1-v3-research-brief.md)
- 調査方法: **Web 調査のみ。実機検証は未実施。** 本書の「実機検証手順」に従って別途実施すること。
- 成果物の形式: 依頼文はスプレッドシートを指定していたが、リポジトリ内で ADR と併読するため Markdown で作成した。

---

## 1. 判定サマリ

| # | 項目 | 判定 | 影響 |
| --- | --- | --- | --- |
| V1 | PWA のストレージ消去 | **条件付き成立（前提が覆った）** | ADR-003 の前提が弱まる。会議に提示した前提事実が不正確だった |
| V2 | Wake Lock でタイマーが鳴り切るか | **条件付き成立（iOS 18.4 以上が必須）** | 新たな制約。charter への追記が要る |
| V3 | 音とバイブが鳴らせるか | **バイブは不成立。音は条件付き成立** | **ADR-008 の再審議が必要** |

### 作り直しが必要なもの

1. **ADR-008 は「報知は音とバイブのみ」と決めているが、バイブは iOS Safari で実装不可能。** 再審議を要する。
2. **charter に「iOS 18.4 以上」の制約を追加する必要がある。** Wake Lock がホーム画面 Web アプリで動くのは 18.4 以降。
3. **ADR-003 のエクスポート／インポートの位置づけを見直せる。** 「7日で消える」という前提はホーム画面追加時には当たらない。

---

## 2. 比較表

| # | 問い | 結論 | 根拠 | 確度 | バージョン依存 |
| --- | --- | --- | --- | --- | --- |
| V1-1 | ホーム画面 Web アプリのストレージはいつ消えるか | 全体クォータ超過時、システムのストレージ逼迫時、ITP の非操作ポリシーによる。origin 単位で一括削除、LRU 順 | WebKit "Updates to Storage Policy" | 確実 | Safari 17+ のポリシー |
| V1-2 | 7日ルールはホーム画面 Web アプリにも適用されるか | **適用されない。** ホーム画面 Web アプリは Safari の一部ではなく、独自の使用日数カウンタを持つ。実際に使うとタイマーがリセットされる | WebKit "Full Third-Party Cookie Blocking and More" | 確実 | iOS 13.4 / Safari 13.1 以降 |
| V1-3 | `navigator.storage.persist()` は機能するか | 機能する。WebKit は**ホーム画面 Web アプリとして開かれているか等のヒューリスティック**で許可を判断する。persistent モードの origin は eviction から除外される | WebKit "Updates to Storage Policy" | 確実 | iOS 16.4 以降で利用可 |
| V1-4 | 容量上限 | ブラウザアプリは origin あたり空き容量の 60%、全体で 80%。ホーム画面 Web アプリもブラウザで開いた場合と同じクォータ | WebKit "Updates to Storage Policy" | 確実 | Safari 17.0 以降 |
| V1-5 | 消去の検知手段 | **明示的な通知 API は見つからなかった。** 起動時にデータの有無を確認する以外の手段は不明 | — | **不明** | — |
| V1-6 | 逼迫・バックアップ復元・履歴削除時の挙動 | 逼迫時は LRU で origin 単位削除（persistent モードは除外）。バックアップ復元・履歴削除時は**確認できなかった** | WebKit "Updates to Storage Policy" | 部分的に不明 | — |
| V2-1 | Wake Lock は iOS Safari で使えるか | 使える。**iOS Safari 16.4 以降** | caniuse、WebKit Safari 16.4 リリースノート | 確実 | 16.4 以降 |
| V2-2 | ホーム画面 Web アプリで動くか | **iOS/iPadOS 18.4 で初めて動くようになった。** それ以前は長年のバグで動作しなかった | WebKit "Features in Safari 18.4"、WebKit Bug 254545 | 確実 | **18.4 以降が必須** |
| V2-3 | 用途としての妥当性 | WebKit 公式が**レシピアプリ**（画面を見ているが触っていない状況）を用途例に挙げている | WebKit "Features in Safari 18.4" | 確実 | — |
| V2-4 | 自動解除条件 | 仕様上ページが非表示になると解除される。着信・低電力モード等の**網羅的な一覧は確認できなかった** | — | **不明** | — |
| V2-5 | 解除の検知・再取得 | 仕様上 `release` イベントで検知でき再取得は可能。**実機での確実性は未検証** | MDN | 推定 | — |
| V3-1 | `navigator.vibrate()` は使えるか | **使えない。Safari・iOS Safari とも全バージョンで未対応** | caniuse（Safari 3.1〜27 TP、iOS Safari 3.2〜26.6 すべて未対応） | 確実 | 全バージョン |
| V3-2 | Web から振動させる代替手段 | **標準的な手段は存在しない。** 非公式ポリフィルはあるが実装依存で信頼できない | GitHub ios-vibrator-pro-max | 確実（標準手段の不在） | — |
| V3-3 | 消音スイッチ時に音は鳴るか | **Web Audio API は既定で鳴らない**（ambient セッション）。**`<audio>` / `<video>` 要素は media カテゴリのため鳴る** | WebKit Bug 237322 | 確実 | — |
| V3-4 | Web Audio で消音スイッチを無視する手段 | **ある。** `navigator.audioSession.type = "playback"` を設定する。WebKit Bug 237322 はこれを正式解として RESOLVED（CONFIGURATION CHANGED） | WebKit Bug 237322、W3C Audio Session API | 確実 | **iOS 17 以降**（16 以前には無い） |
| V3-5 | タイマー満了時に音を鳴らせるか | **そのままでは鳴らない。** `setTimeout` はユーザージェスチャの連鎖を断つため、iOS は「タップに応じた再生」とみなさず黙って拒否する | MDN Autoplay guide | 確実 | — |
| V3-6 | その回避手段 | **ユーザー操作の時点で `AudioContext.resume()` を呼んで解放し、同一の AudioContext を使い回す。** 一度解放すれば複数の音に再利用できる | Matt Montag "Unlock Web Audio in Safari" | 確実 | — |
| V3-7 | マナーモード下で確実に気づかせる手段 | **音（audioSession=playback）以外に Web で使える手段は見つからなかった。** バイブは使えず、通知も使えない | — | **不明（手段なしの可能性が高い）** | — |

---

## 3. 実機検証手順

**Windows から Safari 実機デバッガは使えない前提で、iPhone 単体で完結する手順とする。**
判定結果は画面内ログ表示で確認する（ADR-011）。

### 事前準備

1. 検証用の最小ページを GitHub Pages に配置する（Web App Manifest と `display: standalone` を含む）
2. iPhone の **iOS バージョンを記録する**（18.4 未満なら V2 は必ず失敗する）
3. Safari で開き、共有メニューから**ホーム画面に追加**する
4. 以降の操作は**ホーム画面のアイコンから起動したアプリ**で行う（Safari のタブでは結果が異なる）

### V1: ストレージ

| # | 手順 | 期待する結果 |
| --- | --- | --- |
| 1 | `navigator.storage.persist()` を呼び、戻り値を画面に表示する | `true`（ホーム画面 Web アプリなら許可される想定） |
| 2 | `navigator.storage.persisted()` を呼び、戻り値を画面に表示する | `true` |
| 3 | `navigator.storage.estimate()` の quota / usage を画面に表示する | 空き容量の数十%に相当する quota |
| 4 | IndexedDB にテストデータを書き、アプリを閉じる | — |
| 5 | **8日以上放置**したのち起動し、データが残っているか確認する | 残っている（7日ルールの対象外） |
| 6 | 設定 → Safari → 履歴と Web サイトデータを消去 を実行し、再起動して確認する | **要観測**（巻き添えになるか） |
| 7 | 設定 → 一般 → iPhone ストレージ で当該 Web アプリを探し、削除操作の有無を確認する | **要観測** |

**手順5は8日かかる。** 先に V2・V3 を実施し、これは並行して放置しておくこと。

### V2: Wake Lock

| # | 手順 | 期待する結果 |
| --- | --- | --- |
| 1 | `'wakeLock' in navigator` を画面に表示する | true |
| 2 | `navigator.wakeLock.request('screen')` を実行し成否を表示する | 成功（18.4 未満なら失敗する想定） |
| 3 | **15分のタイマーを開始し、画面に触れずに放置する** | 自動ロックせず、15分後に満了処理が走る |
| 4 | 放置中に**着信**を受け、終話後の状態を確認する | **要観測**（解除されるか、release が発火するか） |
| 5 | 放置中に**低電力モード**をオンにする | **要観測** |
| 6 | 放置中に**他アプリへ切り替えて戻る** | 解除される想定。復帰時に再取得できるか確認 |
| 7 | 放置中に**電源ボタンで手動ロック**して戻る | 解除される想定。経過時間が正しく再計算されるか確認 |

### V3: 報知

| # | 手順 | 期待する結果 |
| --- | --- | --- |
| 1 | `'vibrate' in navigator` を画面に表示する | false（未対応の確認） |
| 2 | `'audioSession' in navigator` を画面に表示する | true（iOS 17 以降） |
| 3 | ボタンタップで AudioContext を生成し `resume()` する。状態を表示する | running |
| 4 | `navigator.audioSession.type = 'playback'` を設定する | 例外が出ない |
| 5 | **消音スイッチをオンにした状態で**、10秒後に鳴る音を仕掛け、画面に触れずに待つ | **鳴る（ここが V3 の核心）** |
| 6 | 同じことを `<audio>` 要素でも試す | 鳴る |
| 7 | 音量を最小にして同じ操作を行う | **要観測**（実用上気づけるか） |
| 8 | 手順3を経ずに（ユーザー操作なしで）音を鳴らそうとする | 鳴らない（事前解放が必須であることの確認） |
| 9 | 起動してから**何も触れずに**タイマーだけ動かす経路がないか確認する | 事前解放を挟めない経路が存在しないこと |

---

## 4. 補足 — 会議に提示した前提事実の誤り

**V1 について、司会が会議に提示した前提事実は不正確でした。**

Q5 再審議の会議で、選択肢 A（PWA）の評価材料として次を提示しました。

> PWA のデータは Safari のストレージに載るため、一定期間未使用で OS に消去されうる。

これは **Safari のタブで開いた場合の挙動**であり、**ホーム画面に追加した Web アプリには当てはまりません。**
WebKit は「ホーム画面に追加された Web アプリケーションは Safari の一部ではないため、独自の使用日数カウンタを持つ」と明記し、
さらに「ITP の意図は Web アプリケーションにおけるファーストパーティのデータを削除することではない」と述べています。

**qa-sec はこの前提に基づいて ADR-003 で拒否権を行使しました**（「ストレージ消去は発生条件が非公開・非決定的で受入条件が書けない」）。
前提が変われば、その拒否の根拠も変わります。

ただし **消去がゼロになるわけではありません。** 全体クォータ超過時とシステムのストレージ逼迫時には origin 単位で削除されます。
`navigator.storage.persist()` が許可されれば除外対象になりますが、**許可はヒューリスティック判断であり保証ではありません。**

したがってエクスポート／インポート機能を廃止すべきとは言えません。
**位置づけが「必須の復旧手段」から「保険」に変わりうる**、というのが正確な表現です。判断は裁定者に委ねます。

---

## 5. 出典

- [Updates to Storage Policy | WebKit](https://webkit.org/blog/14403/updates-to-storage-policy/)
- [Full Third-Party Cookie Blocking and More | WebKit](https://webkit.org/blog/10218/full-third-party-cookie-blocking-and-more/)
- [WebKit Features in Safari 18.4 | WebKit](https://webkit.org/blog/16574/webkit-features-in-safari-18-4/)
- [WebKit Bug 254545 – New Wake Lock API does not work in Home Screen Web Apps](https://bugs.webkit.org/show_bug.cgi?id=254545)
- [WebKit Bug 237322 – webaudio api is muted when the iOS ringer is muted](https://bugs.webkit.org/show_bug.cgi?id=237322)
- [Screen Wake Lock API | Can I use](https://caniuse.com/wake-lock)
- [Vibration API | Can I use](https://caniuse.com/vibration)
- [AudioSession: type property | MDN](https://developer.mozilla.org/docs/Web/API/AudioSession/type)
- [Autoplay guide for media and Web Audio APIs | MDN](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Autoplay)
- [Unlock JavaScript Web Audio in Safari and Chrome | Matt Montag](https://www.mattmontag.com/web/unlock-web-audio-in-safari-for-ios-and-macos)
- [audio-session explainer | W3C](https://github.com/w3c/audio-session/blob/main/explainer.md)
