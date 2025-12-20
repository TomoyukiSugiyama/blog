---
title: "eBPF を学び、未来をつくるための温故知新"
emoji: "🐳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["eBPF", "Kubernetes", "AI"]
published: false
---

# 前書き
こんにちわ、ハードウェア、組み込み ( 生産技術 ) からソフトウェア ( SRE ) の業界に転向したエンジニアです。転向から 3 年程度が経ち、様々な技術に触れることで飽きる事なく学び続ける事ができ、緩やかな成長を実感しています。昨今、AI 技術が著しく成長していることを AI コーディングなどから身にしみて感じておりますが、SRE としてはどのように AI をインフラに組み込み、基盤技術と AI による相乗効果を発揮させるのか、AI と既存技術の親和性はあるのか、といった点について考えるようになりました。そこで、eBPF という技術に焦点を当て話してみたいと思います。

# 概要
本記事は Uzabase Advent Calendar 2025 の 22 日目の記事です。

https://qiita.com/advent-calendar/2025/uzabase

さて、今回は歴史ある技術である eBPF の歴史変遷から現代の利用事例を紹介し、未来の応用に触れていきたいと思います。なぜ eBPF を題材に取り上げたかというと、AIOps やセキュリティの強化を進める為には必要な技術だと感じているからです。この記事では eBPF について紹介し、eBPF を利用した tcpdump をつくることで深く理解し、AIOps の実現に向けたアプローチを提案します。

# eBPF について紹介

eBPF は Linux カーネルに導入された革新的な技術です。カーネルのソースコード変更やカーネルモジュールのロードを必要とせず、カーネルの機能を安全に効率的に拡張する技術です。ネットワーキング、セキュリティ、オブザーバビリティと様々な分野の課題に取り組む事ができます。

https://ebpf.io/static/e293240ecccb9d506587571007c36739/f2674/overview.png


## eBPF の歴史
BPF ( BSD Packet Filter ) として提案された論文が eBPF の起源となります。BPF はユーザレベルでパケットをキャプチャする新たな機構として提案されました。その後、「Berkeley Packet Filter」の頭文字として BPF が Linux のカーネルに実装されるようになります。tcpdump の中で効果的にパケットをキャプチャする方法として、使われるようになりました。そこから時が経ち、seccomp-bpf が、Linux へ導入されます。これはセキュリティに関する機能であり、パケットフィルタリングだけでないより汎用的な機能へと進化していきました。そして、2014 年から現在の eBPF が導入されました。

| 年代 | 導入 | 内容 | 
| --- | --- | --- |
| 1993 年 | BPF ( BSD Packet Filter ) が，効率的なパケットフィルタリング手法として提案 | ネットワークパケットを通過させるか、拒絶されるかを決定するためのプログラム、フィルタを動かすための擬似的な機械を提案 |
| 1997 年 | BPF ( Berkeley Packet Filter ) が、Linux ( カーネルバージョン : 2.1.75 ) に初めて導入 | tcpdump の中で、パケットのキャプチャに利用 |
| 2012 年 | seccomp-bpf が、Linux ( カーネルバージョン : 3.5 ) に導入 | ユーザ空間のアプリケーションからのシステムコール呼び出し許可、禁止を決定 |
| 2014 年 | eBPF (extended BPF ) が、 Linux (カーネルバージョン : 3.18 ) に導入 |  BPF の命令セットは完全に作り直され、eBPF Map、bpf() システムコールが追加 |


## eBPF の特徴
eBPF は現代の技術進歩に合った特徴を持ちます。Linux カーネルへの機能追加には高い専門性が必要であり、追加する機能についても汎用的でなければなりません。たとえ機能追加しても Linux カーネルのリリースサイクルは 2 から 3 ヶ月ごとであり、実際に利用するオペレーティングシステムで利用可能になるには更に多くの時間が必要となります。柔軟に Linux カーネルを拡張する上で eBPF は最適と言えます。カーネルを拡張する方法として、Linux ではカーネルモジュールがサポートされています。カーネルモジュールはカーネルの振る舞いを変更したり、拡張することが可能な一方で、カーネルプログラミングの高度な技術が必要なだけでなく、注意深く実装するする必要があります。バグを含みカーネルがクラッシュする可能性があるからです。安全に拡張できるという点において、eBPF は eBPF 検証機 ( Verifier ) を機能として提供しています。

eBPF はカーネル空間で動作するため、ユーザ空間との間で発生するシステムコールによるオーバーヘッドの追加を最小限に抑える事ができます。この特徴により、eBPF でネットワーキングの機能を実装することで、パフォーマンスを向上させる事ができます。また、Kubernetes のようなコンテナ環境下では、それぞれのコンテナに対して (サイドカーとして) ルーティング機能を付与する事なくカーネル空間で全ての通信に適用し、コンテナ数増加に伴う CPU やメモリリソースの増加を抑える事ができます。オブサーバビリティに関しては、アプリケーション単位で個別にプロファイリングを設定する事なく、全体に適用できる特徴を持ちます。

- eBPF はカーネルのソースコードの改変や、カーネルモジュールのロード不要で、安全にカーネルの拡張が可能
- ネットワークのパフォーマンス向上、CPU、メモリリソースの削減
- アプリケーション ・・・ プロファイリング、モニタリング、トレーシングに利用可能
- カーネル・・・監視、セキュリティ、ネットワーキング、ロードバランシングに利用可能

## eBPF を利用したサービス
eBPF はネットワーク、セキュリティ、可観測性など様々な分野で利用可能な技術となります。2016 年頃には、eBPF を利用したサービスが数多く提供されるようになりました。

https://ebpf.io/applications/

具体的な導入事例は以下の通りです。Cilium は、クラウドネイティブなネットワーク、可観測性、セキュリティに関するサービスを提供し、2021 年に CNCF 傘下のプロジェクトとなり、2023 年には Graduated のレベルに移行しています。Google の [Google Kubernetes Engine (GKE) にネットワーキング](https://cloud.google.com/blog/products/containers-kubernetes/bringing-ebpf-and-cilium-to-google-kubernetes-engine?hl=en)として、 AWS の [EKS Anyware にネットワーキングとセキュリティ](https://isovalent.com/blog/post/2021-09-aws-eks-anywhere-chooses-cilium/?utm_source=website-cilium&utm_medium=referral&utm_campaign=cilium-blog)として採用されるなど代表的なプロジェクトです。Tetragon は、Cilium Enterprise の機能からセキュリティに関するサービスを切り出した OSS のサービスになります。Falco や Istio に関しても CNCF 傘下で Graduated のレベルに移行したプロジェクトとなります。他にも Datadog や NewRelic といった監視サービスでも利用され、NewRelic では継続的なアプリケーションプロファイリングを可能にしています。

- Cilium : ネットワーク、可観測性、セキュリティに関するサービスを提供
    - Tetragon : Cilium プロジェクトの一つでセキュリティに特化したサービス
- Falco : ランタイム・セキュリティ監査とインシデント対応
- Istio : Ambient モードで eBPF を利用し、ネットワーク、可観測性、セキュリティに関するサービスを提供
- Datadog : ネットワーク監視、システム監視、ファイルシステム監視、プロセス実行監視
- NewRelic : 買収した Pixie でアプリケーションプロファイリング


## eBPF の仕組み
eBPF の中心的な機能はプログラムです。eBPF バイトコードをカーネルの内部に動的にロードし、カーネル内の様々なポイントにアタッチすることで、関数のように実行することができます。

![](/images/article-0008/trigger-function.png)

https://ebpf.io/what-is-ebpf/

作成した eBPF プログラムは、 Clang や LLVM などのツールによって BPF バイトコードにコンパイルされます。eBPF アプリケーションは、bpf() システムコールによって eBPF バイトコードをカーネルに渡しますが、カーネルにロードする前に eBPF 検証機 (Verifier) によってバイトコードの安全性を検証します。ループが有限時間で完了するか、コードサイズが大き過ぎないかといった観点で検証され、安全ではないと判断された場合は、スタックトレースにエラーを出力しロードされずに終了します。検証に成功し正常にロードが完了すると、設定したフックポイント ( kprobes、uprobes、tracepoints、perf_events など ) にフックし、イベントが発生するとバイトコードへのコールバックがトリガされます。バイトコード内のロジックは eBPF 仮想マシンで実行されます。JIT コンパイラが搭載されており、eBPF の命令セットからマシン固有の命令セットへの変換、プログラムの実行速度の最適化が実施されます。これにより、カーネルコードやカーネルモジュールとしてロードされたコードと同等の効率で実行されます。eBPF の Map を利用することで、bpf() システムコールを通して、ユーザ空間とのやりとりが可能になります。

![](/images/article-0008/load-ebpf-byte-code.png)


# tcpdump で、eBPF を深く理解
eBPF をより深く理解するために tcpdump をつくります。tcpdump はネットワークを流れるパケットをリアルタイムでキャプチャして解析できるコマンドラインツールです。ネットワークトラフィックの分析やセキュリティ分析に用いられます。

![](/images/article-0008/tcpdump-demo.gif)

主に以下の二つを作成します。eBPF プログラムはネットワークのパケットをリアルタイムで取得します。eBPF アプリケーションは取得したパケットを解析し、UI 上に分析した結果を表示します。

1. eBPF プログラム ( カーネル内で実行されるプログラム )
2. eBPF アプリケーション ( ユーザ空間で実行されるプログラム )

## Aya を利用し、Rust で eBPF プログラミング
[Aya](https://github.com/aya-rs/aya) は Rust 製の eBPF フレームワークです。[libbpf](https://github.com/libbpf/libbpf) や [bcc](https://github.com/iovisor/bcc) といった C 言語で記述された eBPF ライブラリに依存せずシステムコールを実行するために [libc](https://docs.rs/libc/latest/libc/) クレートのみが使われ、Rust のみでゼロから実装されているところが特徴の一つです。また、非同期処理に [tokio](https://docs.rs/tokio/latest/tokio/) と [async_std](https://docs.rs/async-std/latest/async_std/) の両方をサポートしています。

https://docs.rs/aya/latest/aya/index.html

aya には [aya-template](https://github.com/aya-rs/aya-template) というテンプレートが用意されており、`cargo generate` によってテンプレートから新規プロジェクトを開始する事ができます。

```bash
cargo install cargo-generate
cargo generate https://github.com/aya-rs/aya-template
```

template から作成する際に、フックするポイントを決定する必要があります。tcpdump を作成する場合は、xdp で作成します。kprobe はカーネルに対するプローブになりますが、[tcp_connect](https://elixir.bootlin.com/linux/latest/source/net/ipv4/tcp_output.c#L4249) といったカーネル関数にフックした場合に、[sock](https://elixir.bootlin.com/linux/v6.18.1/source/include/net/sock.h#L354) 構造体に含まれた情報以上の情報が取得できず、tcpdump を実装するには不十分です。[XDP (eXpress Data Path)](https://prototype-kernel.readthedocs.io/en/latest/networking/XDP/introduction.html#what-is-xdp) は、ネットワークパケットを高速処理するために基盤で、データリンク層以上のレベルでパケットのデータを操作する仕組みを提供します。

```bash
🔧   project-name: tcpdump ...
🔧   Generating template ...
? 🤷   Which type of eBPF program? ›
  cgroup_skb
  cgroup_sockopt
  cgroup_sysctl
  classifier
  fentry
  fexit
  kprobe
  kretprobe
  lsm
  perf_event
  raw_tracepoint
  sk_msg
  sock_ops
  socket_filter
  tp_btf
  tracepoint
  uprobe
  uretprobe
❯ xdp
```

aya-template によって以下が作成されます。
* {{project-name}} ... eBPF アプリケーション ( ユーザ空間で実行されるプログラム )
* {{project-name}}-ebpf ...  eBPF プログラム ( カーネル内で実行されるプログラム )
* {{project-name}}-common ... ユーザ空間とカーネルで動作するプログラム間のインターフェース


```bash
% tree
.
|-- Cargo.toml
|-- LICENSE-APACHE
|-- LICENSE-GPL2
|-- LICENSE-MIT
|-- README.md
|-- rustfmt.toml
|-- tcpdump
|   |-- Cargo.toml
|   |-- build.rs
|   `-- src
|       `-- main.rs
|-- tcpdump-common
|   |-- Cargo.toml
|   `-- src
|       `-- lib.rs
`-- tcpdump-ebpf
    |-- Cargo.toml
    |-- build.rs
    `-- src
        |-- lib.rs
        `-- main.rs
```


## eBPF プログラム (カーネル内)

1. データの取得
取得するパケット長が 0 となる場合やバウンダリチェックによって閾値を超える場合は早期リターンし `Ok(xdp_action::XDP_PASS)` を返します。データ型なども含めて適切に処置されていないと Verifier によってエラーが出力され弾かれます。

```rust
    let data = ctx.data(); // パケットデータの先頭ポインタ
    let data_end = ctx.data_end(); // パケットデータの終端ポインタ
    let total_len = data_end - data; // パケット長を計算
```

2. RingBuf へのイベント送信
```rust
        let events = &mut *core::ptr::addr_of_mut!(EVENTS);

        // events ( RingBuf ) から PACKET_EVENT_CAPACITY 分のバッファを予約
        if let Some(mut entry) = events.reserve_bytes(PACKET_EVENT_CAPACITY, 0) {
            // パケットデータのコピー
            match load_packet(&ctx, len_u32, &mut entry) {
                Ok(()) => {
                    entry.submit(0); // 送信
                }
                Err(_) => {
                    entry.discard(0); // 破棄
                    return Err(xdp_action::XDP_ABORTED);
                }
            }
        }
 ```


## Maps

https://docs.ebpf.io/linux/concepts/maps/

## Vrerifier

https://docs.ebpf.io/linux/concepts/verifier/



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


注意すべき点は、ログを絞らないと大量のログが出力される。

```bash
$ cat /proc/kallsyms
# 関数の引数を取得
$ perf probe --vars <function>
# トレースポイントを取得
$ ls /sys/kernel/debug/tracing/ebents
```

# 結論
eBPF の良さは、ネットワーク、セキュリティ、可観測性などカバレッジの広さにあると思う。


# ご参考

論文
* [The BSD Packet Filter: A New Architecture for User-level Packet Capture](https://www.tcpdump.org/papers/bpf-usenix93.pdf)

記事とブログ
* [eBPF Docs](https://docs.ebpf.io/)
* [XDP - eXpress Data Path](https://prototype-kernel.readthedocs.io/en/latest/networking/XDP/introduction.html)
* [AYA](https://docs.rs/aya/latest/aya/index.html)
* [BPF for security—and chaos—in Kubernetes](https://lwn.net/Articles/790684/)
* [Cilium - eBPF-based Networking, Observability, Security](https://cilium.io/)
* [Datadog - eBPF Overview](https://www.datadoghq.com/knowledge-center/ebpf/)

参考書籍
* [入門 eBPF --- Linux カーネルの可視化と機能拡張](https://www.oreilly.co.jp/books/9784814400560/)

動画
* [eBPF を活用した Kubernetes セキュリティ: Tetragon 完全ガイド](https://www.youtube.com/watch?v=xGcCsIJ5AVU)
