ランタイムの例外について全開発者が知るべきこと
============================================================
(これは https://github.com/dotnet/coreclr/blob/master/Documentation/botr/exceptions.mdの日本語訳です。対象rev.は 8d3936b)

Date: 2005

CLRの「例外」について話す際には気を付けるべき重要な区別があります。まずマネージド例外というものがあります。これは、アプリケーションに対してはC#でいうところのtry/catch/finallyのような機構によって公開されますが、それらを実装するランタイム機構もすべて存在しています。それに加えて、ランタイム自身の内部でも例外を使用しています。ランタイム開発者のほとんどは、マネージド例外モデルを構築し公開する方法について考える必要などまずありません。しかしランタイム開発者なら全員が、ランタイムの実装における例外の使われ方については理解しておかなければなりません。これらの区別をはっきりさせる必要がある場合には、このドキュメントでは __マネージド例外__ と __内部例外__ という使い分けをします。前者はマネージドアプリケーションが投げたりキャッチしたりする可能性がある例外を指し、後者はランタイムが自身のエラー処理のために使う __CLRの内部例外__ を指します。とはいえほとんどの場合、このドキュメントではCLRの内部例外を扱います。

例外が問題となる箇所はどこですか？
===========================

例外はほぼどこでも問題となりますが、ほとんどの場合は例外を投げたりキャッチしたりしている関数の中でしょう。なぜならそのようなコードは、例外を投げたり、例外をキャッチして適切に処理するように明示的に書かれなければならないからです。もし、とある関数が自分自身では例外を投げていなかったとしても、例外を投げる別の関数を呼び出している可能性は十分ありますし、その場合は元の関数も、呼び出し先の関数から例外が投げられた際に適切にふるまうように書かれなければなりません。__ホルダー__ を慎重に扱えば、そのようなコードを正しく書くのがとても楽になります。

CLRの内部例外は何が特別なのですか？
==========================================

CLRの内部例外はC++の例外によく似ていますが、完全に同じではありません。Rotor（訳注：シェアードソースCLI実装）はMac OS X、BSD、そしてWindowsでビルド可能です。OSとコンパイラーの違いが影響して、単に標準C++のtry/catchを使うことができません。さらに、CLRの内部例外はマネージド例外の"finally"や"fault"に似た機能を提供しています。

いくつかのマクロを使って、標準C++とほぼ同程度に読み書きしやすい例外処理コードを書くことが可能になっています。

例外をキャッチする
=====================

EX_TRY
------

基本的なマクロは、もちろん EX_TRY / EX_CATCH / EX_END_CATCH です。使用法は次のようなものです。

    EX_TRY
      // Call some function.  Maybe it will throw an exception. 
      Bar();
    EX_CATCH 
      // If we're here, something failed. 
      m_finalDisposition = terminallyHopeless; 
    EX_END_CATCH(RethrowTransientExceptions) 

EX_TRY マクロは単純にtryブロックを導入するものです。これはC++の "try" によく似ていますが、左波かっこ（ { ）も含まれている点が異なります。

EX_CATCH
--------

EX_CATCHマクロは右波かっこ（ } ）を含んでおり、tryブロックを終了させてcatchブロックを開始させます。EX_TRYと同様に、左波かっこ（ { ）を含んだ形でcatchブロックを開始させます。

そして、C++例外との大きな違いがあります。CLR開発者は何をキャッチするか指定する機会がありません。実際、このマクロの組はすべてをキャッチします。C++例外ではない、アクセス違反やマネージド例外などもです。あるコードがただ1種類の例外だけをキャッチするか、あるいは例外の部分集合をキャッチするという必要がある場合には、そのコードはキャッチして例外を調べ、無関係なものはすべて投げ直す必要があるでしょう。

大事なことなので2度言いますが、EX_CATCHマクロはすべてをキャッチします。この挙動は関数が望むものではないことも多々あります。次の2つのセクションで、キャッチするつもりではなかった例外をどう扱うかについて詳細に論じていきます。

GET_EXCEPTION() & GET_THROWABLE() 
---------------------------------

それでは、CLR開発者はどうすれば、何をキャッチしたのか見分けて、それからどうすべきかを判断できるのでしょうか？選択肢はいくつかありますが、要求がどのようなものかに依存します。

まず、キャッチした（C++の）例外が何であれ、それはグローバルなExceptionクラスから派生した何らかのクラスのインスタンスとして受け取ることになるでしょう。それらの派生クラスの中には、たとえばOutOfMemoryExceptionのように、かなりわかりやすいものもあります。また、EETypeLoadExceptionのように、ややドメイン固有のものもあります。それに、他のシステムの例外を単にラップしたクラス、たとえばCLRException（何らかのマネージド例外を参照するOBJECTHANDLEを持っています）やHRException（HRESULTをラップしています）のようなものもあります。元の例外がExceptionから派生していなかった場合は、このマクロはExceptionから派生する何かで例外をラップするでしょう。（気を付けるべきことは、それらの例外はすべてシステムが提供する既知のものだということです。__新しい例外クラスは、コア実行エンジンチームの関与なしに追加するべきではありません！__）

次に、CLRの内部例外には、関連するHRESULTが常に存在します。時には、HRExceptionのように、値は何らかのCOMソースから来たものです。しかし、内部エラーやWin32 APIの失敗も同様にHRESULTを持っています。

最後に、CLR内部のほぼすべての例外はマネージドコードに戻される可能性があるため、内部例外から対応するマネージド例外へのマッピングが存在しています。対応するマネージド例外は必ずしも生成されないかもしれませんが、その可能性は常にあるのです。

さて、これらの特徴を踏まえて、CLR開発者は例外をどのように分類すればよいでしょうか？

多くの場合、分類に必要なものは例外に対応するHRESULTだけです。そしてそれを得るのはとても簡単です。

    HRESULT hr = GET_EXCEPTION()->GetHR();

それ以上の情報を得るにはマネージド例外オブジェクトを利用するのがもっとも簡単であることが多いです。そして、例外がマネージドコードに送られる予定であれば、それが今すぐであれ、今はキャッシュしておいて後で送られるのであれ、マネージドオブジェクトは結局必要になります。しかも例外オブジェクトを得るのも同じように簡単です。もちろん、得られるのはマネージドオブジェクト参照ですから、いつものルールはすべて当てはまります。

    OBJECTREF throwable = NULL;
    GCPROTECT_BEGIN(throwable);
    // . . .
    EX_TRY
        // . . . do something that might throw
    EX_CATCH
        throwable = GET_THROWABLE();
    EX_END_CATCH(RethrowTransientExceptions)
    // . . . do something with throwable
    GCPROTECT_END()

時には、C++例外オブジェクトが必要になることを避けられないこともありますが、しかしそのほとんどは例外の実装の内部です。C++例外の型が正確に何であるかが重要な場合は、RTTIに似た軽量な関数群が用意されているので、それを使って例外を分類できます。たとえば、

    Exception *pEx = GET_EXCEPTION();
    if (pEx->IsType(CLRException::GetType())) {/* ... */}

これで、例外がCLRException（またはその派生型）かどうかを判定できるでしょう。

EX_END_CATCH(RethrowTransientExceptions)
----------------------------------------

上記の例では、"RethrowTransientExceptions"がEX_END_CATCHマクロの引数です。これは「例外処理」を表すと考えられる3つの定義済みマクロのうちのひとつです。それぞれのマクロとその意味は以下の通りです。

- _SwallowAllExceptions_: これは適切な名前がつけられていますし、とても単純です。名前が示す通り、これはすべてを丸飲み（swallow）します。単純で魅力的ではありますが、これをやってはいけないこともしばしばです。
- _RethrowTerminalExceptions_: より良い名前は"RethrowThreadAbort"だったのでしょう。このマクロがやることはそちらなのですから。
- _RethrowTransientExceptions_: 「一過性」の例外について最も良い定義は、もう一度試したとしても、おそらくコンテキストが異なっていて発生しないかもしれないものです。一過性の例外は以下の通りです。
  - COR_E_THREADABORTED
  - COR_E_THREADINTERRUPTED
  - COR_E_THREADSTOP
  - COR_E_APPDOMAINUNLOADED
  - E_OUTOFMEMORY
  - HRESULT_FROM_WIN32(ERROR_COMMITMENT_LIMIT)
  - HRESULT_FROM_WIN32(ERROR_NOT_ENOUGH_MEMORY)
  - (HRESULT)STATUS_NO_MEMORY
  - COR_E_STACKOVERFLOW
  - MSEE_E_ASSEMBLYLOADINPROGRESS

どのマクロを使うか迷っているCLR開発者は、おそらく_RethrowTransientExceptions_を選ぶべきでしょう。

とはいえどんな場合においても、EX_END_CATCHを書く開発者はどの例外をキャッチするべきかよく考える必要がありますし、そしてそれらの例外だけをキャッチするべきです。また、いずれにせよこれらのマクロはすべてをキャッチするのですから、例外をキャッチしないようにする唯一の方法は、その例外を再度投げることだけです。

ある EX_CATCH / EX_END_CATCH ブロックが例外を適切に分類できていて、必要な場所では投げ直しているならば、これ以上の投げ直しは不要だとマクロに伝える方法はSwallowAllExceptionsです。

## EX_CATCH_HRESULT

ある例外に対応するHRESULTしか必要でないことも時々あります。コードがCOMに対するインターフェースの場合などは特にそうです。そのような場合は、EX_CATCHブロックを書くよりもEX_CATCH_HRESULTの方が単純です。典型的なケースは以下のような形になります。

    HRESULT hr;
    EX_TRY
      // code
    EX_CATCH_HRESULT (hr)

    return hr;

__しかし、これは非常に魅力的にもかかわらず、常に正しいわけではありません。__ EX_CATCH_HRESULT はすべての例外をキャッチして、HRESULTを保存して、そしてキャッチした例外を握りつぶします。したがって、その関数で本当に必要なことが例外を握りつぶすことではないのなら、EX_CATCH_HRESULTは適切ではありません。

EX_RETHROW 
----------

上で説明したように、例外マクロはすべての例外をキャッチします。ある特定の例外をキャッチするには、全てをキャッチした後で、興味のある例外以外は投げ直すしか方法がありません。したがって、ある例外がキャッチされ、調査され、あるいはログ出力などがされた後で、キャッチされるべきでない例外は投げ直されるかもしれません。EX_RETHROWは同じ例外をもう一度起こします。

例外をキャッチしない
=========================

あるコードが、例外をキャッチする必要はないけれども、何らかのクリーンアップや補償動作を実行する必要があるというのはよくあることです。このシナリオにホルダーがぴったりはまることは多いのですが、常にそうとは限りません。ホルダーでは十分でない場合のために、CLRは"finally"ブロックのバリエーションを2つ備えています。

EX_TRY_FOR_FINALLY
------------------

コードを抜けるときに補償処理のようなことをする必要がある場合は、finallyが適切でしょう。CLRにはtry/finallyを実装するマクロの組があります。

    EX_TRY_FOR_FINALLY
      // code
    EX_FINALLY
      // exit and/or backout code
    EX_END_FINALLY

**重要** : EX_TRY_FOR_FINALLYマクロはC++例外処理ではなくSEHを使って組まれています。そして、C++コンパイラーはSEHとC++例外処理とを同じ関数内で混ぜることを許していません。自動デストラクターを持ったローカル変数は、デストラクターを実行するためにC++例外処理を要求します。したがって、EX_TRY_FOR_FINALLYを使っている関数では、EX_TRYを使うことはできませんし、自動デストラクターを持ったローカル変数を持つこともできません。

EX_HOOK
-------

よくある話ですが、補償処理のコードが必要だけれども、それは例外が投げられた時だけでいいという場合があります。そのような場合のためにEX_HOOKがあります。EX_FINALLYに似ていますが、「フック」節は例外が起こったときしか実行されません。起こった例外は「フック」節の末尾で自動で気に投げ直されます。

    EX_TRY
      // code
    EX_HOOK
      // code to run when an exception escapes the “code” block.
    EX_END_HOOK

この構成にはEX_CATCHとEX_RETHROWを単に組み合わせたものよりちょっといい点があります。なぜかと言えば、スタックオーバーフロー以外の例外は投げ直されますが、スタックオーバーフローの場合はキャッチして（スタックを巻き戻し）、それから新しいスタックオーバーフロー例外を投げるからです。

例外を投げる
=====================

CLR内部で例外を投げるというのは、一般的に言って次の関数を呼び出す話です。

    COMPlusThrow ( < args > )

これには多数のオーバーロードがありますが、共通する考え方は、例外の「種別」をCOMPlusThrowに渡すということです。「種別」の一覧は一連のマクロで[Rexcep.h](https://github.com/dotnet/coreclr/blob/master/src/vm/rexcep.h)上に生成されており、そのさまざまな「種別」とは、kAmbiguousMatchExceptionや、kApplicationExceptionといったものです。（オーバーロードの）追加の引数はリソースや代理テキストを指定するものです。正しい「種別」を選択するには、だいたいにおいて同様のエラーを報告する別のコードを見つければよいでしょう。

便利なバリエーションがいくつか定義済みです。

COMPlusThrowOOM();
------------------

ThrowOutOfMemory()関数にしたがって、C++のメモリ不足（OOM）例外を投げます。この関数はあらかじめ確保された例外を投げます。メモリ不足例外を投げようとしてメモリが不足してしまうのを避けるためです！

この例外からマネージド例外オブジェクトを得る際には、ランタイムはまず新しいマネージドオブジェクトを割り当てようとします<sup>[1]</sup>。そしてそれに失敗した場合は、あらかじめ確保されて共有されているグローバルなメモリー不足例外オブジェクトを返します。

[1] というのも、失敗したのが2GBの配列の要求だった場合は、ささいなオブジェクトなら問題ないかもしれませんから。

COMPlusThrowHR(HRESULT theBadHR);
---------------------------------

これには多数のオーバーロードがあり、IErrorInfoなどを持つ場合に使用します。あるHRESULTに対応するのがどの種類の例外なのかを判別するコードがありますが、驚くほど入り組んでいます。

COMPlusThrowWin32(); / COMPlusThrowWin32(hr); 
---------------------------------------------

基本的には、HRESULT_FROM_WIN32(GetLastError())で例外を投げます。

COMPlusThrowSO(); 
-----------------

スタックオーバーフロー（SO）例外を投げます。気を付けてほしいことは、この例外は確実なSOではなく、このまま行けば確実なSOを引き起こすと思われる場合に投げる例外だということです。

この関数はOOMと同様に、あらかじめ確保されたC++のSO例外オブジェクトを投げます。OOMと異なるのは、マネージドオブジェクトを取得する際に、ランタイムはあらかじめ確保されて共有されているグローバルなスタックオーバーフロー例外オブジェクトを常に返すことです。

COMPlusThrowArgumentNull() 
--------------------------

「値を Null にすることはできません。パラメーター名:foo」という例外を投げるヘルパー関数です。

COMPlusThrowArgumentOutOfRange() 
--------------------------------

同様に、名前の通りです。

COMPlusThrowArgumentException() 
-------------------------------

さらに別種の、不正引数例外です。

COMPlusThrowInvalidCastException(thFrom, thTo)
----------------------------------------------

キャストに失敗した時、このヘルパー関数はキャスト前とキャスト後の型ハンドルを受け取って、良い感じにフォーマットされた例外メッセージを生成します。

EX_THROW
--------

これは低レベルなthrowを構成するもので、通常のコードでは普通は必要になりません。COMPlusThrowXXX関数の多くがEX_THROWを内部的に使用しており、特化した形のThrowXXX関数もそうです。推奨したいのは、EX_THROWを直接使うのは最小限にして、例外機構の根本的な詳細はできる限りカプセル化しておくことです。しかし、高レベルなThrow関数でうまく動作するものがなければ、EX_THROWを使っても構いません。

このマクロは2つの引数を取ります。投げたい例外の型（C++のExceptionクラスから派生する何らかの型）と、例外型のコンストラクター引数になる、かっこでくくられたリストです。

SEHを直接使う
==================

ごく限られた状況に置いては、SEHを直接使うことが適切な場合もあります。特に、SEHが唯一の選択肢になるのは、初回パス、つまりスタックが巻き戻る前に、何らかの処理が必要となる場合です。SEHの __try/__except 内のフィルターコードは、例外を処理する方法を決定するのに加えて、どんなことでもできます。デバッガーへの通知は、時に初回パスでの処理が必要となる様な分野です。

フィルターコードはよく注意して書かれなければなりません。総じて言えば、フィルターコードはどんなに乱雑で、そしておそらく一貫性のない状態でも想定しておかなければなりません。フィルターは初回パスで動作し、デストラクターは二回目のパスで動作するため、ホルダーはまだ動作しておらず、したがって状態はまだ復元されていないのです。

PAL_TRY / PAL_EXCEPT, PAL_EXCEPT_FILTER, PAL_FINALLY / PAL_ENDTRY
-----------------------------------------------------------------

フィルターが必要な時は、PAL_TRYの仲間を使うのがCLR内でフィルターを書くポータブルな方法です。フィルターはSEHを直接使いますから、同じ関数内ではC++例外処理と組み合わせられませんし、したがって関数内にはホルダーがあってはいけません。

繰り返しますが、これらはめったに使うものではありません。

__try / __except, __finally
---------------------------

これらのマクロをCLR内で直接使う正当な理由はありません。

例外とGCモード
======================

COMPlusThrowXXX()で例外を投げることはGCモードに影響を与えませんし、どのモードであっても安全です。例外によってEX_CATCHまで進められると、スタック上にあったすべてのホルダーは巻き戻され、リソースを解放して状態をリセットします。EX_CATCH内で実行が再開する時までには、ホルダーが保護している状態はEX_TRYの時点まで復元されているでしょう。

相互遷移
===========

マネージドコードやCLRやCOMサーバーやそのほかのネイティブコードについては、呼び出し規約やメモリー管理や、そしてもちろん例外処理機構にも、多くの相互遷移がありえます。例外について言えば、CLR開発者にとっては幸運なことに、それらの遷移のほとんどは完全にランタイムの外部で起こるか、または自動的に処理されるかのどちらかです。CLR開発者が常々気にしなければならない遷移は3つあります。その3つ以外については高度な話題になりますし、その話題について知っておかなければならない人たちというのは、自分たちがそれを知るべきだとよく分かっています！

マネージドコードからランタイムへの遷移
-----------------------------

これは「fcall」や「JITヘルパー」といったもののことです。ランタイムがエラーの報告をマネージドコードに戻す典型的な方法はマネージド例外を使うものです。よって、fcall関数が、直接的あるいは間接的に、マネージド例外を起こす場合は、事態は完璧にうまくいきます。CLRマネージド例外の普通の実装は「正しいことをする」であり、適切なマネージドハンドラーを探します。

一方で、fcall関数が、CLRの内部例外（C++例外のどれか）を投げるような何かをやれてしまう場合は、その例外をマネージドコードに漏らすことは許されていません。この場合に対処するために、CLRにはUnwindAndContinueHandler（UACH）というものがあります。これは、C++例外処理の例外をキャッチして、マネージド例外として再び起こす一連のコードです。

ランタイムの関数で、マネージドコードから呼ばれる可能性があり、そしてC++例外処理の例外を投げる可能性があるものはどれも、例外を投げるコードをINSTALL_UNWIND_AND_CONTINUE_HANDLER / UNINSTALL_UNWIND_AND_CONTINUE_HANDLERでラップしなければなりません。HELPER_METHOD_FRAMEを設置することで、自動的にUACHが設置されます。UACHの設置には無視できないほどのオーバーヘッドがありますから、どこでも使うようなことはするべきではありません。性能上クリティカルなコードで使う場合の1つのテクニックは、UACHを使わないでコードを実行し、例外を投げる直前に設置するというものです。

C++の例外が投げられた際にUACHが見つからない場合の典型的な障害は、CPFH_RealFirstPassHandlerにおける「GC_NOTRIGGERの領域でGC_TRIGGERSが呼び出されました」という契約違反エラーでしょう。修正するには、マネージドからランタイムへの遷移を探して、INSTALL_UNWIND_AND_CONTINUE_HANDLERまたはHELPER_METHOD_FRAME_BEGIN_XXXをチェックします。

ランタイムコードからマネージドコードへの遷移
------------------------------

ランタイムからマネージドコードの呼び出しにはプラットフォームに強く依存する要求があります。32ビットWindowsプラットフォームでは、CLRのマネージド例外コードはマネージドコードに入る直前に"COMPlusFrameHandler"が設置されることを要求します。これらの遷移を処理するのは高度に特殊化したヘルパー関数で、そのヘルパー関数は適切な例外ハンドラーの面倒を見ます。マネージドコードの典型的な新しい呼び出しが何か別の入口を通るようなことは非常に考えにくいことです。COMPlusFrameHanderが見つからなかったというイベントの中で、一番あり得る結果は、ターゲットのマネージドコード内の例外処理コードが単純に実行されない―ーどのfinallyブロックも、どのcatchブロックもです―ーというものでしょう。

ランタイムコードから外部ネイティブコードへの遷移
--------------------------------------

ランタイムから他のネイティブコード（OSやCRTやその他のDLL）の呼び出しには特別な注意が必要となるかもしれません。問題になりうるケースは、外部コードが例外を引き起こすかもしれない場合です。そのことが問題になるのはEX_TRYマクロの実装、特に、このマクロが例外でないものを例外に変換したりラップしたりする方法に原因があります。C++例外処理では、どんな例外でもすべて（"catch(...)"によって）キャッチすることができますが、それはキャッチしたものについての情報をすべて取得することをあきらめた場合だけなのです。何らかのException* をキャッチする際には、当該マクロは調査すべき例外オブジェクトを持っていますが、例外でない何かをキャッチする際には、調査する対象が存在しないため、マクロは実際の例外が何なのかを推測しなければなりません。そしてその例外がランタイムの外から来ているものだった場合は、マクロは必ず誤った推測をするでしょう。

現時点での解決策は、外部コードの呼び出しを「呼び出しフィルター」でラップするというものです。このフィルターは外部例外をキャッチした時に、それをSEHExceptionというランタイムの内部例外に変換します。このフィルターは定義済みで、簡単に使うことができます。しかし、フィルターを使うことはSEHを使うことになりますから、同じ関数内でC++例外処理を使うことはもちろん不可能になります。C++例外処理を使用している関数に呼び出しフィルターを追加する場合は、関数を2つに分割する必要があります。

呼び出しフィルターを使うには、以下のようにする代わりに

    length = SysStringLen(pBSTR);

以下のように書きます。

    BOOL OneShot = TRUE;

    PAL_TRY
    {
      length = SysStringLen(pBSTR);
    }
    PAL_EXCEPT_FILTER(CallOutFilter, &OneShot)
    {
      _ASSERTE(!"CallOutFilter returned EXECUTE_HANDLER.");
    }
    PAL_ENDTRY;

呼び出しフィルターを掛けないコード呼び出しで例外が起きた場合は間違いなく、ランタイムには誤った例外が報告されるでしょう。誤って報告される例外の型がいつも一意に決まるとも限りません。というのも、何らかのマネージド例外が既に「飛行中」だった場合は、そのマネージド例外が報告されることになります。もしその時点で例外がなければ、OOMが報告されることになります。チェック済みビルドにおいては、通常は呼び出しフィルターが見つからなければ発火するアサーションがいくつかあります。それらのアサートメッセージには「ランタイムは例外の型が追跡できなくなったかもしれません」というテキストが含まれるでしょう。

その他いろいろ
=============

実際には、EX_TRYの内側には多くのマクロが関係しています。それらのほとんどは、マクロ実装の外部では絶対に、決して使うべきではありません。

BEGIN_EXCEPTION_GLUE / END_EXCEPTION_GLUEの組については特別に言及しておいた方がよいでしょう。これらは遷移マクロになる予定だったのですが、Whidbeyプロジェクトにおいて、もっと適切なマクロで置き換えられることになりました。もちろん、このマクロの組はきちんと動作しましたし、したがってすべてが置き換えられてしまったわけではありませんでした。うまくいけば、このマクロを使用している個所はすべて「クリーンアップ」のマイルストーン期間中に変換され、このマクロは取り去られてしまうことになるでしょう。それまでにこのマクロを使いたくなってしまったCLR開発者は、ぐっと我慢して、EX_TRY/EX_CATCH/EX_CATCH_ENDやEX_CATCH_HRESULTを使ってコードを書きましょう。