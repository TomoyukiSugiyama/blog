---
title: "eBPF を学び、未来をつくるための温故知新"
emoji: "🐳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["eBPF"]
published: false
---
# 概要
本記事は Uzabase Advent Calendar 2025 の 22 日目の記事です。

https://qiita.com/advent-calendar/2025/uzabase

なぜ eBPF を題材に取り上げたかというと、AIOps を進める為には必要な技術だと感じているからです。eBPF について紹介し、eBPF を利用した tcpdump をつくることで深く理解し、AIOps の実現に向けたアプローチを提案します。

# eBPF について紹介

## eBPF とは
eBPF は Linux カーネルに導入された革新的な技術です。カーネルのソースコード変更やカーネルモジュールのロードを必要とせず、カーネルの機能を安全に効率的に拡張する技術です。eBPF の中心的な機能はプログラムです。eBPF のカスタムプログラムをカーネルの内部に動的にロードし、カーネル内の様々なポイントにアタッチすることで、関数のように実行することができます。

https://ebpf.io/what-is-ebpf/


## eBPF の歴史

## eBPF の特徴

## eBPF を利用したサービス
eBPF はネットワーク、セキュリティ、可観測性など様々な分野で利用可能な技術となります。そのために、eBPF を利用したサービスが数多く提供されるようになりました。

https://ebpf.io/applications/


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
1. (Monitorring) まず考えることは、データ収集基盤を整えることでしょう。
* 各サーバのデータを統一されたフォーマットのデータとして収集
* CPU、メモリ、ネットワークだけでなく、アプリケーション固有のメトリクス、アクセスログやアプリケーションログを収集

2. (Serrvice Desk) データを元にした兆候管理、パフォーマンス最適化
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

# 参考

* [eBPF](https://docs.ebpf.io/)
* [XDP](https://prototype-kernel.readthedocs.io/en/latest/networking/XDP/introduction.html)
* [AYA](https://docs.rs/aya/latest/aya/index.html)

書籍
「入門 eBPF」