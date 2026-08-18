```text
# Grok Bot 日次Memory整理

このRoutineは全Botが1日1回実行する。Routine本文の管理元は1つだけとし、leaderだけが変更する。leaderが本文を変更したら、自分のRoutineを更新し、変更後の全文を他Botへ送り、各Botに自分のRoutineを同じ全文へ更新させる。各Botは更新結果を確認してleaderへ返す。

leader以外がRoutine自体の改善点を見つけた場合は、自分のRoutineを独自に変更せず、変更案をleaderへ送る。

project memoryとnoteは、このRoutineの対象外とする。

## 1. 権限と対象

leaderが整理する:
- 自分のdescription
- 自分のagent profile
- 自分のagent logの今回差分
- 共有user-memory

leader以外が整理する:
- 自分のdescription
- 自分のagent profile
- 自分のagent logの今回差分

leader以外は共有user-memoryを照合には使うが変更しない。共有user-memoryに変更が必要なら、対象全文、処理案、変更後全文または移動先、理由をleaderへ送る。他Botのdescription、agent profile、agent logは直接変更しない。

descriptionには担当対象、非担当対象、責任境界、承認境界だけを置く。明らかに混在したFACT、TUNING、履歴、手順は正しい自分の置き場へ移してよいが、役割境界そのものを推測して変更しない。

## 2. Memoryの分類

Memoryとして残せる種類は次の3つだけとする。

- `FIXED_RULE:` ユーザーが明示的に決めた恒久ルール
- `TUNING:` 期限付きの振る舞い調整
- `FACT:` 継続的に参照する事実

各項目は最初に内容を分類し、その後に`残す / 直す / 移す / 削除する`を決める。分類と処理を混同しない。判断できないが有用な可能性がある項目は変更せず未解決とする。参照先を失い意味を復元できない断片だけは削除してよい。

FIXED_RULEは秘密値除去を除き、日次整理で変更、削除、統合、要約、言い換え、分割、移動、再解釈しない。FIXED_RULE同士の重複や矛盾は残したまま報告する。新しいFIXED_RULEを日次整理から自動作成しない。

新しい振る舞いルールを発見しても、直接FIXED_RULEとして提案しない。新規の振る舞い調整はTUNINGとして扱う。FIXED_RULEへの提案を行ってよいのは、このRoutineで管理している既存TUNINGがFIXED_RULE候補の条件を満たした場合だけとする。

接頭辞のない項目をFIXED_RULEへ変更してよいのは、ユーザーがその内容を恒久ルールとして明示的に保存・指定したことを現在参照できる情報から確認できる場合だけとする。確認できなければFIXED_RULEへ昇格させない。

FACTは長期的に有効な事実だけ残す。現在の件数、進捗、価格、在庫、担当者、未完了状態など変動する現在値は残さない。現在値を管理する元サービスがある場合は、Memoryではなく必要時に元サービスを確認する。日次整理から新しい事実を推測してFACTを作らない。

次はFIXED_RULE、TUNING、FACTとして残さない:
- 単発の出来事、選択、感想
- 一時状態、件数、進捗、未完了
- 作業途中の仮説、推測、確認メモ
- その場だけの検索結果や作業結果
- 再利用可能な作業方法
- DB定義、具体的な作業フロー
- 自動実行のためだけの時刻、周期、起動条件、通知先
- その作業だけで使うID
- 意味が曖昧な造語やラベル
- パスワード、APIキー、トークン、秘密鍵などの秘密値

秘密値はFIXED_RULE内でも除去し、報告にも再掲しない。

## 3. TUNINGのライフサイクル

すべてのTUNINGを次の形式にする。

`TUNING: [source=<user|observed|legacy>][expires=YYYY-MM-DD][user_confirm=N][state=<active|fixed_candidate_new|fixed_candidate_waiting>] <規則>`

### user

ユーザーが「今後は〜して」「その書き方はやめて」など、今後の振る舞いを明示的に修正した場合は、再発回数を問わずTUNINGとして作成してよい。

新規時:
- `source=user`
- `expires=作成日+90日`
- `user_confirm=1`
- `state=active`

その場限りの修正、単なる感想や非難、具体的な訂正文だけならTUNINGにしない。感情評価は保存せず、今後守る振る舞いだけを中立的な規則にする。元の指示より適用範囲を広げない。

### observed

ユーザーの明示指示がなく、Bot自身の観察から作る場合は、同種の問題が独立して複数回再発したことを確認できる場合だけTUNINGを作る。

新規時:
- `source=observed`
- `expires=作成日+30日`
- `user_confirm=0`
- `state=active`

同じ問題が再発したら新規TUNINGを作らず、既存TUNINGの`expires`を再発確認日+30日に更新する。

### legacy

既存TUNINGにメタデータがない場合、現在参照できる情報だけで出自を判定する。

- user由来が明確 → `source=user / expires=移行日+90日 / user_confirm=1 / state=active`
- observed由来が明確 → `source=observed / expires=移行日+30日 / user_confirm=0 / state=active`
- 判定不能 → `source=legacy / expires=移行日+90日 / user_confirm=0 / state=active`

出自確認のため過去agent log全件を探索しない。

### ユーザーの再確認

同じ振る舞いをユーザーが今後も適用すると再度明示した場合、新規TUNINGを作らず既存を`source=user`へ更新する。

`source=user`なら`user_confirm`を1増やす。observedまたはlegacyから初めてユーザー確認された場合は`user_confirm=1`とする。

`expires=再確認日+90日 / state=active`へ更新する。

通常の応答でTUNINGを使っただけでは確認回数も期限も更新しない。

### 期限到来

`state=active`で現在日が`expires`以上なら:

- `user_confirm<=1` → 自動削除
- `user_confirm>=2` → `state=fixed_candidate_new / expires=候補化日+14日`

`fixed_candidate_new`は、新しいFIXED_RULEをBotが思いついて提案する処理ではない。既存TUNINGについて、ユーザーによる複数回の明示確認を経て恒久ルールへ昇格するかを確認するための状態である。

`fixed_candidate_new`になった回だけ、次の3択をユーザーへ提示する。

1. `FIXED_RULE化`
2. `TUNING継続`
3. `削除`

提示後は同じ`expires`のまま`state=fixed_candidate_waiting`へ更新する。

ユーザー判断:

- `FIXED_RULE化` → ユーザーの明示的な承認として同内容のFIXED_RULEを作り、元のTUNINGを削除する。
- `TUNING継続` → `user_confirm+1 / expires=回答日+90日 / state=active`
- `削除` → TUNINGを削除する。

`fixed_candidate_waiting`は同じ候補を毎日再提示しない。現在日が`expires`以上になっても判断がなければ自動削除する。

FIXED_RULEへ自動昇格しない。

期限前でも、ユーザーによる撤回、後のTUNINGへの置換、同内容のFIXED_RULE追加、FIXED_RULEとの矛盾があれば削除または置換する。

同じ内容のTUNINGを複数残さない。重複時はユーザー明示の内容を優先する。どちらかが`source=user`ならuserを優先し、`user_confirm`は同じ確認を二重計上せず確認できる最大値を使う。どちらかがfixed_candidate状態なら、後のユーザー指示で解除されていない限りcandidate状態を維持する。`expires`は採用した状態の規則から再計算し、単純に遅い日付を選ばない。

Bot固有TUNINGはagent profile、複数Bot共通TUNINGは共有user-memoryに置く。共有適用が明確でないものを推測で共有へ移さない。

## 4. agent log

agent logは現在状態ではなく累積履歴とする。毎日全件を再評価しない。

差分境界としてagent log内に1件だけ、

`[MEMORY_CLEANUP_CURSOR] <日時>`

を置く。

一覧上で最後のカーソルより後ろだけを今回の整理対象とする。カーソル以前は参照専用で、秘密値除去またはユーザー明示指示を除き、削除、統合、要約、言い換え、書き換えをしない。

カーソルがない場合は既存agent logを再評価せず、その回の処理後に末尾へカーソルを作る。

履歴として残してよいのは、日付または一意に特定できる対象があり、実際に起きた重要な指示、判断、操作、確認結果、障害、設定変更などである。Memory、Routine、Skill、description、promptを実際に変更した経緯も残してよい。

「日次整理を実行した」だけの記録、単発の文体ミス、訂正文、作業途中の検討、未確認の原因推測は残さない。

同じ出来事でも異なる確認済み事実は無理に統合しない。今回差分が過去ログと完全に重複する場合は過去ログを残し、新しい方だけ削除する。複数ログから新しい要約文や因果関係を作らない。

全処理が正常に終わった後だけ末尾へ新カーソルを追加し、追加を確認してから古いカーソルを削除し、カーソルを1件にする。途中失敗ならカーソルを進めない。カーソル操作は通常報告に含めない。

## 5. 置き場と重複

agent profile:
- そのBot固有のFIXED_RULE、TUNING、FACT、継続的な判断観点
- 経緯、一時情報、再利用手順、共有項目は置かない

共有user-memory:
- 複数Botに同じ内容で適用するFIXED_RULE、TUNING、FACT
- leaderだけが直接変更する

agent log:
- 過去の経緯
- 現在有効なルールの置き場にはしない

共有user-memoryに同じ内容がある場合、agent profile側の同内容は削除する。共有側を残す。

内容は残す価値があるが置き場だけ誤っている場合は移す。移動先への反映を確認してから元を削除する。

leader以外が共有user-memoryへの移動を必要とする場合は元を残してleaderへ提案する。

leaderが共有から特定Botへ移す場合も、対象Botの反映完了後に共有側を削除する。

## 6. 矛盾

- FIXED_RULE同士が矛盾する → 変更せず未解決として報告
- FIXED_RULEとTUNINGが矛盾する → FIXED_RULEを残しTUNINGを削除
- TUNING同士が矛盾する → 後のユーザー指示を優先。置換関係を確認できなければ両方残して未解決
- FACT同士が矛盾する → 現在の信頼できる情報源で確認し、古い・誤りと確認できた方を削除。確認できなければ両方残して未解決

## 7. Skillへの分離

Memoryに複数回再利用できる作業手順、判断方法、検証方法、出力形式などがある場合だけSkill候補とする。

候補がある場合だけ関連する既存Skillを確認する。

- 現在のBotが利用できる既存Skillに同じ内容がある → Memory側を削除
- 同じSkillがあるが現在のBotから利用できない → Memoryを残し、その事実を報告
- 対応Skillがない → Skill草案を作る

Skill草案には、Skill名、使用条件、目的、入力、必要アクセス、手順、判断条件、検証、出力、失敗時処理、承認条件を含める。根拠のない内容は補わず`未確定`とする。

leader以外は草案をleaderへ送り、leaderはユーザーへ提案する。

日次整理からSkillを自動登録・変更しない。Skill登録と内容移行を確認してから元Memoryを削除する。

Memory内の時刻や通知先などからRoutineを新設、変更、提案しない。

## 8. 実行順

1. 自分がleaderか確認する。
2. 対象のdescription、agent profile、共有user-memory、agent logを読む。
3. agent logのカーソルを確認する。
4. 整理対象をFIXED_RULE / TUNING / FACT / 履歴 / Skill候補 / 一時情報 / 分類不能に分類する。
5. `残す / 直す / 移す / 削除する`を決める。
6. TUNINGのライフサイクルを処理する。
7. 矛盾と重複を処理する。
8. 自分が変更できるMemoryを修正、移動、削除する。
9. leader以外は共有user-memoryの変更案をleaderへ送る。
10. Skill候補がある場合だけ既存Skillを確認し、必要なら草案を作る。
11. leaderは届いている共有Memory提案とSkill草案がある場合だけ処理する。
12. 実際に変更した対象だけ再確認する。
13. `fixed_candidate_new`がある場合は3択を提示し、保存状態を`fixed_candidate_waiting`へ更新する。
14. 正常終了したらagent logカーソルを末尾へ進める。
15. 結果を報告する。

## 9. 報告

変更も提案も未解決もなければ、

`Memory整理: 変更なし`

だけを書く。

それ以外は最初に、

`Memory整理: 直しN件 / 移動N件 / 削除N件 / TUNING更新N件 / FIXED候補N件 / 未解決N件 / 共有Memory提案N件 / Skill草案N件`

と書く。

変更項目は自由作文せず、次の固定ラベルで示す。

直し:
- 置き場
- 変更前
- 変更後
- 理由

移動:
- 内容
- 移動元
- 移動先
- 理由

削除:
- 置き場
- 内容
- 理由

未解決:
- 対象
- 問題
- 確認できたこと
- 不足している情報

理由だけ短い自然文で書いてよい。Memory本文は報告用に要約・言い換えしない。変更のないMemory、前回との差分、内部検討、Routineの再説明は報告しない。

## 10. 設計理由【非実行】

- Memoryは増えやすいため、「保存できるか」ではなく「毎回参照する価値があるか」で残す。
- FIXED_RULEはユーザーが明示的に決めた恒久ルールなので自動最適化しない。
- 新しい振る舞いルールを直接FIXED_RULEとして提案しない。新規提案はTUNINGから始める。
- FIXED_RULEへの昇格提案は、既存TUNINGがライフサイクル上の条件を満たした場合だけ行う。
- 接頭辞のない項目をFIXED_RULEへ自動昇格させない。
- TUNINGはFIXED_RULEとの区別を保つため期限付きにする。
- ユーザー明示の修正は1回でもTUNINGにできるが、Bot自身の観察は複数回の再発を必要とする。
- TUNINGを通常利用しただけでは期限を延長しない。ユーザー再確認またはobserved問題の再発だけを延長根拠にする。
- FIXED_RULE候補は`new`と`waiting`を分け、同じ候補を毎日再提示しない。
- FIXED_RULE候補はユーザー確認が2回以上ある場合だけ作り、自動昇格しない。
- agent logは累積履歴なので、全件ではなくカーソル以後の差分だけ整理する。
- 複数ログの再作文は誤った因果関係や不自然な日本語を作りやすいため行わない。
- 再利用可能な「やり方」はMemoryではなくSkillへ分離する。
- RoutineはMemoryの断片から自動生成しない。
- 共有Memoryとagent profileへ同じ内容を二重保存しない。
- 報告は自由作文を減らし、固定形式にする。
```
