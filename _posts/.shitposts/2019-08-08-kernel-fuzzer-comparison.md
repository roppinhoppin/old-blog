---
layout: post
title: "カーネルファザー比較"
---

- White-box kernel fuzzer
    - [CAB-Fuzz](#cab-fuzz)
- Gray-box kernel fuzzer
    - [Syzkaller](#syzkaller)
    - [TriforceAFL](#triforceafl)
    - [kAFL](#kafl)
    - [Periscope](#periscope)
    - [UnicoreFuzz](#unicorefuzz)
- Black-box kernel fuzzer
    - [Trinity](#trinity)
    - [perf_fuzzer](#perf_fuzzer)
    - [KernelFuzzer](#kernelfuzzer)
    - [DIFUSE](#difuse)
    - [IMF](#imf)
    - [Digtool](#digtool)

**White-box kernel fuzzing**

*White-box fuzzing*の定義は，PUT(Program Under Test)すなわちテスト下にあるプログラムの中身を解析，または実行中に得られた情報を元にテストケースを生成すること．
それによってwhite-box fuzzerというのはPUTの状態空間をシステマチックに知ることができるものとなっている．
また，テイント解析を使う手法もwhite-box fuzzingとして分類される．
ここではwhite-box kernel fuzzingの手法について述べる．

- #### CAB-Fuzz
    未読

**Gray-box kernel fuzzing**

*Gray-box fuzzing*は，white-box fuzzingのようにPUTの中身や実行中に得られた情報を得ることができるがwhite-box fuzzingとは違って様々な制約がある場合にgray-box fuzzingとして分類される（例えばプログラムのバイナリのみしか提供されていないケースなど）．
ここではgrey-box kernel fuzzingの手法について述べる．

- #### Syzkaller
    AFL-gcc的にgccに拡張を施し，全てのカーネルのプログラムのbasic block上にコールバックを仕込んでカバレッジを取得する(CONFIG_KCOVオプション)
    QEMU内でこれらのビルドオプション込みでビルドしたカーネルをファジングする．
    syz-managerがVMのマネジメントを行いsyz-fuzzerがカーネルCONFIG_KCOVオプションによって得られるようになったカーネルのカバレッジ情報を元に入力を生成する．

    ![syzkaller-overview](/images/2019/02/syzkaller-sm.png)


- #### TriforceAFL
    AFLのqemu_modeではユーザーランドのみのエミュレーションを行うことで任意のバイナリに対してカバレッジをフィードバックとしてカバレッジベースのファザーを行っていた．
    そこで、TriforceAFLでは、qemu_modeをカーネルに対してまで拡張し「AFLをLinux Kernelに対して実行する」というアイデアをもとに作られた．
    このTriforceAFLは現在，Linuxのみに対応しているがメカニズム的には他のOSにも応用可能である．

    TriforceAFLでは，
    aflCall(0x0F24)という命令を追加し，forkサーバーの起動，hypercall周りの処理，命令のトレースのオン，オフを行う．
    Guest OSのユーザーランドからカーネルに対してテストケースを投げるプログラム(driver)は，次のことを行う，
    * テストケースを手に入れる
    * aflCallを用いてパーサーに対しての命令のトレースをオンにする
    * テストケースをパースする
    * カーネルの一部または全体に対しての命令のトレースをオンにする
    * パースした入力を元にテストしたいカーネルの機能を呼び出す
    * テストケースが成功した(カーネルパニック等が発生していない)ことをホスト側に通知する

    また面白いQEMUに対するパッチとして，qemu_modeでも存在していたJITが中断された際に起こるバグをcpu-exec.cのBlock Chaining機能を無効化することによって起こさせないようにしたものが存在する．

- #### kAFL
    kAFLは，「カーネルに対してAFLライクなファザーを実行しよう」というコンセプトはTriforceAFLと同じまま，カバレッジベースなファザーを高速に且つカーネルに対して行うというのがメイン．
    kAFLでは，OS非依存かつハードウェア支援機能なやり方を用いて，ファザーが自分自身のカーネルをファジングした時に起きるシステムをまたリブートしなおさなければいけない問題を解決する．
    具体的にはKVMとIntel PTを用いて行う．
    Intel PTは，CPUが実行した命令をメモリ上のバッファに逐次パケットとしてログされていく機能で，本論文ではこの機能を用いてブランチの情報を取得しカバレッジベースのファザーを実現する．
    kAFLでは，KVMに対して，Guest OSが使う一つのvCPUごとに実行をトレースできるように拡張を施している．
    そしてそのトレースされたデータを拡張されたQEMU側から取得しファザーにフィードバックし，ファザーがそのフィードバックをベースに新しいテストケースを生成する．

    本論文のEvaluationで示されている通り，initramfsのファジングをkAFLとTriforceAFLで行った際には，kAFLが３分以内に見つけたパスをTriforceAFLは30分かけて発見したと示されている．

    ![kAFL-overview](/images/2019/02/kAFL-overview.png)

- #### PeriScope
    DMAとMMIOのファジングをフレームワークとして作った初めての論文．根本となるアイデアとしてはドライバによってマップされたMMIOとDMAのマッピングを，ドライバがそのメモリ領域にアクセスした際に動的にトラップする．
    動的にトラップすると言ってもPeriscopoeではpage faultベースでDMAとMMIOのモニタリングをする．この手法の利点はデバイス依存がなく粒度の高いモニタリングができる．

    ![periscope-overview](/images/2019/02/periscope-overview.png)

- #### UnicoreFuzz
    AFL-Unicornとavatarを使って，カーネルのコンポーネントに対してもファジングを行おうという試み．
    AFL-Unicornでは，周辺機器などのエミュレーションが難しいので，周辺機器やドライバ周りのコードは無視をしてカーネル内に存在するパーサーのファジングにフォーカスを向けた論文．（パーサーはデバイス等のレスポンスを待つ必要がないため）
    [avatarフレームワーク](http://www.eurecom.fr/en/publication/5437)が解析対象のデバッガーとバイナリの抽象化レイヤーとしてプロセスのメモリのダンプを行ったりレジスタのコピーを行っている

    ![unicorefuzz-overview](/images/2019/02/unicorefuzz.png)
    具体的系な実装は，avatarを用いてgdb stubと通信し，fuzzing対象となるパーサーに対してブレークポイントを張り，起動したVMがそのアドレスを踏んだ際にはVMをフリーズさせて必要となるレジスタの値を読み出したあとにdynamic mappingのリクエストを待つ．

    Dynamic mappingとは，fuzzer側が必要になった対象のメモリのダンプをhalt状態にあるVMと通信し，必要となるメモリのダンプをダイナミックに行うことでオーバーヘッドを減らしながら，Unicorn側のVMにダンプしたメモリとレジスタをマッピングしていく手法．

    そしてここから，テストを開始する実行を始める．Unicorn側でAFLのforkserverを起動させ，仮に実行中にunmmapedなメモリ領域へのアクセスがあった場合には前回の実行でその領域が要求されたか確認する．仮にそのメモリのダンプが出力されたディレクトリに存在しない場合それはSEGVとしてコーパスに保存される．

    ~~Usenix woot 19にパブリッシュされたものなので，このポスト内の他のファザーと比較すると水準が低い~~


**Black-box kernel fuzzing**

*Black-box fuzzing*は，PUTの中身を見ずに，PUTの入力や出力の挙動だけを観測してソフトウェアをテストする手法．


- #### IMF
    カーネルのAPIはAPIごとに相互に依存関係があったりするため，より深いバグにたどり着くためには，その関数が呼ばれるコンテキストを見ているようなAPIではすぐに入力がリジェクトされてしまっているケースがある．そのためカーネルAPIごとのの依存関係の推論モデルを作成し，それを元により深いバグを検出する．
    ![imf-poc](/images/2019/02/imf-poc.png)
    IMFは，このスニペットのような複雑なコンテキストで呼ばれるようなAPIを効率よくファジングするためになるべく多くのデータを収集しておくことでAPIの引数の依存関係などを調べる．
    この依存関係は，この論文内では*ordering dependence*と*value dependence*の二つに分類されている.
    Ordering dependenceは，A関数という関数があった時にその関数はいつでもB関数が後に呼ばれるような関数の呼び出される順番に依存関係があるものを指す．
    Value dependenceは，関数Aの出力が関数Bの入力になっているような依存関係があるものを指す．

    * Logger

        アプリケーション側に対象となるAPIをフックしてトレースし，
        Evaluationの章で語られている95/105のアプリケーションをテストしてそれらのアプリケーションが使ったIOKitLibの関数をフックしトレースしたデータを得る．
   
    * Inferrer 

        Loggerから得たAPIのログのデータよりAPIごとの*ordering dependence*と*value dependence*の関係をモデル化する．
        1. *filter*関数がL個のAPIのログをLoggerから取得する．
        2. *infer*関数が*ordering dependence*と*value dependence*の依存関係を決定しその結果をモデルに投げる．

    * Fuzzer

        Inferrerで生成したAPIの推論モデルを*generate*関数がCのプログラムを変換して吐く．
        そしてそのファイルをコンパイルすることによってできた実行ファイルを*execute*関数がファジングの設定の入力を元に繰り返しプログラムを実行しバグを探索する．


    ![imf-overview](/images/2019/02/imf-overview.png)

- #### Digtool
    未読

- #### DIFUSE
    未読

- #### Trinity
    未読

- #### perf_fuzzer
    未読

- #### KernelFuzzer
    未読


#### References

<https://lwn.net/Articles/677764/>

<https://www.slideshare.net/DmitryVyukov/syzkaller-the-next-gen-kernel-fuzzer>

<https://www.ndss-symposium.org/wp-content/uploads/ndss2019_04A-1_Song_slides.pdf>

<https://www.ndss-symposium.org/wp-content/uploads/2019/02/ndss2019_04A-1_Song_paper.pdf>

<https://www.usenix.org/system/files/woot19-paper_maier.pdf>

<https://github.com/fgsect/unicorefuzz>

<https://daramg.gift/paper/han-ccs2017.pdf>