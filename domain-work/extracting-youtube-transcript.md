---
name: extracting-youtube-transcript
description: YouTube動画のトランスクリプト（字幕/キャプション）をダウンロードし、クリーンなテキストファイルとして保存する。yt-dlpを使用し、手動字幕・自動生成字幕・Whisper文字起こしの順に試行する。
trigger: ユーザーがYouTube URLを提示し、トランスクリプト/字幕/キャプションの取得を依頼した時
---

# YouTube トランスクリプト取得

YouTube動画の字幕（サブタイトル/キャプション）を yt-dlp で取得し、テキストファイルとして保存する。

## 発動条件

- YouTube URLを提示してトランスクリプトを求めた時
- 「YouTubeの字幕をダウンロードして」と言われた時
- 「動画のキャプションを取得して」と言われた時
- 「YouTube動画を文字起こしして」と言われた時

## 処理フロー（優先順）

1. **yt-dlp のインストール確認**
2. **利用可能な字幕を一覧表示**
3. **手動字幕を試行**（`--write-sub`）- 最高品質
4. **自動生成字幕にフォールバック**（`--write-auto-sub`）
5. **最終手段: Whisper 文字起こし**（字幕が無い場合、ユーザー確認後）
6. **VTTからプレーンテキストに変換**（重複除去）
7. **保存確認とファイル場所の表示**

> ツールのインストール手順は元リポジトリを参照:
> https://github.com/yt-dlp/yt-dlp#installation

## yt-dlp インストール確認

```bash
which yt-dlp || command -v yt-dlp
```

## 字幕の確認（必ず最初に実行）

```bash
yt-dlp --list-subs "YOUTUBE_URL"
```

手動字幕（高品質）と自動生成字幕の有無、利用可能な言語を確認する。

## ダウンロード方法

### 方法1: 手動字幕（推奨）

```bash
yt-dlp --write-sub --skip-download --output "OUTPUT_NAME" "YOUTUBE_URL"
```

### 方法2: 自動生成字幕（フォールバック）

```bash
yt-dlp --write-auto-sub --skip-download --output "OUTPUT_NAME" "YOUTUBE_URL"
```

両方とも `.vtt` ファイル（WebVTT字幕形式）を生成する。

### 方法3: Whisper 文字起こし（最終手段）

字幕が一切無い場合のみ使用。**必ずユーザー確認を取ること。**

```bash
# ファイルサイズ確認
yt-dlp --print "%(filesize_approx)s" -f "bestaudio" "YOUTUBE_URL"
yt-dlp --print "%(duration)s %(title)s" "YOUTUBE_URL"
```

ユーザーに「字幕なし。音声をダウンロードしてWhisperで文字起こしするか？」と確認。

```bash
# 音声ダウンロード
yt-dlp -x --audio-format mp3 --output "audio_%(id)s.%(ext)s" "YOUTUBE_URL"

# Whisper で文字起こし（base モデル推奨）
whisper audio_VIDEO_ID.mp3 --model base --output_format vtt
```

Whisper モデル: `tiny`（最速）/ `base`（推奨）/ `small` / `medium` / `large`（最高精度）

## 動画タイトルの取得

```bash
yt-dlp --print "%(title)s" "YOUTUBE_URL"
```

ファイル名用にクリーニング:
```bash
VIDEO_TITLE=$(yt-dlp --print "%(title)s" "URL" | tr '/' '-' | tr ':' '-')
```

## VTT → プレーンテキスト変換（重複除去）

自動生成VTTはタイムスタンプ重複で同じ行が複数回出現する。必ず重複除去すること。

```bash
python3 -c "
import sys, re
seen = set()
with open('transcript.en.vtt', 'r') as f:
    for line in f:
        line = line.strip()
        if line and not line.startswith('WEBVTT') and not line.startswith('Kind:') and not line.startswith('Language:') and '-->' not in line:
            clean = re.sub('<[^>]*>', '', line)
            clean = clean.replace('&amp;', '&').replace('&gt;', '>').replace('&lt;', '<')
            if clean and clean not in seen:
                print(clean)
                seen.add(clean)
" > transcript.txt
```

## 完全ワークフロー

```bash
VIDEO_URL="https://www.youtube.com/watch?v=VIDEO_ID"
VIDEO_TITLE=$(yt-dlp --print "%(title)s" "$VIDEO_URL" | tr '/' '_' | tr ':' '-' | tr '?' '' | tr '"' '')
OUTPUT_NAME="transcript_temp"

# 1. yt-dlp 確認
if ! command -v yt-dlp &> /dev/null; then
    echo "yt-dlp が見つかりません。インストールしてください。"
    exit 1
fi

# 2. 字幕一覧確認
yt-dlp --list-subs "$VIDEO_URL"

# 3. 手動字幕を試行
if yt-dlp --write-sub --skip-download --output "$OUTPUT_NAME" "$VIDEO_URL" 2>/dev/null; then
    echo "手動字幕を取得しました"
else
    # 4. 自動生成字幕にフォールバック
    if yt-dlp --write-auto-sub --skip-download --output "$OUTPUT_NAME" "$VIDEO_URL" 2>/dev/null; then
        echo "自動生成字幕を取得しました"
    else
        echo "字幕が利用できません。Whisper文字起こしを検討してください。"
        exit 1
    fi
fi

# 5. プレーンテキスト変換（重複除去）
VTT_FILE=$(ls ${OUTPUT_NAME}*.vtt 2>/dev/null | head -n 1)
if [ -f "$VTT_FILE" ]; then
    python3 -c "
import sys, re
seen = set()
with open('$VTT_FILE', 'r') as f:
    for line in f:
        line = line.strip()
        if line and not line.startswith('WEBVTT') and not line.startswith('Kind:') and not line.startswith('Language:') and '-->' not in line:
            clean = re.sub('<[^>]*>', '', line)
            clean = clean.replace('&amp;', '&').replace('&gt;', '>').replace('&lt;', '<')
            if clean and clean not in seen:
                print(clean)
                seen.add(clean)
" > "${VIDEO_TITLE}.txt"
    rm "$VTT_FILE"
    echo "保存先: ${VIDEO_TITLE}.txt"
fi
```

## 出力形式

- **VTT形式**（`.vtt`）: タイムスタンプ付き、動画プレーヤー用
- **プレーンテキスト**（`.txt`）: テキストのみ、読解・分析用

## エラー対応

| 問題 | 対応 |
|------|------|
| yt-dlp 未インストール | インストール案内を表示 |
| 字幕なし | `--write-sub` と `--write-auto-sub` の両方試行。両方失敗でWhisper提案 |
| 非公開/年齢制限動画 | yt-dlp のエラーメッセージをユーザーに通知 |
| Whisper インストール失敗 | `pip3 install openai-whisper` を案内 |
| 複数言語の字幕 | `--sub-langs en` で言語指定。`--list-subs` で確認 |

## ベストプラクティス

- ダウンロード前に必ず `--list-subs` で確認する
- 各ステップの成功を検証してから次へ進む
- 大容量ダウンロード（音声、Whisperモデル）はユーザー確認を取る
- 一時ファイルは処理後に削除する
- 各段階で進捗を明確にフィードバックする