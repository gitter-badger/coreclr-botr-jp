CLRスレッディング概要
======================
(これは https://github.com/dotnet/coreclr/blob/8d3936bff7ae46a5a964b15b5f0bc3eb8d4e32db/Documentation/botr/threading.md の日本語訳です。対象rev.は 8d3936b）

マネージドスレッド対ネイティブスレッド
==========================

マネージドコードは"マネージドスレッド"上で動作します。これはオペレーティングシステムが提供するネイティブスレッドとは区別されます。
ネイティブスレッドは物理マシン上でネイティブコードを実行するスレッドですが、マネージドスレッドはCLRの仮想マシン上でコードを実行する仮想スレッドです。

JITコンパイラーがILの"仮想的な"命令を物理マシン上で動作するネイティブの命令に対応付けるのと同様に、
CLRのスレッド基盤は"仮想的な"マネージドスレッドをオペレーティングシステムが提供するネイティブスレッドに対応付けます。

ある1つのマネージドスレッドはどんなときでも、実行のために1つのネイティブスレッドが割り当てられているか、割り当てられていないかのどちらかです。
たとえば、（"new System.Threading.Thread"によって）作成されたマネージドスレッドが、まだ（System.Threading.Thread.Startによって）開始されていなければ、そのマネージドスレッドにはネイティブスレッドが割り当てられていません。
同様に、原理的には、ある1つのマネージドスレッドが実行中に複数のネイティブスレッド間を移動する可能性もあります。とはいえ実際には、現在のCLRはそのような動作をサポートしていません。

マネージドコードに対して公開されているThreadインターフェースは、背後にあるネイティブスレッドの詳細を意図的に隠しています。それは次の理由からです。

- マネージドスレッドは必ずしも1つのネイティブスレッドに対応付けられているわけではありません。（もしかしたらどのネイティブスレッドにも対応付いていないかもしれません。）
- オペレーティングシステムによって、ネイティブスレッドの抽象も異なります。
- 原理的には、マネージドスレッドとは"仮想化されている"ものです。

CLRはマネージドスレッドの抽象に相当するものを提供します。そしてそれはCLR自身に実装されているものです。
たとえば、CLRはオペレーティングシステムのスレッドローカルストレージ（TLS）機構を露出するのではなく、その代わりに管理された"スレッド静的"な変数を提供します。
同様に、CLRはネイティブスレッドの"スレッドID"を露出するのではなく、その代わりにOSとは独立して生成された"マネージドスレッドID"を提供します。
ただし、背後にあるネイティブスレッドの詳細情報の中には、診断に利用する目的のために、System.Diagnostics名前空間の型を通じて取得できるものもあります。

マネージドスレッドには、ネイティブスレッドでは通常必要とされない追加の機構も必要です。
まず、マネージドスレッドはGC参照を自身のスタック上に保持するので、CLRはGCが起こるたびにそれらの参照を列挙（して、おそらく修正）することができなければなりません。そのために、CLRは各マネージドスレッドを"サスペンド"する（すべてのGC参照を見つけられる時点で停止する）必要があります。
次に、AppDomainがアンロードされた場合、CLRはそのAppDomainにはコードを実行しているスレッドがないことを保証しなければなりません。そのために、スレッドをAppDomainからほどくことができる必要があります。CLRはそのようなスレッドにThreadAbortExceptionを注入します。

データ構造
===============

すべてのマネージドスレッドには関連づいたThreadオブジェクト（訳注：System.Threading.Threadクラスとは異なった、内部的なデータ構造）があります。
Threadオブジェクトは[threads.h][threads.h]で定義されています。
このオブジェクトは対応するマネージドスレッドについてVMが知っておくべきことをすべて追跡しています。
VMが知っておくべきことの中には _不可欠_ なもの、たとえばスレッドの現在のGCモードやFrameのチェインなどもあれば、
単に性能上の理由でスレッドごとに割り当てられていること（たとえばアリーナ形式の高速なメモリアロケーターなど）も多くあります。

ThreadオブジェクトはすべてThreadStore（こちらも[threads.h][threads.h]で定義されています）に格納されます。
これは既知のThreadオブジェクトをすべて格納する単純なリストです。
マネージドスレッドをすべて列挙するには、まずThreadStoreLockを獲得して、それからThreadStore::GetAllThreadListを使ってThreadオブジェクトをすべて列挙します。
このリストにはその時点でネイティブスレッドが割り当てられていないマネージドスレッド（たとえば、まだ開始していないとか、あるいは割り当てられたネイティブスレッドがすでに終了したなど）も含まれます。

[threads.h]: (https://github.com/dotnet/coreclr/blob/master/src/vm/threads.h)

現在ネイティブスレッドが割り当てられているマネージドスレッドは、ネイティブコードからアクセス可能です。
それには、割り当てられているネイティブスレッドがそれぞれ持っている、ネイティブスレッドのローカルストレージ（TLS）スロットを介します。
それにより、そのネイティブスレッド上で動作しているコードは、GetThread()を呼ぶことで対応するThreadオブジェクトを得ることができます。

それに加えて、多くのマネージドスレッドは、 _管理された_ Threadオブジェクト（System.Threading.Thread）を持っています。これはネイティブスレッドオブジェクトとは区別されるものです。
このマネージドスレッドオブジェクトは、マネージドコードがスレッドと相互作用するためのメソッド群を提供します。
また、このオブジェクトの大部分はネイティブスレッドオブジェクトが提供する機構のラッパーです。
現在のマネージドスレッドオブジェクトは、（マネージドコードから）Thread.CurrentThreadを介してアクセス可能です。

デバッガーでは、"!Threads"というSOS拡張コマンドを使用して、ThreadStore内のThreadオブジェクトをすべて列挙することができます。

スレッドの生存期間
================

マネージドスレッドは次の状況で生成されます。

1. マネージドコードがSystem.Threading.Threadを使って新しいスレッドを生成するようにCLRに明示的に要求するとき。
2. CLRが特定のマネージドスレッドを直接生成するとき（下記の「特別なスレッド」を参照のこと）。
3. まだマネージドスレッドに対応付けられていないネイティブスレッド上で、ネイティブコードが（「逆P/Invoke」またはCOM相互運用を使用して）マネージドコードを呼び出すとき。
4. あるマネージドプロセスが開始するとき（プロセスのメインスレッドがMainメソッドを呼び出します）。

上記の #1 と #2 の場合は、CLRにはマネージドスレッドの背後にあるネイティブスレッドを生成する責任があります。
これはマネージドスレッドが実際に _開始_ するまでは実施されません。
開始された後は、生成されたネイティブスレッドはCLRが「所有」し、CLRはネイティブスレッドの生存期間について責任があります。
これらの場合では、CLRは生成されたスレッドの存在を認識しています。そもそもスレッドを生成したのはCLRだという事実があるからです。

上記の #3 と #4 の場合は、マネージドスレッドが生成される前からネイティブスレッドは存在しており、CLRの外にあるコードによって所有されています。
CLRはそのネイティブスレッドの生存期間について責任がありません。
CLRはネイティブスレッドがマネージドコードを呼び出そうとした時に初めてそれらのスレッドの存在を認識します。

ネイティブスレッドが終了した時はDllMain関数を通じてCLRに通知されます。これはOSの「ローダーロック」の内側で起こるため、この通知を処理する際に（安全に）できることはほとんどありません。ですので、終了するスレッドは、マネージドスレッドに関連づいたデータ構造について、破棄するのではなく単に「終了」とマークした上で、ファイナライザースレッドに開始のシグナルを送ります。それを受けてファイナライザースレッドはThreadStore内のスレッドを一通り眺め、終了していて _かつ_ マネージドコードから到達不可能なスレッドがあれば破棄します。

サスペンド（一時中断）
==========

CLRがGCを実行するためにはすべてのマネージドオブジェクトを見つけ出せる必要があります。
マネージドコードは定常的にGCヒープにアクセスして、スタックやレジスタに保持されている参照を操作しています。
CLRはすべてのマネージドスレッドが停止していることを保証しなければなりません。
それによって（ヒープが変更されないので）、安全に確実にすべてのマネージドオブジェクトを見つけ出すことができます。
スレッドは _セーフポイント_ でのみ停止します。その時に、生存している参照を探すためにレジスタとスタック位置を調査できます。

別の方法は、GCヒープ、および全スレッドのスタックとレジスターの状態が、複数のスレッドからアクセスされる「共有状態」になることです。
世のほとんどの共有状態と同様に、状態を守るにはある種の「ロック」が要求されます。
マネージドコードはヒープにアクセスする間そのロックを保持しなければなりませんし、ロックを開放できるのはセーフポイントに達したときだけです。

CLRはそのロックをスレッドの「GCモード」として参照します。
「協調（cooperative）モード」のスレッドは自身のロックを保持します。
GCを進行させるためには、そのスレッドは（ロックを解放することで）GCと「協調」しなければなりません。
「割り込み（preemptive）モード」のスレッドは自身のロックを保持しません――GCは「割り込み」によって進行する可能性があります。なぜならそのスレッドはGCヒープにアクセスしていないことが分かっているからです。

GCは全マネージドスレッドが「割り込み」モードにある（ロックを保持していない）時だけ進行します。全マネージドスレッドを割り込みモードに移行するプロセスは「GCサスペンド」または「実行エンジン（EE）の中断」と呼ばれています。

この「ロック」をナイーブに実装するとすれば、個々のマネージドスレッドがGCヒープにアクセスするたびに本物のロックを実際に獲得・解放するというものになるでしょう。
その場合GCは単にそれぞれのスレッドについてロックを獲得しようと試みるだけでよくなります。
全スレッドのロックを獲得してしまえば、GCを実行しても安全ということになります。

しかしながら、このナイーブな手法は2つの理由で満足のいくものではありません。
1つは、マネージドコードがロックを獲得・解放するために（少なくとも、GCがロックを獲得しようとしていないかチェックする、いわゆる「GCポーリング」に）多くの時間を使わないとならないことです。もう1つは、JITが「GC情報」を出力しないとならないことです。それはJIT化されたコードのあらゆる点におけるスタックとレジスターのレイアウトを表すものです。この情報は多量のメモリを必要とするでしょう。

私たちはこのナイーブなアプローチを洗練させ、JIT化されたマネージコードを「部分的に割り込み可能」なコードと「完全に割り込み可能」なコードの2つに分離しました。
部分的に割り込み可能なコードでは、セーフポイントは他メソッドの呼び出しと、明示的な「GCポーリング」の場所だけとなります。
そしてそこでは、JITはGCが延期されているかチェックするコードを出力します。GC情報はそれらの場所においてだけ出力されればよくなります。
完全に割り込み可能なコードでは、すべての命令がセーフポイントであり、JITはすべての命令についてGC情報を出力します――ただしJITはGCポーリングを出力しません。
その代わり、完全に割り込み可能なコードはスレッドのハイジャック（本ドキュメントの後ろの方で説明します）によって「割り込まれる」可能性があります。
完全に割り込み可能なコードと部分的に割り込み可能なコードのどちらを出力するかをJITが判断する際にはヒューリスティックスが用いられます。
そのヒューリスティックスはコード品質、GC情報のサイズ、GCサスペンドの遅延時間について最善のトレードオフを発見するものです。

ここまでを踏まえて、3つの基礎的な処理が定義されています。協調モードへの突入、協調モードからの脱出、そして実行エンジン（EE）の中断です。

協調モードへの突入
-------------------------

Thread::DisablePreemptiveGCを呼ぶことでスレッドは協調モードに入ります。これは現在のスレッドの「ロック」を獲得します。詳細は以下の通りです。

1. もしGCが処理中（GCがロックを保持している）なら、GCが完了するまでブロックします。
2. スレッドを協調モードにあるとマークします。スレッドが再び割り込みモードに入るまでGCは進行しません。

これらの2つのステップはアトミックに進行します。

協調モードからの脱出
------------------------

Thread::EnablePreemptiveGCを呼ぶことでスレッドは割り込みモードに入ります（ロックを解放します）。
これは単に、スレッドが協調モードではなくなったとマークして、GCスレッドに対して進行してもよいと伝えるだけです。

実行エンジン（EE）の中断
-----------------

GCが稼働しようとするときの最初のステップはEEを中断することです。これはGCHeap::SuspendEEで実行されます。プロセスは以下の通りです。

1. グローバルなフラグ（g\_fTrapReturningThreads）を立てて、GCが動作中であることを示します。協調モードに入ろうとするスレッドはすべて、GCが完了するまでブロックされます。
2. 現在協調モードで動作しているスレッドをすべて探し出します。見つかったスレッドのそれぞれについて、スレッドをハイジャックして協調モードから脱出させようとします。
3. 協調モードで動作するスレッドがなくなるまで繰り返します。

ハイジャック
---------

Thread::SysSuspendForGCによってGCサスペンドのためのハイジャックが起こります。このメソッドは現在協調モードで動作しているすべてのスレッドについて、「セーフポイント」で協調モードから抜けるように強制しようとします。
これは（ThreadStoreを調査して）すべてのマネージドスレッドを列挙したうえで、現在協調モードで動作しているスレッドごとに以下のことを行います。

1. 裏側にあるネイティブスレッドを一時停止します。これにはWin32のSuspendThread APIが用いられます。このAPIはスレッドの実行を強制的に止めます。停止地点は実行中のランダムな場所です（セーフポイントである必要はありません）。
2. GetThreadContextでスレッドの現在のコンテキストを取得します。これはOSの概念で、コンテキストとはスレッドの現在のレジスター状態を表すものです。これによってスレッドの命令ポインターを調査できるので、したがって現在実行中のコードの種類を判別できます。
3. スレッドが協調モードか再確認します。そのスレッドが一時停止できるようになる前にすでに協調モードから抜け出ているかもしれないからです。もしそうなら、そのスレッドは危険な領域にいます。スレッドは任意のネイティブコードを実行している可能性があり、デッドロックを避けるには即座に再開されなければなりません。
4. スレッドがマネージドコードを実行しているかチェックします。可能性としてあり得るのは、スレッドが協調モードでネイティブのVMコードを実行しているという状況です（下記の「同期」の節を参照のこと）。その場合には上のステップ同様、スレッドは即座に再開されなければなりません。
5. このステップまで来れば、スレッドはマネージドコード内で中断しています。コードが完全に割り込み可能なのか部分的に割り込み可能なのかによって、次のどちらかが実行されます。
  * 完全に割り込み可能な場合は、どの地点でGCを実行しても安全です。なぜならスレッドは、定義上、セーフポイントにいるからです。このままスレッドを一時停止させることも（安全であれば）考えられますが、OSには多くの歴史的なバグがあるのでそのように動作させることはできません。すでに取得しているコンテキストが壊れる可能性があるためです。その代わりに、スレッドの命令ポインターは上書きされ、より完全なコンテキストをキャプチャするスタブにリダイレクトされます。それから協調モードから出て、GCが完了するのを待ち、再度協調モードに入り、スレッドを以前の状態に復元します。
  * 部分的に割り込み可能な場合は、スレッドは、定義上、セーフポイントにはいません。しかし、呼び出し元は（メソッドへの推移で）セーフポイントへと移動します。それを踏まえて、CLRは最上位のスタックフレームのリターンアドレスを「ハイジャック」（スタック上のその場所を物理的に上書き）し、完全に割り込み可能な場合と同様のスタブへと置き換えます。メソッドから制御が戻るときには、もはや実際の呼び出し元へと戻ることはなく、その代わりにスタブへと移動します（その地点より前に、メソッドはJITによって挿入されたGCポーリングを実行するかもしれません。その場合は協調モードから抜け、ハイジャックは取り消されることになります）。

ThreadAbort / AppDomainのアンロード
==============================

AppDomainをアンロードするためには、CLRはそのAppDomainの中でスレッドが実行されていないことを確認する必要があります。これを達成するために、マネージドスレッドをすべて列挙し、アンロードされるAppDomainに属するスタックフレームを持つスレッドをすべて「中止」します。実行中のスレッドにThreadAbortExceptionが「注入」され、それによってスレッドは巻き戻されます（実行経路に沿って取り消しコードを実行します）。スレッドがAppDomain内で実行されなくなった時点で、ThreadAbortExceptionはAppDomainUnloadedExceptionに変換されます。

ThreadAbortExceptionは特殊なタイプの例外です。これはユーザーコードによってキャッチされるかもしれませんが、ユーザーの例外ハンドラが実行された後で、例外が再スローされることをCLRが保証します。そのためThreadAbortExceptionは「キャッチできない」と呼ばれることもありますが、これは厳密には正しくありません。

ThreadAbortExceptionは、典型的には、単にマネージドスレッドのあるビットを設定して「強制終了中(aborting)」とマーキングすることで「スロー」されます。このビットは、CLRのさまざまな部分で（最も顕著なのは、すべてのP/Invokeから復帰するたびに）チェックされます。また、多くの場合、スレッドを適時に中止するために必要なことはこのビットを設定することだけです。

ただし、例えばスレッドがマネージドループを長い間実行している場合などでは、このビットがチェックされない可能性があります。このようなスレッドをより速く強制終了するために、スレッドは「ハイジャック」され、ThreadAbortExceptionが強制的に投げられます。このハイジャックはGCのサスペンドと同じように行われますが、スレッドがリダイレクトされるスタブはGCの完了を待つのではなく、ThreadAbortExceptionを発生させる点が異なります。

このハイジャックは、マネージドコードでは本質的に任意の時点でThreadAbortExceptionが投げられうることを意味します。このことで、マネージドコードがThreadAbortExceptionに正しく対処するのは非常に困難になっています。したがって、AppDomainのアンロード以外の目的のためにこの機構を使用することは賢明ではありません。AppDomainのアンロードであれば、ThreadAbortによって破損した任意の状態がアプリケーションドメインと一緒にクリーンアップされることが確実になります。

同期: マネージドの場合
========================

マネージドコードは同期プリミティブへのアクセスをいくつも持っていて、それらはSystem.Threading名前空間に集められています。Mutex、Event、SemaphoeオブジェクトのようなネイティブOSプリミティブのラッパーもあれば、BarrierやSpinLockといった抽象もあります。しかし、ほとんどのマネージドコードで使用されている一番の同期メカニズムはSystem.Threading.Monitorです。これは、  _任意のマネージドオブジェクト_ に高性能なロック機能を提供します。また、ロックによって保護された状態の変化を通知する「条件変数」のセマンティクスも提供します。

モニタは、「ハイブリッドロック」として実装されています。すなわち、スピンロックと、ミューテックスのようなカーネルベース​​ロックの、両方の機能を備えています。これは次のような考えによるものです。つまり、ほとんどのロックは短い時間しか保持されないため、ロックの解放を待つには、カーネルを呼び出してスレッドをブロックするよりも、単なるスピンで待機する方が時間がかからないということです。重要なのはCPUサイクルを無駄に回さないことですので、短期間のスピンでロックが取得されなかった場合、実装はカーネルでのブロッキングにフォールバックします。

任意のオブジェクトを潜在的にロック/条件変数として使用できるので、すべてのオブジェクトはロック情報を格納する場所を持っている必要があります。これは「オブジェクトヘッダ」と「同期ブロック」によって実現されています。

オブジェクトヘッダは、すべてのマネージドオブジェクトの先頭にある、マシンワードサイズの1つのフィールドです。これは多くの目的に使用されます。たとえばオブジェクトのハッシュコードを格納しておくなどです。そのような目的の一つに、オブジェクトのロック状態を保持することがあります。オブジェクト単位のデータが多くなってオブジェクトヘッダに収まらなくなった場合は、「同期ブロック」を作成してオブジェクトを「膨張」させます。

同期ブロックは、同期ブロックテーブルに格納され、同期ブロックインデックスから参照されます。同期ブロックと関連づいたオブジェクトは、それぞれのオブジェクトのオブジェクトヘッダに同期ブロックインデックスのインデックスを持っています。

オブジェクトヘッダーと同期ブロックの詳細は、 [syncblk.h][syncblk.h]/[.cpp][syncblk.cpp] に定義されています。

[syncblk.h]: https://github.com/dotnet/coreclr/blob/master/src/vm/syncblk.h
[syncblk.cpp]: https://github.com/dotnet/coreclr/blob/master/src/vm/syncblk.cpp

オブジェクトヘッダに余裕がある間は、Monitorはロックを現時点で保持しているスレッドのマネージドスレッドID（ロックを保持しているスレッドがない場合はゼロ（0））をオブジェクトに保存します。この場合、ロックを取得するのは単純なことです。オブジェクトヘッダのスレッドIDがゼロになるまでスピン待ちした後で、現在のスレッドのマネージドスレッドIDをアトミックにセットするだけです。

このような形で何度かスピンしてもロックを取得できなかった場合や、またはオブジェクトのヘッダが既に他の目的に使われていた場合は、そのオブジェクトには同期ブロックを作成しなければなりません。同期ブロックはいくつかのデータを追加で含んでいます。たとえば現在のスレッドをブロックするために使用することができるイベントです。これによって、スピンを止め、ロックが解除されるのを効率的に待機できます。

（Monitor.WaitとMonitor.Pulseで）条件変数として使用されるオブジェクトは常に膨張させる必要があります。同期ブロックには必要な状態を保持するための十分なスペースがないからです。

同期: ネイティブの場合
=======================

CLRのネイティブ部も同様にスレッド処理を認識する必要があります。それは複数のスレッド上のマネージドコードから呼び出されるためです。よってロックやイベントなどのネイティブな同期機構が必要です。

ITaskHost APIを使うと、ホストはスレッドの作成、破棄、および同期を含むマネージドスレッドの多くの側面を上書きすることができます。ホストがネイティブ同期を上書きできるということがどういう意味を持つかというと、一般的にVMコードはネイティブの同期プリミティブ（クリティカルセクション、ミューテックス、イベントなど）を直接使用することはできず、その代わりこれらに対するVMのラッパーを使用しなければならないということです。

さらに、上述したようにGCサスペンドは特殊な「ロック」であり、CLRのあらゆる側面に影響を与えます。VMのネイティブコードは、GCヒープのオブジェクトを操作する必要がある場合に「協調」モードに入る可能性があります。したがって、「GCサスペンドロック」は、マネージドの場合と同様に、ネイティブVMコードにおいても最も重要な同期機構の一つとなります。

ネイティブVMコードで使用される主な同期メカニズムは、GCモードとCrstです。

GCモード
-------

上述したように、すべてのマネージドコードは協調モードで実行されます。それはGCヒープを操作する可能性があるためです。一般的に、ネイティブコードはマネージドオブジェクトに触らないので、割り込みモードで実行されています。しかし、VM内のいくつかのネイティブコードはGCヒープにアクセスする必要があるため、協調モードで実行する必要があります。

一般的に、ネイティブコードはGCモードを直接操作するのではなく、GCX\_COOPとGCX\_PREEMPの二つのマクロを使用します。これらのマクロは指定のモードに入ります。そして、スコープが終了した時に「所有者」にスレッドを元のモードに戻させます。

GCX\_COOPがGCヒープのロックを効率的に獲得することを理解しておいてください。スレッドが協調モードにある間は、GCは進行しないでしょう。そしてネイティブコードは、マネージドコードのように「ハイジャック」することはできません。よって、スレッドは明示的に割り込みモードに切り替えるまでは協調モードのままです。

したがって、ネイティブコードでは協調モードに入ることは推奨されません。どうしても協調モードに入らなければならない場合には、できるかぎり短時間に抑えるべきです。スレッドは、協調モードにある間はブロックすべきではありません。特に、一般的にはロックを安全に取得することができません。

同様に、GCX\_PREEMPは、スレッドによって保持されていたロックを _解放する_ 可能性があります 。割り込みモードに入る前には細心の注意を払って、すべてのGC参照が適切に保護されるようにする必要があります。

[コードの規則](../coding-guidelines/clr-code-guide.md)に関するドキュメントでは、GCモードを切り替える際の安全を確保するために必要な規律を説明しています。

Crst
----

マネージドコードのための好ましいロック機構はモニターでしたが、それと同様に、VMコードに適したメカニズムがCrstです。モニターと同じようにCrstもハイブリッドロックで、ホストやGCモードを認識します。Crstも「ロックの平準化」によるデッドロック回避を実装しています。[BotRのCrstレベリングのチャプター](../coding-guidelines/clr-code-guide.md#entering-and-leaving-crsts)でそのことが説明されています。

協調モードでCrstを取得することは一般的にはやってはいけないことですが、どうしても必要な箇所には例外事項が作られています。

特別なスレッド
===============

CLRは、マネージドコードによって作成されたスレッドを管理するだけでなく、いくつかの 「特別」なスレッドを作成して自分自身が使用します。

ファイナライザースレッド
----------------

このスレッドは、マネージドコードを実行するすべてのプロセスで作成されます。ファイナライズ可能なオブジェクトがもはや到達可能でないとGCが判断した場合には、そのオブジェクトをファイナライズキューに配置します。GCが終了する時にはファイナライザースレッドに通知され、現在このキューに入っているすべてのオブジェクトのファイナライザーが処理されます。各オブジェクトは、一つずつキューから取り出されてファイナライザーが実行されます。

このスレッドは、CLR内部の様々な維持管理タスクを実行するのにも使われますし、何らかの外部イベントの通知を待つためにも使われます。（たとえば、メモリ不足状態が通知され、GCがより積極的に回収する合図となるなどです。）詳細については、GCHeap :: FinalizerThreadStartを参照してください。

GCスレッド
----------

GCを「コンカレント」または「サーバ」モードで実行している場合、GCは1つ以上のバックグラウンドスレッドを作成し、ガベージコレクションの様々な段階を並列に実行します。これらのスレッドはすべてGCに所有・管理され、マネージドコードを実行することはありません。

デバッガーのスレッド
---------------

CLRは、すべてのマネージドプロセスにそれぞれ一つのネイティブスレッドを保持しています。そのスレッドは、接続されるマネージドデバッガーのために様々なタスクを実行します。

AppDomainのアンロードスレッド
-----------------------

このスレッドはアプリケーションドメインをアンロードする責任があります。これは、AppDominをアンロードするよう要求したスレッドではなく、CLR内部のある独立したスレッドで行われます。それには2つの理由があります。a) アンロード・ロジックのための保証されたスタック空間を提供するためと、b) アンロードを要求し​​たスレッドを、必要に応じてAppDomainの外にほどくことができるようにするためです。

ThreadPoolのスレッド
------------------

CLRのThreadPoolは、ユーザーの「作業項目」を実行するために、マネージドスレッドのコレクションを保持しています。これらのマネージドスレッドはThreadPoolが所有するネイティブスレッドに結びついています。ThreadPoolは、少数のネイティブスレッドを別途保持しています。それらは、「スレッド注入」や、タイマーや、「登録されている待機動作」などの機能を処理するために使われます。