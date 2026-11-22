# hermes-daily-digest-video

豊中市 少路の「明日の天気」を、毎朝 **2本**（女の子版・テテ版）自動生成して公開するリポジトリ。
映像は ComfyUI（MiniMax H3）、音声は VOICEVOX / GPT-SoVITS で作る。

- **公開（Cloudflare Workers）**: <https://hermes-daily-digest-video.ryosuke-tanimoto.workers.dev/>
- 予備（GitHub Pages）: <https://tazrt.github.io/hermes-daily-digest-video/latest/>
- 公開フォルダ: `latest/`（`wrangler.jsonc` の `assets.directory`）。**サイトのルートが `latest/` になる**ので URL に `latest/` は付かない

## 構成

```
latest/                          # 配信される唯一のフォルダ
├── index.html                   # 自動生成（site/template.html から）
├── weather_girl.mp4  / .jpg     # 女の子版（VOICEVOX つむぎ）768x1344 / 14.38秒 / 2.95MB
└── weather_tete.mp4  / .jpg     # テテ版（GPT-SoVITS v5turbo）  768x1344 / 13.67秒 / 2.08MB
site/template.html               # index.html の元テンプレート（配信対象外）
tts/                             # TTS 品質比較用サンプル（v5turbo / v2ProPlus、配信対象外）
wrangler.jsonc                   # Cloudflare Workers（静的アセット）設定
```

- **`.jpg` はカードのポスター画像**。mp4 から ffmpeg で自動生成（t=2秒・768px幅・70〜90KB）
- 動画は固定ファイル名で上書きする（日付フォルダは作らない）

## 公開の流れ

```bash
# 動画（＋必要なら HTML）を latest/ へ入れて main に commit & push
python3 ~/.hermes/scripts/publish_video_repo.py <動画.mp4> [<ページ.html> ...]

# latest/ の中身はそのままに index.html だけ作り直す
python3 ~/.hermes/scripts/publish_video_repo.py --index-only

# commit までで止める（push しない）
python3 ~/.hermes/scripts/publish_video_repo.py --no-push <動画.mp4>
```

- `girl_final.mp4` → `weather_girl.mp4`、`tete_final.mp4` → `weather_tete.mp4` は自動リネーム（`--rename SRC=DEST` で任意指定）
- **1ファイル 25MiB 超は `FAILED over25MiB` で停止**（Cloudflare の上限。push しない）
- 成功時は最終行に `PUSHED <sha> <n> files | latest/ = ...` を出す
- main へ push すると **Cloudflare Workers が自動でデプロイされる**（実測 45〜60 秒で反映）
- GitHub Pages（main / ルート）も同じ `latest/index.html` を配信するため、2つの URL で中身は同一

## 実装上の決めごと

- **Git LFS は使わない。** Cloudflare は LFS ポインタを実体に解決しないので、LFS 管理のままだと
  132 バイトのテキストが配信されてブラウザでは真っ黒になる。mp4 は実体バイナリのままコミットする
  （現在 2〜3MB なので GitHub の 100MB 制限にも余裕がある）
- **`preload="none"` ＋ ポスター画像。** Cloudflare の静的アセットは Range リクエストを返さない
  （`Range: bytes=0-1023` を投げても 200 で全体が返り、`accept-ranges` も付かない）。
  そのままだと初回に 3本ぶん（約 8MB）を落とすので、押すまで動画を取りに行かないようにしている。
  初回の転送は HTML 約 10KB ＋ ポスター 2枚（約 164KB）で済む
- **`latest/index.html` は生成物。** `site/template.html` をプレースホルダ置換して作る
  （`{{CARDS}}` `{{COUNT}}` `{{RUNTIME}}` `{{UPDATED_JA}}` `{{UPDATED_ISO}}` `{{GRID_CLASS}}`）。
  手で直しても次回の公開で上書きされるので、デザイン変更はテンプレート側を編集する
- ページは本数に追従する（1本＝460px 1列 / 2本＝860px 2列 / 3本以上＝3列）
- フッターは置かない（更新時刻・名前・外部リンクは載せない）

## 生成パイプライン（Hermes ホスト側）

| 時刻 (JST) | スクリプト | 役割 |
| --- | --- | --- |
| 5:40 | `weather_daily_launch.sh` | WoL で生成PCを起こす → SSH トンネル（18188）→ `weather_daily_all.sh` をデタッチ起動 |
| 5:40〜6:40 | ComfyUI（MiniMax H3 / FastH3 8step + VSA10%） | 女の子版（VOICEVOX つむぎ）→ テテ版（GPT-SoVITS v5turbo）を**直列**レンダー。768×1344 |
| 7:10 | `weather_daily_publish_all.sh` | BGM ミックス → Koofr へ退避 → `publish_video_repo.py` で公開 → Telegram へ2本送信 |

- 尺は音声の実測長から決める（345フレーム＝14.375秒が上限）。映像だけ長いとリップシンクがズレる
- 描画は1インスタンスに固定する（ComfyUI や TTS が同居すると VRAM を奪い合って数倍遅くなる）

## 補足

- `.git` が約 76MB あるのは、過去に LFS でコミットした動画の履歴が残っているため（履歴改変はしていない）
- `tts/` は音声エンジン比較用のサンプルで、公開ページからは参照していない
