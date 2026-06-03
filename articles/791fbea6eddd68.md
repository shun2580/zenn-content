---
title: "multi-agent-shogunをGemini CLI + Ollamaで拡張した話 — 異種AIを同一パイプラインで動かすまで"
emoji: "⚔️"
type: "tech"
topics: ["claudecode", "multiagent", "gemini", "ollama", "llm"]
published: false
---

## はじめに

[multi-agent-shogun](https://github.com/yohey-w/multi-agent-shogun) という、Claude Code を複数起動してマルチエージェント開発環境を構築するOSSプロジェクトがあります。おしおさんの[紹介記事](https://zenn.dev/shio_shoppaize/articles/5fee11d03a11a1)を読んで「これは面白い」と思い、自分の環境に合わせてカスタマイズしながら使い始めました。

この記事は、その過程で**詰まったこと・工夫したこと・気づいたこと**をまとめた体験記です。

:::message
本記事はmulti-agent-shogunをベースにしています。素晴らしいプロジェクトを公開してくださっているおしおさんに感謝します。
:::

---

## 最初の構成と課題

最初は**Claude Pro（$20/月）+ Ollama ローカルLLM**というミニマム構成で始めました。Claude の使用量を節約するため、実行系の足軽（Ashigaru）を Ollama で動かす方針です。

しかし実際に運用してみると、すぐに2つの壁にぶつかりました。

### 壁1：ローカルLLMが指示をループし続ける

家老（Karo）から足軽に「このファイルを修正して報告せよ」と指示を送っても、Ollama（qwen2.5:7b）が指示の意図を正しく解釈できず、同じ操作を繰り返したり、見当違いの処理を始めたりするケースが頻発しました。

そのたびに人間が手動でリセットする必要があり、「自律実行」にほど遠い状態でした。

### 壁2：Claude Pro の5時間レート制限

Claude Pro には連続使用時間の制限があります。マルチエージェントで複数の Claude インスタンスを動かすと、思ったより早く制限に達し、開発の勢いが途切れてしまいます。

---

## Claude Max 5x への移行と設計方針

これらの課題を解消するため、**Claude Max 5x（$100/月）** へアップグレードしました。

ただし「全員 Claude Sonnet にすれば解決」とはしませんでした。Gemini CLI（無料）と Ollama（ローカル GPU）を維持することで、以下のバランスを取る設計にしています。

- **Claude 枠の節約**：単純なタスクは Gemini や Ollama に任せる
- **Claude のレート制限保険**：仮に Claude が詰まっても Gemini/Ollama は動き続ける
- **ローカル GPU の活用**：RTX 4060 Ti 8GB でプライバシーが必要なコードを処理

---

## 7名ハイブリッド構成の全体像

現在の構成はこうなっています。

```mermaid
graph TD
    A(["👤 殿<br/>自然言語で指示・承認のみ"])
    B["⚔️ 将軍<br/>Claude Sonnet 4.6<br/>戦略判断・統括"]
    C["🏯 家老<br/>Claude Sonnet 4.6<br/>タスク分解・品質判定"]
    D["🎯 軍師<br/>Claude Sonnet 4.6<br/>品質チェック・ダッシュボード"]
    E["⚙️ 足軽1・2<br/>Claude Sonnet 4.6<br/>複雑な実装タスク"]
    F["⚡ 足軽5<br/>Claude Haiku 4.5<br/>高速・軽量タスク"]
    G["💎 足軽3・6・7<br/>Gemini 2.5 Flash<br/>調査・コード生成（無料）"]
    H["🖥️ 足軽4<br/>Ollama qwen3.5:9b<br/>ローカルGPU推論（無料）"]

    A --> B --> C
    C --> D
    C --> E
    C --> F
    C --> G
    C --> H
```

| エージェント | CLI | モデル | 用途 |
|---|---|---|---|
| 将軍・家老・軍師 | Claude Code | Sonnet 4.6 | 指揮・判断・品質管理 |
| 足軽1/2 | Claude Code | Sonnet 4.6 | 複雑な実装タスク |
| 足軽3/6/7 | Gemini CLI | Gemini 2.5 Flash | 調査・コード生成（無料枠） |
| 足軽4 | OpenCode | Ollama qwen3.5:9b | ローカルGPU推論 |
| 足軽5 | Claude Code | Haiku 4.5 | 軽量・高速タスク |

### コスト構成

| 項目 | コスト |
|---|---|
| Claude Max 5x（将軍・家老・軍師・足軽1/2/5） | $100/月（固定） |
| Gemini CLI（足軽3/6/7） | 無料 |
| Ollama（足軽4） | 無料（電気代のみ） |
| **合計** | **$100/月** |

---

## 詰まった：Gemini CLI が inbox を処理できない問題

ここが今回の記事で一番お伝えしたい部分です。

multi-agent-shogun では、エージェント間の通信に**ファイルベースの inbox システム**を使っています。家老が足軽に仕事を頼むとき、こんな流れで動きます。

```
家老 → inbox.yaml に書き込み
     → inbox_watcher.sh が変更を検知
     → "inbox3" という短いシグナルを tmux send-keys で足軽のペインに送信
     → 足軽が inbox.yaml を読んでタスクを実行
```

Claude Code はこの「inbox3」を受け取ると、「3件の未読メッセージがある」と理解してファイルを読みに行きます。

**ところが Gemini CLI と OpenCode は「inbox3」の意味がわからない。**

Gemini CLI に「inbox3」を送ると、こんな返答が来ます。

```
I'm not sure what "inbox3" refers to. Could you please clarify your request?
```

さらに Gemini CLI はプロジェクトの `queue/inbox/` ディレクトリへのアクセスを「Path not in workspace」として弾くことがありました。ファイルを読もうとしても読めない状態です。

### 解決策：inbox_watcher.sh でCLI種別を判定する

`scripts/inbox_watcher.sh` を修正し、エージェントのCLI種別に応じて送信内容を変えるようにしました。

```bash
# nudge を送る直前に CLI 種別を取得する
local effective_cli_for_nudge
effective_cli_for_nudge=$(get_effective_cli_type)
# get_effective_cli_type() は tmux ペインの @agent_cli 属性を優先して読み取り、
# 未設定の場合は inbox_watcher.sh 起動時の引数（CLI_TYPE）にフォールバックする。
# switch_cli.sh がペイン切替時に @agent_cli を更新するため、
# 設定ファイルの値ではなく実際に動いている CLI を動的に判定できる。

if [[ "$effective_cli_for_nudge" == "opencode" ]] || \
   [[ "$effective_cli_for_nudge" == "gemini" ]]; then
    # Gemini/OpenCode には明示的な指示文を送る
    nudge="queue/inbox/${AGENT_ID}.yaml と \
           queue/tasks/${AGENT_ID}.yaml を Read して \
           タスクを実行せよ。完了後 inbox_write.sh で軍師に報告すること。"
fi
```

加えて、Gemini/OpenCode は inbox の既読処理（`read: false → true`）も自律的にできないため、`inbox_watcher.sh` 側で自動的に既読にする処理も追加しました。

```bash
if [[ "$effective_cli_for_nudge" == "gemini" ]] || \
   [[ "$effective_cli_for_nudge" == "opencode" ]]; then
    python3 -c "
import re
path = '${INBOX}'
with open(path, 'r') as f:
    content = f.read()
# 行頭のインデントを含めてマッチさせることで、
# タスク内容に 'read: false' という文字列が含まれていても
# 誤置換しないようにしている
updated = re.sub(r'^(\s+read: )false', r'\1true', content, flags=re.MULTILINE)
with open(path, 'w') as f:
    f.write(updated)
" 2>/dev/null || true
fi
```

:::message
`re.sub(r'read: false', ...)` のような単純なパターンだと、タスクの説明文に `read: false` という文字列が含まれていた場合に誤置換するリスクがあります。YAMLの `read` キーは必ずインデントされた位置にあるため、`^(\s+read: )false` のように行頭のインデントを含めてマッチさせることで安全に置換できます。
:::

この修正により、Gemini CLI と OpenCode も同一パイプラインで動かせるようになりました。

---

## 自律稼働のために追加した仕組み

「指示を出したら完成まで自律稼働してほしい」という目標のために、いくつかの仕組みを追加しました。

### スマホへのプッシュ通知（ntfy）

[ntfy](https://ntfy.sh/)（無料のプッシュ通知サービス）を使い、cmd完了・エラー・要対応イベントをスマホにリアルタイム通知する設定を追加しました。

```bash
# 家老が cmd 完了時に通知を送る
bash scripts/ntfy.sh "✅ cmd_XXX 完了 — {概要}"
```

「指示を出して離席 → 完了通知が届いたら確認」というハンズフリーの開発フローが実現しました。

### 家老の自己 /clear 機構

Claude Code はコンテキスト使用量が増えるにつれ応答が遅くなり、最終的には `/clear` でリセットが必要になります。しかし家老が「そろそろリセットが必要」と判断しても、自分自身に `/clear` を送る仕組みがありませんでした。

`scripts/inbox_write.sh` に `clear_command` タイプのself-sendを許可する例外を追加し、家老が自律的にリセットできるようにしました。

```bash
# 家老がコンテキスト高騰を感じたときに実行
bash scripts/inbox_write.sh karo "" clear_command karo
```

### 足軽が5分無応答なら別エージェントへ自動再割当て

非Claudeエージェント（Gemini/OpenCode）はまれにタスクを受け取っても処理できないことがあります。5分以上応答がない場合は別の足軽へ再割り当てするルールを家老の指示書に追加しました。

---

## GPU VRAM が Ollama の壁を決める

RTX 4060 Ti 8GB を使っている場合、`qwen3.5:9b` は約6〜8GBのVRAMを消費します。つまり**実質1体しか同時推論できません**。

Gemini CLI はGPUを一切使わないため、VRAM消費ゼロで足軽3/6/7として3体並列で動かせます。この特性を活かして「重いタスクは Sonnet、軽いタスクは Gemini または Ollama」という使い分けが機能しています。

---

## やってみてわかったこと

**詰まりの原因は「CLIの性能」ではなく「プロトコルの非互換」だった**

最初は Gemini CLI が「賢くない」から inbox を処理できないのだと思っていました。実際には「inbox3 という文字列の意味を知らない」というプロトコルの問題であり、明示的な指示文を送れば動くとわかったときは設計の面白さを感じました。モデルの性能を疑う前に、まず通信プロトコルを疑うべきでした。

**「動いているように見えて動いていない」状態が一番危険**

Gemini が inbox を読めないまま待機し続けても、外から見ると静止しているだけで区別がつきません。家老も「足軽が作業中」と思い込んで待ち続けます。タイムアウトと自動再割当ての仕組みが不可欠だと身をもって学びました。

**モデルの選択は「タスクの複雑さ」で決める**

Sonnet は高品質ですが、YAML のフィールドを1行書き換えるような単純タスクでも同じコストがかかります。Haiku や Gemini に振り分けることで Claude 枠を複雑なタスクのために温存できます。

**異種CLIの統合は「プロトコルの違い」を吸収する層が必要**

今回の inbox_watcher.sh の改修が象徴しているように、異なるAI CLIを同一システムで動かすには「各CLIの癖を知ったうえで吸収する中間層」が不可欠です。

---

## おわりに

multi-agent-shogunは非常によく設計されたフレームワークで、カスタマイズの余地も大きく、学びながら改善できる構成になっています。

「Claude だけで全部やる」より「Claude + Gemini + Ollama のハイブリッド」の方が、コスト・耐障害性・経験値のすべてで優れていると感じています。

カスタマイズした内容は以下のリポジトリで公開しています。

https://github.com/shun2580/ZenkakuHiragana-multi-agent-shogun

興味を持っていただけた方はぜひ、おしおさんのオリジナルの記事もあわせてご覧ください。

https://zenn.dev/shio_shoppaize/articles/5fee11d03a11a1
