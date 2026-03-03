---
name: extracting-youtube-transcript
description: yt-dlpでYouTube字幕を取得しプレーンテキスト化。手動字幕→自動生成→Whisperの順に試行。
trigger: YouTube URLのトランスクリプト/字幕/文字起こし依頼時
---
# YouTube トランスクリプト取得

## 処理フロー（優先順）
1. `yt-dlp --list-subs "URL"` で字幕確認
2. `yt-dlp --write-sub --skip-download` （手動字幕）
3. `yt-dlp --write-auto-sub --skip-download` （自動生成字幕）
4. Whisper文字起こし（字幕なし時、ユーザー確認必須）
5. VTT→プレーンテキスト変換（重複除去）

## VTT→テキスト変換
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

## Whisper（最終手段）
```bash
yt-dlp -x --audio-format mp3 --output "audio_%(id)s.%(ext)s" "URL"
whisper audio_VIDEO_ID.mp3 --model base --output_format vtt
```
モデル: tiny（最速）/ base（推奨）/ small / medium / large（最高精度）

## タイトル取得
```bash
VIDEO_TITLE=$(yt-dlp --print "%(title)s" "URL" | tr '/' '-' | tr ':' '-')
```

## エラー対応
| 問題 | 対応 |
|------|------|
| 字幕なし | --write-sub と --write-auto-sub 両方試行→Whisper提案 |
| 非公開/年齢制限 | エラーメッセージをユーザーに通知 |
| 複数言語 | `--sub-langs en` で言語指定 |
