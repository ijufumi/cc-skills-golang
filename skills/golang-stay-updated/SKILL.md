---
name: golang-stay-updated
description: "Provides resources to stay updated with Golang news, communities and people to follow. Use when seeking Go learning resources, discovering new libraries, finding community channels, or keeping up with Go language changes and releases."
user-invocable: false
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.2.3"
  openclaw:
    emoji: "📰"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent WebFetch WebSearch
---

<!-- markdownlint-disable table-column-style -->

# Goの最新情報をキャッチアップする

Goエコシステムの最新動向を把握するための厳選ガイドです。

## 公式Goリソース

| リソース            | 説明                                          |
| ------------------- | -------------------------------------------- |
| **go.dev**          | チュートリアルとツールを備えた公式Goウェブサイト |
| **pkg.go.dev**      | Goパッケージの検索とドキュメント       |
| **tour.golang.org** | インタラクティブなGoチュートリアル                      |
| **play.golang.org** | コードをテストするためのGoプレイグラウンド               |
| **go.dev/blog**     | 公式Goブログ                             |

## ニュースレター

| ニュースレター | 説明 | 購読 |
| --- | --- | --- |
| **Golang Weekly** | 毎週キュレーションされたGoコンテンツ、ニュース、記事 | <https://golangweekly.com/> |
| **Awesome Go Newsletter** | 新しいGoライブラリとツールの最新情報 | <https://go.libhunt.com/> |

## Reddit & コミュニティ

| コミュニティ | 説明 | URL |
| --- | --- | --- |
| r/golang | 30万人以上のメンバーを持つメインのGo subreddit | <https://www.reddit.com/r/golang> |
| golang wiki | リソースとFAQを含む公式wiki | <https://go.dev/wiki/> |
| gophers.slack.com | 公式Go Slackコミュニティ | <https://invite.slack.golangbridge.org> |
| Go Forum | 公式Goディスカッションフォーラム | <https://forum.golangbridge.org> |
| Discuss Go | 公式Goチームのディスカッション | <https://groups.google.com/g/golang-nuts> |

## 著名なGo開発者

影響力のあるGo開発者とコントリビューターをフォローしましょう:

### コアGoチーム

| Name | GitHub | Twitter/X | LinkedIn | Bluesky |
| --- | --- | --- | --- | --- |
| **Rob Pike** | robpike |  |  |  |
| **Ken Thompson** | ken |  |  |  |
| **Russ Cox** | rsc | @\_rsc | <https://www.linkedin.com/in/swtch> | <https://bsky.app/profile/swtch.com> |
| **Brad Fitzpatrick** | bradfitz | @bradfitz | <https://www.linkedin.com/in/bradfitz/> | <https://bsky.app/profile/bradfitz.com> |
| **Andrew Gerrand** | adg |  |  |  |
| **Robert Griesemer** | griesemer |  |  |  |
| **Dmitry Vyukov** | dvyukov | @dvyukov |  |  |

### Goツール & インフラストラクチャ

| Name | GitHub | Twitter/X | LinkedIn | Bluesky |
| --- | --- | --- | --- | --- |
| **Sam Boyer** | sdboyer | @sdboyer |  |  |
| **Daniel Theophanes** | kardianos | @kardianos |  |  |
| **Matt Butcher** | technosophos |  |  |  |
| **Jaana Dogan** | rakyll | @rakyll | <https://www.linkedin.com/in/rakyll/> |  |

### 人気のあるGo著者 & 教育者

| Name | GitHub | Twitter/X | LinkedIn | Bluesky |
| --- | --- | --- | --- | --- |
| **Mat Ryer** | matryer | @matryer | <https://linkedin.com/in/matryer> |  |
| **Dave Cheney** | davecheney | @davecheney | <https://linkedin.com/in/davecheney> |  |
| **Katherine Cox-Buday** | kat-co |  | <https://linkedin.com/in/katherinecoxbuday> |  |
| **Johnny Boursiquot** | jboursiquot | @jboursiquot | <https://linkedin.com/in/jboursiquot> |  |
| **Michał Łowicki** | mlowicki | @mlowicki | <https://linkedin.com/in/michał-łowicki-a60402b> |  |

### ライブラリ & フレームワーク著者

| Name | GitHub | Twitter/X | LinkedIn | Bluesky |
| --- | --- | --- | --- | --- |
| **Steve Francia** | spf13 | @spf13 | <https://linkedin.com/in/spf13> |  |
| **Samuel Berthe** | samber | @samuelberthe | <https://linkedin.com/in/samuelberthe> | <https://bsky.app/profile/samber.bsky.social> |
| **Mitchell Hashimoto** | mitchellh | @mitchellh | <https://linkedin.com/in/mitchellh> | <https://bsky.app/profile/mitchellh.com> |
| **Matt Holt** | mholt | @mholt6 |  |  |
| **Tomás Senart** | tsenart | @tsenart | <https://www.linkedin.com/in/tsenart/> |  |
| **Björn Rabenstein** | beorn7 |  |  |  |

### カンファレンススピーカー & コミュニティリーダー

| Name | GitHub | Twitter/X | LinkedIn | Bluesky |
| --- | --- | --- | --- | --- |
| **Carlisia Campos** | carlisia | @carlisia | <https://linkedin.com/in/carlisia> |  |
| **Erik St. Martin** | erikstmartin | @erikstmartin |  |  |
| **Brian Ketelsen** | bketelsen |  |  | @brian.dev |

## 必読ブログ

| ブログ            | 著者       | URL                                  |
| --------------- | ------------ | ------------------------------------ |
| The Go Blog     | Go Team      | <https://go.dev/blog>                |
| Rob Pike's Blog | Rob Pike     | <https://commandcenter.blogspot.com> |
| Dave Cheney     | Dave Cheney  | <https://dave.cheney.net>            |
| Ardan Labs Blog | Bill Kennedy | <https://www.ardanlabs.com/blog>     |

## YouTubeチャンネル

| チャンネル | コンテンツ | URL |
| --- | --- | --- |
| Go | 公式Goチーム | <https://www.youtube.com/@golang> |
| Gopher Academy | トーク & チュートリアル | <https://www.youtube.com/@GopherAcademy> |
| GopherCon Europe | ヨーロッパのカンファレンストーク | <https://www.youtube.com/@GopherConEurope> |
| GopherCon UK | イギリスのカンファレンストーク | <https://www.youtube.com/@GopherConUK> |
| Golang Singapore | シンガポールのミートアップ & カンファレンストーク | <https://www.youtube.com/@golangSG> |
| Ardan Labs | Goトレーニング & ヒント | <https://www.youtube.com/@ArdanLabs> |
| Applied Go | Goチュートリアル | <https://youtube.com/appliedgocode> |
| Learn Go Programming | 初心者向けチュートリアル | <https://youtube.com/learn_goprogramming> |

## 最新情報をキャッチアップするためのヒント

1. **1〜2つのニュースレターを購読する** — 情報過多にならないようにしましょう
2. **X/Blueskyで定期的に投稿している10〜20人のキーパーソンをフォローする**
3. **Go.dev/blogを毎週チェックする** — 公式アナウンス用
4. **Go Slackに参加する** — リアルタイムのディスカッション用
5. **pkg.go.devをブックマークする** — 新しいライブラリの発見用
6. **年に一度GopherConに参加する**（バーチャルまたは対面）

---

_注: このガイドは定期的に更新されます。追加の提案はGitHub issuesからどうぞ。_
