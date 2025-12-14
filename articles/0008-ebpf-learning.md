---
title: "eBPF を学び、未来をつくるための温故知新"
emoji: "🐳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["eBPF"]
published: false
---

# 前書き
こんにちわ、ハードウェア、組み込む ( 生産技術 ) からソフトウェア ( SRE ) に転向したエンジニアです。転向から 3 年程度が経ち、様々な技術に触れることで飽きる事なく学び続ける事ができ、緩やかな成長を実感しています。


# 概要
本記事は Uzabase Advent Calendar 2025 の 22 日目の記事です。

https://qiita.com/advent-calendar/2025/uzabase

なぜ eBPF を題材に取り上げたかというと、AIOps やセキュリティの強化を進める為には必要な技術だと感じているからです。eBPF について紹介し、eBPF を利用した tcpdump をつくることで深く理解し、AIOps の実現に向けたアプローチを提案します。

# eBPF について紹介

eBPF は Linux カーネルに導入された革新的な技術です。カーネルのソースコード変更やカーネルモジュールのロードを必要とせず、カーネルの機能を安全に効率的に拡張する技術です。


eBPF の中心的な機能はプログラムです。eBPF のカスタムプログラムをカーネルの内部に動的にロードし、カーネル内の様々なポイントにアタッチすることで、関数のように実行することができます。

https://ebpf.io/what-is-ebpf/


## eBPF の歴史
BPF ( BSD Packet Filter ) として提案されたものが eBPF の起源となります。BPF はユーザレベルでパケットをキャプチャする新たな機構として提案されました。
より汎用的な機能へと進化していきました。

| 年代 | 導入 | 内容 | 
| --- | --- | --- |
| 1993 年 | BPF ( BSD Packet Filter ) が，効率的なパケットフィルタリング手法として提案 | ネットワークパケットを通過させるか、拒絶されるかを決定するためのプログラム、フィルタを動かすための擬似的な機械を提案 |
| 1997 年 | BPF ( Berkeley Packet Filter ) が、Linux (カーネルバージョン : 2.1.75 ) に初めて導入 | tcpdump の中で、パケットのキャプチャに利用 |
| 2012 年 | seccomp-bpf が、Linux (カーネルバージョン : 3.5) に導入 | ユーザ空間のアプリケーションからのシステムコール呼び出し許可、禁止を決定 |


## eBPF の特徴

- eBPFはカーネルのソースコードの改変や、カーネルモジュールのロード不要で、カーネルの拡張が可能
- ネットワークのパフォーマンス向上、CPU、メモリリソースの削減
- アプリケーション ・・・ プロファイリング、モニタリング、トレーシングに利用可能
- カーネル・・・監視、セキュリティ、ネットワーキング、ロードバランシングに利用可能

## eBPF を利用したサービス
eBPF はネットワーク、セキュリティ、可観測性など様々な分野で利用可能な技術となります。そのために、eBPF を利用したサービスが数多く提供されるようになりました。

https://ebpf.io/applications/

2016 年頃

- Cilium : ネットワーク、可観測性、セキュリティに関するサービスを提供
    - Tetragon : Cilium プロジェクトの一つでセキュリティに特化したサービス
- Falco : ランタイム・セキュリティ監査とインシデント対応
- Istio : Ambient モードで eBPF を利用し、ネットワーク、可観測性、セキュリティに関するサービスを提供
- Datadog : ネットワーク監視、システム監視、ファイルシステム監視、プロセス実行監視
- NewRelic : 買収した Pixie でアプリケーションプロファイリング

# tcpdump をつくることで、eBPF を深く理解
eBPF をより深く理解するために tcpdump をつくります。

## Maps

https://docs.ebpf.io/linux/concepts/maps/

## Vrerifier

https://docs.ebpf.io/linux/concepts/verifier/

## XDP (eXpress Data Path)

https://prototype-kernel.readthedocs.io/en/latest/networking/XDP/introduction.html#what-is-xdp

## Aya

https://docs.rs/aya/latest/aya/index.html

# AIOps の実現に向けたアプローチ
:::message
個人的見解をを含めたアプローチとなります。
:::

いかに理想とする姿を具体的に想像するかが


1. (Monitorring) まず考えることは、データ収集基盤を整えることでしょう。
* 各サーバのデータを統一されたフォーマットのデータとして収集
* CPU、メモリ、ネットワークだけでなく、アプリケーション固有のメトリクス、アクセスログやアプリケーションログを収集

2. (Service Desk) データを元にした兆候管理、パフォーマンス最適化
* CPU、メモリ、ネットワーク、アプリケーション固有のメトリクス、アプリケーションのログは、障害検知に利用する。また、ボトルネックになっているリソースを調整することでパフォーマンスを改善する。
* リクエストのログは、異常なアクセスを検知するために利用する。

3. (Automation) 分析結果を元にした自動復旧、最適化
* 障害に対して、自動で復旧する。
* セキュリティインシデントの発生を検知し、アクセス制御をリアルタイムで強制する。
* ネットワークのメトリクスから、リアルタイムに通信経路を最適化

## tetragon の導入事例
Tetragon は、Kubernetes に対応した柔軟なセキュリティ観測およびランタイム強制ツールであり、eBPF を使用してポリシーとフィルタリングを直接適用することで、観測オーバーヘッドの削減、あらゆるプロセスの追跡、ポリシーのリアルタイム強制を可能にします。

https://tetragon.io/

# 結論
eBPF の良さは、ネットワーク、セキュリティ、可観測性などカバレッジの広さにあると思う。


# ご参考

論文
* [The BSD Packet Filter: A New Architecture for User-level Packet Capture](https://www.tcpdump.org/papers/bpf-usenix93.pdf)

記事とブログ
* [eBPF](https://docs.ebpf.io/)
* [XDP](https://prototype-kernel.readthedocs.io/en/latest/networking/XDP/introduction.html)
* [AYA](https://docs.rs/aya/latest/aya/index.html)
* [BPF for security—and chaos—in Kubernetes](https://lwn.net/Articles/790684/)
* 

参考書籍
* [入門 eBPF --- Linux カーネルの可視化と機能拡張](https://www.oreilly.co.jp/books/9784814400560/)