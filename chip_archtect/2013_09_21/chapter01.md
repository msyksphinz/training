AMDの64ビットコア詳細アーキテクチャの理解
=========================================

マルチプロセッサ環境において、ワールドクラスの投機アウトオブオーダ64ビットプロセッサを構築するために何が起きているのかを真に知りたい人向け

# 第1章: 整数コア: 整数スーパハイウェイ

- 1.1 整数スーパーハイウェイ
- 1.2 3-wayスーパスカラCISCアーキテクチャ
- 1.3 命令の3種の階層: ダブルディスパッチ操作
- 1.4 128bit SSE(2)命令はDoublesに分解される
- 1.5 128bit SSE(2)命令にDoubleを使うことにより25%のレイテンシペナルティの発生を避けることができる
- 1.6 Doublesはいくつかの整数命令とx87命令を同時に利用する
- 1.7 Doubleは128bitのメモリアクセスを処理できる
- 1.8 スケジューリング前のアドレス加算
- 1.9 レジスタリネーミングとアウトオブオーダ処理
- 1.10 整数レジスタのリネーミング
- 1.11 IFFRF: Integer Future File and Register File
- 1.12 IFFRFの"Future File"セクション
- 1.13 例外および分岐予測ミス: リタイア値で投機実行結果を上書きする
- 1.14 リオーダバッファ
- 1.15 リタイア処理と例外処理
- 1.16 例外処理は、リタイア処理まで常に遅延される
- 1.17 ベクトルパスとダブルディスパッチ命令のリタイア処理
- 1.18 アウトオブオーダ処理: 命令ディスパッチ
- 1.19 スケジューラ / リザベーションステーション
- 1.20 各x86命令はALUとAGU操作を起動することができる
- 1.21 ALU操作のスケジューリング
- 1.22 メモリアクセスのためのAGU操作のスケジューリング
- 1.23 Opteronの整数コアのマイクロアーキテクチャの利点

# 第1章: 整数コア: 整数スーパハイウェイ

## 1.1 整数スーパーハイウェイ

整数コアのダイ写真を見ると、殆どの領域が64bitのデータバスであることが分かる。
データバスは北から南にかけて張られている。
いくつかの領域では、最大で20種類のバスが集積されている。
バスは、整数ユニットで利用される全てのソースオペランドと出力オペランドを転送している。
バスのレイアウトは*ビットインタリーブ*の形式で設計されており、各ビットはビットインデックスでグループ化されている。
全てのバスのビット0は集められ、隣同士に配置されている。
一方で、ビット63はもう片方の領域にまとめられている。
各バイトで分割されているのは、レイアウトからも分かりやすく確認できる。

## 1.2 3-wayスーパスカラCISCアーキテクチャ

Opteronは3-wayスーパスカラプロセッサである。
このプロセッサは1サイクルあたり3つのx86命令をデコード、実行、リタイアすることができる。
これらの実行される命令は非常に複雑な命令(CISC)であり、複数(2以上)のソースオペランドを持っている。
Pentium 4はサイクルあたり3つのuOpsと呼ばれる操作を実行することができる。
単一のx86命令を実行するためには、複数のuOpsを実行する必要がある。
私達の調査により、Prescottではサイクルあたり最大で4つのuOpsを実行することができることが分かっている。

一般的に、x86命令は F(reg, reg)、F(reg, mem)、F(mem, reg)として表現される。
最初のオペランドは、ソースと出力の両方を取ることができる。
最初の2つの形式は整数命令、MMX、SSE(2)において標準的な形式である。
最後の形式は基本的に整数命令において見られる。
1つのソースオペランドがメモリから読み込まれ、結果は同じ領域に書き戻される。
整数パイプラインは、浮動小数点命令とマルチメディア命令を含む、全ての処理においてロードとストア処理を実行するのに利用される。

## 1.3 命令の3種の階層: ダブルディスパッチ操作

オリジナルのAthlon(Athlon 32とも呼ばれている)は命令をDirectPathとVectorPathに分類していた。
最初のクラスの命令は、複雑度の低い命令で、ハードウェアで単一の操作で実行できるような命令である。
より複雑な命令(VectorPath)では複数のマイクロシーケンサが起動され、マイクロコードプログラムが実行される。
命令はマイクロコードROMから読み込まれ、3-wayパイプラインに挿入される。

Opteronは3番目の命令クラスを導入した。
Double Dispatch命令、もしくは単に"Doubles"と呼ばれる。
Doublesはデコーディングパイプラインの最後の段で生成される。
DirectPathもしくはMicroCodeシーケンサにより生成された直後の命令は、2つの独立した命令に分割される。
3-wayパイプラインは、従って1サイクルあたり6つの命令を生成することになる。
これらの命令はPACKステージにより「再パッキング」され、3つの命令に戻される。
この追加されたパイプラインステージは、投機実行のために追加されたものであり、Opteronが2001年にMicro Processor Forumで紹介されたときからのものである。
6つの対照的な形をした"doubling stae"は、上記のダイ写真からもはっきりと確認できる。

## 1.4 128bit SSE(2)命令はDoublesに分解される

Opteronの最適化ガイドでは、全ての命令のどのクラスに分類されるかが記述されている。
ほとんどの128bitのSSE、SSE2命令はDouble Dispatch命令として実装されている。
独立した64bitの命令に分割することのできない命令のみ、VectorPath(マイクロコード)命令として処理される。
これらのSSE2命令で、128bitレジスタの半分しか使わない命令はSingle命令(Direct Path)命令として実装される。

Double Dispatch命令には、性能面においてトレードオフが存在する。
欠点は、128bit SSE2命令のデコード率がサイクルあたり1.5に制限される点である。
しかし一般的には、最大スループットは128bitSSE Single命令においてFPユニットとリタイアのハードウェアにに制限されるため、デコード率により性能の制約になることはない。
より重要なのはサイクルレイテンシが余分に必要な問題であり、Pentium4スタイルの実装により生じるレイテンシの問題を避けることができる。
