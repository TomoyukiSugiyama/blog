---
title: "eBPF を学び、未来をつくるための温故知新"
emoji: "🐳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["eBPF", "Kubernetes", "AI"]
published: false
---

# 前書き
こんにちは、ハードウェア、組み込み ( 生産技術 ) からソフトウェア ( SRE ) の業界に転向したエンジニアです。転向から 3 年程度が経ち、様々な技術に触れることで飽きることなく学び続けることができ、緩やかな成長を実感しています。昨今、AI 技術が著しく成長していることを AI コーディングなどから身にしみて感じておりますが、SRE としてはどのように AI をインフラに組み込み、基盤技術と AI による相乗効果を発揮させるのか、AI と既存技術の親和性はあるのか、といった点について考えるようになりました。Kubernetes ブログでは、[Headlamp AI Assistant](https://kubernetes.io/blog/2025/08/07/introducing-headlamp-ai-assistant/) という AI による Kubernetes 環境下でのアシスタントツールが紹介されています。AI と相性の良い技術は様々あると思いますが、その中で eBPF という技術に焦点を当てて話してみたいと思います。

# 概要
本記事は Uzabase Advent Calendar 2025 の 22 日目の記事です。

https://qiita.com/advent-calendar/2025/uzabase

さて、今回は歴史ある技術である eBPF の歴史変遷から現代の利用事例を紹介し、未来の応用に触れていきたいと思います。なぜ eBPF を題材に取り上げたかというと、AIOps やセキュリティの強化を進めるためには必要な技術だと感じているからです。この記事では eBPF について紹介し、eBPF を利用した tcpdump をつくることで深く理解し、AIOps の実現に向けたアプローチを提案します。

:::message
eBPF 仮想マシンの命令セットやアーキテクチャ、bpf() システムコールの詳細には触れません。
:::

# eBPF について紹介

eBPF は Linux カーネルに導入された革新的な技術です。カーネルのソースコード変更やカーネルモジュールのロードを必要とせず、カーネルの機能を安全かつ効率的に拡張する技術です。ネットワーキング、セキュリティ、可観測性と様々な分野の課題に取り組むことができます。

https://ebpf.io/static/e293240ecccb9d506587571007c36739/f2674/overview.png

## eBPF の歴史
BPF ( BSD Packet Filter ) として提案された[論文](https://www.tcpdump.org/papers/bpf-usenix93.pdf)が eBPF の起源となります。BPF はユーザレベルでパケットをキャプチャする新たな機構として提案されました。その後、「Berkeley Packet Filter」の頭文字として BPF が Linux のカーネルに実装されるようになります。BPF は tcpdump の中で効果的にパケットをキャプチャする方法として使われるようになりました。そこから時が経ち、seccomp-bpf が、Linux へ導入されます。これはセキュリティに関する機能であり、パケットフィルタリングだけでない、より汎用的な機能へと進化していきました。そして、2014 年から現在の eBPF が導入されました。

| 年代 | 導入 | 内容 | 
| --- | --- | --- |
| 1993 年 | BPF ( BSD Packet Filter ) が，効率的なパケットフィルタリング手法として提案 | ネットワークパケットを通過させるか、拒絶されるかを決定するためのプログラム、フィルタを動かすための擬似的な機械を提案 |
| 1997 年 | BPF ( Berkeley Packet Filter ) が、Linux ( カーネルバージョン : 2.1.75 ) に初めて導入 | tcpdump の中で、パケットのキャプチャに利用 |
| 2012 年 | seccomp-bpf が、Linux ( カーネルバージョン : 3.5 ) に導入 | ユーザ空間のアプリケーションからのシステムコール呼び出し許可、禁止を決定 |
| 2014 年 | eBPF (extended BPF ) が、 Linux (カーネルバージョン : 3.18 ) に導入 |  BPF の命令セットは完全に作り直され、eBPF Map、bpf() システムコールが追加 |

## eBPF の特徴
eBPF は現代の技術進歩に合った特徴を持ちます。Linux カーネルへの機能追加には高い専門性が必要であり、追加する機能についても汎用的でなければなりません。たとえ機能を追加しても Linux カーネルのリリースサイクルは 2 から 3 ヶ月ごとであり、実際に利用するオペレーティングシステムで利用可能になるには更に多くの時間が必要となります。柔軟に Linux カーネルを拡張する上で eBPF は最適と言えます。カーネルを拡張する方法として、Linux ではカーネルモジュールがサポートされています。カーネルモジュールはカーネルの振る舞いを変更したり、拡張することが可能な一方で、カーネルプログラミングの高度な技術が必要なだけでなく、注意深く実装する必要があります。バグを含みカーネルがクラッシュする可能性があるからです。安全に拡張できるという点において、eBPF は eBPF 検証機 ( Verifier ) を機能として提供しています。

eBPF はカーネル空間で動作するため、ユーザ空間との間で発生するシステムコールによるオーバーヘッドの追加を最小限に抑えることができます。この特徴により、eBPF でネットワーキングの機能を実装することで、パフォーマンスを向上させることができます。また、Kubernetes のようなコンテナ環境下では、それぞれのコンテナに対して ( サイドカーとして ) ルーティング機能を付与することなくカーネル空間で全ての通信に適用し、コンテナ数増加に伴う CPU やメモリリソースの増加を抑えることができます。Istio の サイドカーパターンと eBPF を利用した Ambient モードで評価した[ベンチマーク結果](https://istio.io/latest/docs/ops/deployment/performance-and-scalability/)からもパフォーマンスの改善が判断できます。可観測性に関しては、アプリケーション単位で個別にプロファイリングを設定することなく、全体に適用できる特徴を持ちます。

- eBPF はカーネルのソースコードの改変や、カーネルモジュールのロード不要で、安全にカーネルの拡張が可能
- ネットワークのパフォーマンス向上、CPU、メモリリソースの削減
- アプリケーションのプロファイリング、モニタリング、トレーシングに利用可能
- 監視、セキュリティ、ネットワーキング、ロードバランシングに利用可能

## eBPF を利用したサービス
eBPF はネットワーク、セキュリティ、可観測性など様々な分野で利用可能な技術となります。2016 年頃には、eBPF を利用したサービスが数多く提供されるようになりました。

https://ebpf.io/applications/

具体的な導入事例は以下の通りです。Cilium は、クラウドネイティブなネットワーク、可観測性、セキュリティに関するサービスを提供し、2021 年に CNCF 傘下のプロジェクトとなり、2023 年には Graduated のレベルに移行しています。Google の [Google Kubernetes Engine (GKE) にネットワーキング](https://cloud.google.com/blog/products/containers-kubernetes/bringing-ebpf-and-cilium-to-google-kubernetes-engine?hl=en)として、AWS の [EKS Anywhere にネットワーキングとセキュリティ](https://isovalent.com/blog/post/2021-09-aws-eks-anywhere-chooses-cilium/?utm_source=website-cilium&utm_medium=referral&utm_campaign=cilium-blog)として採用されるなど代表的なプロジェクトです。Cilium は、[iptables を BPF に置き換える](https://cilium.io/blog/2018/04/17/why-is-the-kernel-community-replacing-iptables/)ことでネットワークパフォーマンスの改善を達成しています。 Tetragon は、Cilium Enterprise の機能からセキュリティに関するサービスを切り出した OSS のサービスになります。Falco や Istio に関しても CNCF 傘下で Graduated のレベルに移行したプロジェクトとなります。他にも Datadog や NewRelic といった監視サービスでも利用され、NewRelic では継続的なアプリケーションプロファイリングを可能にしています。

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

作成した eBPF プログラムは、Clang や LLVM などのツールによって BPF バイトコードにコンパイルされます。eBPF アプリケーションは、bpf() システムコールによって eBPF バイトコードをカーネルに渡しますが、カーネルにロードする前に eBPF 検証機 ( Verifier ) によってバイトコードの安全性を検証します。ループが有限時間で完了するか、コードサイズが大き過ぎないかといった観点で検証され、安全ではないと判断された場合は、スタックトレースにエラーを出力しロードされずに終了します。検証に成功し正常にロードが完了すると、設定したフックポイント ( kprobes、uprobes、tracepoints、perf_events など ) にフックし、イベントが発生するとバイトコードへのコールバックがトリガされます。バイトコード内のロジックは eBPF 仮想マシンで実行されます。JIT コンパイラが搭載されており、eBPF の命令セットからマシン固有の命令セットへの変換、プログラムの実行速度の最適化が実施されます。これにより、カーネルコードやカーネルモジュールとしてロードされたコードと同等の効率で実行されます。eBPF の Map を利用することで、bpf() システムコールを通して、ユーザ空間とのやりとりが可能になります。

![](/images/article-0008/load-ebpf-byte-code.png)

## Verifier
eBPF の特徴は安全性にあります。ここでの安全性とは、いかなる場合もカーネルを破壊しないこと、システムのセキュリティポリシーに違反しないことを指します。具体的なルールは、妥当な時間内にプログラムが終了すること、任意のメモリを読みとり機密情報が漏洩しないこと、隣接するメモリに機密情報が保存されている可能性があるため、パケット境界の外側にアクセスできないこと、などがあります。Verifier は実際に eBPF プログラムを実行するのではなく、一命令ずつ参照し、レジスタの状態の履歴を [bpf_reg_state](https://github.com/torvalds/linux/blob/master/include/linux/bpf_verifier.h) 構造体に保存し評価します。

https://docs.ebpf.io/linux/concepts/verifier/

## Map
eBPF における Map は、eBPF プログラムとユーザ空間からアクセス可能なデータ構造を指します。一般的なユースケースでは、ユーザ空間で設定情報を書き込み、eBPF プログラムからその情報を取得する。eBPF プログラム間で情報を共有する。eBPF プログラムで取得したメトリクスを、ユーザ空間でメトリクスを表示するといった用途で利用されています。Linux の bpf.h には様々な種類の [bpf_map_type](https://github.com/torvalds/linux/blob/master/include/uapi/linux/bpf.h#L980-L1031) が定義されています。用途にあった Map を選択し利用します。

https://docs.ebpf.io/linux/concepts/maps/

# tcpdump で、eBPF を深く理解
eBPF をより深く理解するために tcpdump をつくります。tcpdump はネットワークを流れるパケットをリアルタイムでキャプチャして解析できるコマンドラインツールです。ネットワークトラフィックの分析やセキュリティ分析に用いられます。

![](/images/article-0008/tcpdump-demo.gif)

主に以下の二つを作成します。eBPF プログラムはネットワークのパケットをリアルタイムで取得します。eBPF アプリケーションは取得したパケットを解析し、UI 上に分析した結果を表示します。

1. eBPF プログラム ( カーネル内で実行されるプログラム )
2. eBPF アプリケーション ( ユーザ空間で実行されるプログラム )

## Aya を利用し、Rust で eBPF プログラミング
[Aya](https://github.com/aya-rs/aya) は Rust 製の eBPF フレームワークです。[libbpf](https://github.com/libbpf/libbpf) や [bcc](https://github.com/iovisor/bcc) といった C 言語で記述された eBPF ライブラリに依存せずシステムコールを実行するために [libc](https://docs.rs/libc/latest/libc/) クレートのみが使われ、Rust のみでゼロから実装されているところが特徴の一つです。また、非同期処理に [tokio](https://docs.rs/tokio/latest/tokio/) と [async_std](https://docs.rs/async-std/latest/async_std/) の両方をサポートしています。

https://docs.rs/aya/latest/aya/index.html

aya には [aya-template](https://github.com/aya-rs/aya-template) というテンプレートが用意されており、`cargo generate` によってテンプレートから新規プロジェクトを開始することができます。

```bash
cargo install cargo-generate
cargo generate https://github.com/aya-rs/aya-template
```

テンプレートから作成する際に、フックするポイントをあらかじめ決定する必要があります。tcpdump を作成する場合は、xdp で作成します。kprobe はカーネルに対するプローブになりますが、[tcp_connect](https://github.com/torvalds/linux/blob/master/net/ipv4/tcp_output.c#L4257) といったカーネル関数にフックした場合に、[sock](https://github.com/torvalds/linux/blob/master/include/net/sock.h#L359) 構造体に含まれた情報以上の情報が取得できず、tcpdump を実装するには不十分です。[XDP (eXpress Data Path)](https://prototype-kernel.readthedocs.io/en/latest/networking/XDP/introduction.html#what-is-xdp) は、ネットワークパケットを高速処理するための基盤で、データリンク層以上のレベルでパケットのデータを操作する仕組みを提供します。NIC ドライバに近い場所で操作することで、高速処理を実現しています。

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

## eBPF プログラム ( カーネル内 )
eBPF 側では、パケットの到達後、パケットの範囲チェックなどを行い、RingBuf に送信する役割を担います。Aya では、様々な種類の [Map](https://docs.rs/aya-ebpf/latest/aya_ebpf/maps/index.html) が利用可能です。
* 高頻度のイベント転送 : RingBuf
* キー、バリューの保存 : HashMap または PerCPUHashMap
* パフォーマンス分析 : PerfEventArray
* プログラム間連携 : ProgramArray
* ソケット管理 : SockMap または SockHash

他にも多数あり、用途に応じて適切な Map を選択することで、 eBPF プログラムの効率を向上できます。

1. データの取得
取得するパケット長が 0 となる場合やバウンダリチェックによって閾値を超える場合は早期リターンし `Ok(xdp_action::XDP_PASS)` を返します。データ型なども含めて適切に処理されていないと Verifier によってエラーが出力され弾かれます。

```rust
let data = ctx.data(); // パケットデータの先頭ポインタ
let data_end = ctx.data_end(); // パケットデータの終端ポインタ
let total_len = data_end - data; // パケット長を計算
```

2. RingBuf へのイベント送信
EVENTS（ RingBuf ）からバッファを予約します。予約できた場合、load_packet でパケットデータをコピーします。成功時は entry.submit(0) で送信、失敗時は entry.discard(0) で破棄し XDP_ABORTED を返します。

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

## eBPF アプリケーション ( ユーザ空間 )
カーネル空間では、eBPF の XDP プログラムがネットワークインターフェースを通過するパケットをキャプチャし、RingBuf に送信します。RingBuf は、カーネル空間とユーザー空間の間で非同期にデータを転送します。

ユーザー空間では、spawn_packet_reader がバックグラウンドタスクとして RingBuf を監視し、パケットイベントを取得してパースし、Tokio のチャネル ( packet_tx ) 経由で packet_rx へ送信します。同時に、spawn_input_listener が別スレッドでキーボード入力を監視し、入力イベントを input_rx チャネルへ送信します。

メインループの run_app は、tokio::select! で packet_rx からのパケット受信、input_rx からの入力イベント、Ctrl+C シグナルを待機します。パケットを受信すると app.on_packet() で処理し、入力イベントを受信すると app.handle_input() で処理します。ループごとに terminal.draw() で画面を更新し、パケット一覧、詳細、生データをターミナル UI で表示します。

アーキテクチャの概要
```bash
カーネル空間 (eBPF)
    ↓ RingBuf
ユーザー空間
    ├─ spawn_packet_reader → packet_tx → packet_rx
    │                                        ↓
    └─ spawn_input_listener → input_rx ─→ run_app
                                              ↓
                                         terminal (UI描画)
```

1. 初期化
起動時に CLI 引数を解析し ( 監視するインターフェースやポート )、ロガーを初期化、eBPF で必要な memlock 制限を解除します。

```rust
let opt = Opt::parse();           // コマンドライン引数の解析 （ iface, port ）
env_logger::init();               // ロガーの初期化
bump_memlock_rlimit();            // メモリロック制限の解除 （ eBPF に必要 ）
```

2. eBPF プログラムの読み込み
ビルド済みの eBPF バイナリを読み込み、ロガーを設定します。

```rust
let mut ebpf = aya::Ebpf::load(...);  // コンパイル済み eBPF プログラムを読み込み
setup_logger(&mut ebpf).await;        // eBPF ロガーの設定
```

3. RingBuf の取得
eBPF 側の RingBuf を取得してユーザー空間側の RingBuf ハンドルに変換します。

```rust
let ring_buf_map = ebpf.take_map("EVENTS")?;  // eBPF 側の EVENTS マップを取得
let ring_buf = RingBuf::try_from(ring_buf_map)?;  // RingBuf に変換
```

4. eBPF プログラムのアタッチ
eBPF プログラムを指定インターフェースにアタッチし、パケットをカーネルでフックできる状態にします。

```rust
let program: &mut Xdp = ebpf.program_mut("tcpdump")?;
program.load()?;                              // カーネルにロード
program.attach(&iface, XdpFlags::default())?; // 指定ネットワークインターフェースにアタッチ
```

5. 非同期 IO の設定
RingBuf ハンドルを Tokio の AsyncFd に包み、非同期で読み取れるようにします。

```rust
let ring_buf = tokio::io::unix::AsyncFd::with_interest(ring_buf, ...)?;
```

6. チャネルとタスクの起動
パケット転送用のチャネルを作成し、バックグラウンドタスク spawn_packet_reader を起動します。RingBuf から受け取ったイベントをパースし、アプリ側に CapturedPacket として送ります。ターミナル UI を初期化し、別スレッドでキーボード入力リスナーを起動します。入力イベントを非同期チャネルで受け取ります。

```rust
let (packet_tx, packet_rx) = mpsc::channel::<CapturedPacket>(1024);  // パケット用チャネル
let reader_handle = spawn_packet_reader(ring_buf, packet_tx.clone()); // バックグラウンドでパケット読み取り
drop(packet_tx);  // 送信側をドロップ （ reader_handle のみが所有 ）

let mut terminal = setup_terminal()?;         // ターミナル UI の初期化
let input_rx = spawn_input_listener();        // キーボード入力リスナーの起動
```

7. メインアプリケーションループ
メインループ run_app で、パケット受信・入力イベント・Ctrl+C を Tokio の select で待ち合わせながら、都度画面を描画します。パケットはフィルタを通し、一覧・詳細・生データ表示を更新します。

```rust
let app_result = run_app(&mut terminal, packet_rx, input_rx, port).await;
```
* パケット受信 ( `packet_rx` )、
* キーボード入力 ( `input_rx` )、
* ターミナル描画を実行します。


8. クリーンアップ
終了時はターミナルを元に戻し、パケットリーダータスクを中断して終了を待機します。ログも随時フラッシュされます。

```rust
restore_terminal(&mut terminal)?;  // ターミナルを元の状態に復元
reader_handle.abort();              // パケットリーダータスクを中断
reader_handle.await;                // タスクの終了を待機
```

# AIOps の実現に向けたアプローチ
:::message
個人的見解を含めたアプローチとなります。
:::

AI と実行基盤を融合する上で、3 つのステップを踏む必要があると思っています。具体的には、AI で利用するデータの収集基盤を整え ( **Monitoring** )、取得したデータを元に兆候管理、パフォーマンスの改善箇所を検知 ( **Service Desk** )、最終的には分析結果を踏まえて自動復旧する ( **Automation** ) といったステップです。

1. ( **Monitoring** ) まず考えることは、データ収集基盤を整えることでしょう。
* 各サーバのデータを統一されたフォーマットのデータとして収集する。
* CPU、メモリ、ネットワークだけでなく、アプリケーション固有のメトリクス、アクセスログやアプリケーションログを収集する。
* セキュリティに関するメトリクスを収集する。

2. ( **Service Desk** ) データを元にした兆候管理、パフォーマンス最適化
* CPU、メモリ、ネットワーク、アプリケーション固有のメトリクス、アプリケーションのログは、障害検知に利用する。また、ボトルネックになっているリソースを調整することでパフォーマンスを改善する。
* セキュリティのメトリクスから、不正アクセスなどの異常を検知する。

3. ( **Automation** ) 分析結果を元にした自動復旧、最適化
* 障害に対して、自動で復旧する。
* ネットワークのメトリクスから、リアルタイムに通信経路を最適化する。
* セキュリティインシデントの発生を検知し、アクセス制御をリアルタイムで強制する。

## Tetragon の導入事例
Monitoring のステップでセキュリティに関するメトリクスを取得する方法として、eBPF をベースとした Tetragon の活用が有効です。Tetragon は、Kubernetes に対応した柔軟なセキュリティ観測およびランタイム強制ツールであり、eBPF を使用してポリシーとフィルタリングを直接適用することで、観測オーバーヘッドの削減、あらゆるプロセスの追跡、ポリシーのリアルタイム強制を可能にします。

https://tetragon.io/

セキュリティポリシーを定義し、ポリシーに反したものをログとして出力します。注意すべき点は、ログを絞り込まないと大量のログが出力されます。Kubernetes 環境に以下のポリシーを適用することで、kprobe を利用しカーネル関数の `tcp_connect` に対してフックし、`10.111.11.111` に TCP 接続しようとした時にログが出力されます。

```yaml
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: "deny-connect"
spec:
  kprobes:
  - call: "tcp_connect"
    message: "TCP接続しようとしています"
    syscall: false
    args:
    - index: 0
      type: "sock"
    selectors:
    - matchArgs:
      - index: 0
        operator: "DAddr"
        values:
          - 10.111.11.111
```

kprobe の[フックポイント](https://tetragon.io/docs/concepts/tracing-policy/hooks/#kprobes)は以下で確認することができます。

```bash
# フックポイントの取得
$ cat /proc/kallsyms
```

# 結論
eBPF の良さは、ネットワーク、セキュリティ、可観測性などカバレッジの広さにあると思います。一方で、カーネルに関する専門性が必要になるため導入のハードルの高さもあります。セキュリティに関して Tetragon はポリシーの適用のみで利用でき、特定のユースケースに柔軟に対応できるため eBPF を初めて導入する際や各サーバにセキュリティのレイヤを設ける場合に活用できると思います。eBPF を利用したプロダクトの発展により、今後より普及し広がりを見せる技術となるでしょう。

# ご参考

論文
* [The BSD Packet Filter: A New Architecture for User-level Packet Capture](https://www.tcpdump.org/papers/bpf-usenix93.pdf)

記事とブログ
* [eBPF Docs](https://docs.ebpf.io/)
* [XDP - eXpress Data Path](https://prototype-kernel.readthedocs.io/en/latest/networking/XDP/introduction.html)
* [AYA](https://docs.rs/aya/latest/aya/index.html)
* [BPF for security—and chaos—in Kubernetes](https://lwn.net/Articles/790684/)
* [Cilium - eBPF-based Networking, Observability, Security](https://cilium.io/)
* [Tetragon](https://tetragon.io/)
* [Datadog - eBPF Overview](https://www.datadoghq.com/knowledge-center/ebpf/)
* [Kubernetes ブログ](https://kubernetes.io/blog/2025/08/07/introducing-headlamp-ai-assistant/)
* [Istio - Performance and Scalability](https://istio.io/latest/docs/ops/deployment/performance-and-scalability/)

書籍
* [入門 eBPF --- Linux カーネルの可視化と機能拡張](https://www.oreilly.co.jp/books/9784814400560/)
* [マスタリング TCP/IP 入門編](https://www.ohmsha.co.jp/book/9784274224478/)

動画
* [eBPF を活用した Kubernetes セキュリティ: Tetragon 完全ガイド](https://www.youtube.com/watch?v=xGcCsIJ5AVU)
