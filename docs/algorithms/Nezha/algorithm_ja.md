# Nezha

## Nezhaとは

https://github.com/nezha-dt/nezha

NezhaはLLVMのlibFuzzerを拡張して作られたdifferential fuzzingの実装で、greybox fuzzingでもblackbox fuzzingでも用いる事ができる(とされているが、上記の実装ではgreybox fuzzingのみが実装されているように見える)。

NezhaはlibFuzzerの一部を書き換えて実装されているため、アルゴリズムの殆どの部分はlibFuzzerと同じである。ただし、1つの入力値使って、複数の異なる実装で同じ処理をするターゲットを実行し、一部のターゲットだけが他と異なる出力になるような入力を「一部の実装の不具合を突く入力」と見做す点がlibFuzzerと異なる。

入力値の選択やmutationはlibFuzzerの実装がそのまま用いられる。

ターゲットを1つ実行する毎にfeatureを計算してcorpusの追加までを行う。

ターゲット毎にカバレッジのedgeのindexが重複しない値になるようにすることで(オリジナルの実装では全てのターゲットを単一のバイナリにリンクするため勝手にそうなる)異なるターゲットの実行結果同士が同一と見做される事を防ぐ。

つまりcorpusには全く同じ入力値で実行しているにもかかわらず結果が異なる実行結果が最大でターゲットの数だけ登録される。

これらのcorpusの要素に対して個別に重みが与えられるため、ある入力値が実際に選ばれる確率はcorpusに登録されている同じ入力値の実行結果の重みの総和になる。

全てのターゲットを実行した後でcorpusへの追加状況と各ターゲットからの出力を見て実行結果を「一部の実装の不具合を突く入力」と見做すかどうかを判断する。

## NezhaがlibFuzzerから分岐した位置について

githubのNezhaのリポジトリではlibFuzzerのコードの追加および新しいバージョンへの追従とNezhaの実装が混ざって1つのcommitになっているため、NezhaがLLVMのどの時点からforkしたのかは明らかではない。

ただし、様々な時点のlibFuzzerのソースとNezhaのソースでdiffをとって差分の量を比較した結果、現在のNezhaのmasterはLLVM 4.xブランチがLLVMのmasterから分かれた後、LLVM 5.0.0-rc1が作られるより前のmasterのどこかからforkしたとすると最も差分が小さくなる。

NezhaのmasterとLLVM 5.0.0-rc1時点のlibFuzzerのdiffをとった物は[こんな内容](/docs/algorithms/Nezha/nezha-5.0.0-rc1.diff)になる。

## fuzzufにおける実装

[移植の状況](/docs/algorithms/Nezha/porting_status_ja.md)

## Nezha固有のノード

fuzzufの実装では以下のノードをlibFuzzerとは別に実装している。

### collect\_features

libFuzzerのcollect\_featuresではfeatureが初めて発見された物であるか、あるいは過去にこのfeatureを発見した入力より短い入力でfeatureを発見した場合に限り、実行結果のunique\_feature\_setにfeatureを追加する。

一方Nezhaのcollect\_featuresでは発見した全てのfeatureをunique\_feature\_setに追加する。

同じfeatureを持った実行結果が出来やすくなるため、NezhaではlibFuzzerより頻繁にcorpusの要素のreplaceが起こるようになる。

### add\_to\_solutions

入力値を複数のターゲットで実行して得た複数の実行結果から

* それぞれの実行結果がcorpusに追加されたかどうかをvectorにしたもの
* それぞれの出力をvectorにしたもの

を引数に取り、それらのうちいずれかが過去にadd\_to\_solutionsを呼び出した際に渡された事のない値の場合かつ、ターゲット毎に実行結果が食い違っている場合にその実行結果と入力値をsolutionsに追加する。

### gather\_trace

引数で渡した実行結果がcorpusに追加されていたかどうかを、引数で渡したvectorに追加する

### gather\_status

引数で渡した実行結果に含まれる終了理由を、引数で渡したvectorに追加する

### gather\_output

引数で渡した標準出力のハッシュ値を、引数で渡したvectorに追加する

## 標準的なNezhaの組み方

test/algorithms/libfuzzer/execute.cpp の中で組み立てているものがオリジナルのNezhaのFuzzer::Loop()に近い処理になっている

test/algorithms/libfuzzer/output\_hash.cpp の中で組み立てているものはexecute.cppと殆ど同じ動きをするが、ターゲットの出力として終了理由ではなく標準出力のハッシュ値を用いる
