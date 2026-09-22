# fizz-brain-orchestrator

**Fizz** (AITuber system in [Almide](https://github.com/almide/almide)) — §3 brain の合成 CLI。

責務は一行: **comment → sentence-fragment ストリーム**。各部品を束ねるだけの薄い層。

## パイプライン (1 コメントあたり)

```
comment → classify(参考ログ) → few-shot 検索 → prompt 構築
        → LLM 呼び出し (fizz-llm-client) → 文 + タグ切り出し
        → thinking 注釈 → SentenceFragment を NDJSON で stdout へ
```

会話履歴は窓つきで保持 (前の話題を引きずらせない方針)。
出力は `SpeakItem` ではなく `SentenceFragment` (テキスト世界) — 音声化は後続の
TTS 部品の責務 (`SpeakItem.audio` は in-process 専用でワイヤを越えない)。

## 依存している Fizz 部品

[fizz-protocol](https://github.com/aiviecast/fizz-protocol) /
[fizz-persona](https://github.com/aiviecast/fizz-persona) /
[fizz-llm-client](https://github.com/aiviecast/fizz-llm-client) /
[fizz-comment-classifier](https://github.com/aiviecast/fizz-comment-classifier) /
[fizz-few-shot-retriever](https://github.com/aiviecast/fizz-few-shot-retriever) /
[fizz-stream-sentence-parser](https://github.com/aiviecast/fizz-stream-sentence-parser) /
[fizz-conversation-history](https://github.com/aiviecast/fizz-conversation-history) /
[fizz-thinking-cue-detector](https://github.com/aiviecast/fizz-thinking-cue-detector) /
[fizz-prompt-builder](https://github.com/aiviecast/fizz-prompt-builder)

> LLM 層は本来 [almide-ai/almai](https://github.com/almide-ai/almai) を使う計画だったが、
> almai は 2026-05 以降未メンテで現行 almide (v0.24/v0.25) ではコンパイルできない
> (streaming callback closure の codegen 不整合 + Value Repr 欠落)。そこで almai と
> 同形 interface の [fizz-llm-client](https://github.com/aiviecast/fizz-llm-client) を使う。
> almai が現行 almide で通るようになれば drop-in 置換できる。

## Usage

```bash
almide build src/main.almd -o fizz-brain-orchestrator

fizz-youtube-chat-source | fizz-comment-dedupe \
  | FIZZ_PERSONA_DIR=personae/lime FIZZ_MODEL=anthropic/claude-haiku-4-5 \
    ./fizz-brain-orchestrator \
  | <TTS 部品>   # SentenceFragment → 音声
```

persona ディレクトリには `system.md` (必須) と `examples.tsv` (任意) を置く。

| env | 意味 | default |
|---|---|---|
| `FIZZ_PERSONA_DIR` | persona ディレクトリ | (必須) |
| `FIZZ_MODEL` | `<provider>/<model>` | `anthropic/claude-haiku-4-5` |
| `ANTHROPIC_API_KEY` 等 | provider の API キー | (provider に応じて) |

## 制約 (v1)

- LLM 呼び出しは **非ストリーミング** (almide v0.25 が streaming callback codegen
  バグのため)。応答全文を受けてから文に切り出す。
- `first-time-greeter` は recall/memory 層が配線されてから組み込む (本 v1 は未接続)。

## Tests

```bash
almide test          # 純ロジック (型変換 / thinking 注釈) + 全依存の in-file テスト
almide check src/main.almd
```
