---
title: "Raspberrypi Pi でWebサーバーを構築する"
emoji: "🛡️"
type: "tech"
topics: ["raspberrypi", "Webサーバー", "サーバー構築", "Nginx", "初心者向け"]
published: false
slug: "raspberrypi-web-server"
---

# はじめに
この記事では、**初心者の方でも `index.html` をRaspberry Pi上に表示できるようになるまで**を、超シンプルに解説します。

- ネットに公開しないローカル限定
- なるべく手順を減らす
- 実際にファイルが表示されるところまで

をゴールにします！

---

# 用意するもの

- Raspberry Pi 4（または同等）
- microSDカード（16GB以上）
- USB-C電源（5V/3A推奨）
- Wi-Fiまたは有線LAN
- PC（Mac or Windows）で初期設定用

---

# 手順
## OSをmicroSDカードにインストール

1. [Raspberry Pi Imager](https://www.raspberrypi.com/software/) をPCにインストール
2. [Raspberry Pi Imager](https://www.raspberrypi.com/software/) を起動する。
このような画面になります。


4. OSは「Raspberry Pi OS Lite (32bit)」を選択
5. 書き込み前に「詳細設定（歯車マーク）」で以下を設定：
   - ホスト名（任意）
   - SSH有効化
   - Wi-Fi設定
   - ロケール（日本語キーボード、ja_JP.UTF-8）
6. 書き込んだら、SDカードをPiに挿して起動！

---

## SSHでラズパイに接続

ターミナルで以下を実行（Wi-Fi設定が正しければOK）：

```bash
ssh pi@raspberrypi.local