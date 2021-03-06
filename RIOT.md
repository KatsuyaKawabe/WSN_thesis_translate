# RIOT OS: Towards an OS for the Internet of Things
Emmanuel Baccelli and Oliver Hahm
INRIA, France
Mesut Gunes and Matthias W ¨ ahlisch ¨
Freie Universitat Berlin, Germany ¨
Thomas C. Schmidt
HAW Hamburg, Germany

## Abstract

Internet of Things（IoT）は、異質のデバイスによって特徴付けられます。
8ビットマイクロコントローラ（MCU）を搭載した非常に軽いセンサから、より強力でエネルギー効率の高い32ビットプロセッサを搭載したデバイスまでさまざまです。
インターネットホスト上で現在動作している従来のオペレーティングシステム（OS）も、センサネットワーク用の典型的なOSも、このような広範なデバイスの多様な要件を満たすことはできません。
IoTを活用するには、冗長な開発を避け、メンテナンスコストを削減する必要があります。
本稿では、IoTにおけるOSの要件を再検討する。
最小限のリソースでデバイスを明示的に検討するが、幅広いデバイスでの開発を容易にするOSであるRIOT OSを紹介します。
RIOT OSは、標準のCおよびC ++プログラミングを可能にし、マルチスレッドおよびリアルタイム機能を提供し、最小1.5kBのRAMしか必要としません。

## 1. INTRODUCTION

センサノード、家電、スマートフォン、車両などの数十億もの異機種デバイスは、自発的な無線ネットワークや電力線通信によって相互接続され、IoTを生み出すことが期待されています[1]。 このようなデバイスは、コンピューティングパワー、利用可能なメモリ、通信、およびエネルギー容量の点で極端に制約されてはいるものの、（i）信頼性、（ii）リアルタイム動作、および（iii） インターネットをシームレスに統合するための適応型通信スタックです。

IoTデバイス用OSは、これらの要件を満たし、低消費電力MCUベースのノードから、新しい世代の高効率32ビット・プロセッサを搭載したノードまで、幅広いハードウェアで動作する必要があります。 既存のOSのいずれも、Wireles Sensor Networks（WSN）や本格的なOSをターゲットとした軽量OSのいずれも、多様な要件を満たすことはできません。 したがって、冗長な開発とメンテナンスのコストを避けるために、このポスターの主題である新しい統一タイプのOSが必要です。

### 1.1 Problem Statement

理想的には、本格的なOSの機能は、すべてのIoTデバイスで使用できるようにする必要があります。
以下は、主要な本格的なOSの中でオープンソースOSの良い例であるため、Linuxの特徴に焦点を当てています。 開発者が標準のCまたはC ++でコーディングできるという意味で、利用可能な多数のシステムライブラリ、ネットワークプロトコルまたはアルゴリズム、およびゼロに近い学習曲線が、開発者にとって使いやすいものになっています。 しかし、LinuxのCPUとメモリに関する最小限の要件は、小型MCUによって駆動される制約の厳しいIoTデバイスには適合しません。 これらの要求を押し進めようと努力してきたが[2]、LinuxはIoT用に設計されておらず、厳格なエネルギー効率を達成できないと主張する。 これらの理由から、LinuxはIoTですべてを統治する1つのOSになるとは考えられません。

他方、最も制約の厳しいIoTデバイス上で動作するWSNをターゲットとする典型的な軽量OSが可能にするトレードオフは、開発者にはあまりにもやさしく、より強力なIoTデバイスでの使用は、エネルギー効率の低い実装となる デバイスの完全な機能を活用します。
支配的なWSN OS、ContikiおよびTinyOS [3]は、典型的なWSNシナリオには便利なイベントドリブンデザインに従いますが、効率的で機能的なネットワーキング実装の欠点を示します。
たとえば、典型的なWSNシナリオでは、ループ内でタスクを順番に処理するだけで十分ですが、そのようにすると、制限されたメモリのために最大で1つのTCP接続にウィンドウサイズが制限されます。 ネイティブのマルチスレッディング、ハードウェア抽象化、動的メモリ管理など、最新の本格的なOSの機能を提供する代替システムアーキテクチャが望ましい。 しかし、これらのタイプのシステムは、これまでIoTデバイスにとっては複雑すぎると考えられてきた。

## 2. RIOT: AN OS FOR SMALLER THINGS IN THE INTERNET

### 2.1 Design Aspects for an IoT OS

基本的に、OSの特徴は、カーネルの構造、スケジューラ、プログラミングモデルです。
カーネルは、（i）モノリシックな方法で構築すること、（ii）階層化アプローチに従うこと、または（iii）マイクロカーネルアーキテクチャを実装することができる。
スケジューリング戦略の選択は、リアルタイムサポート（またはその欠如）、異なるタスク優先順位のサポート、またはサポートされているユーザーインタラクションの程度に密接に結びついています。
最後に、プログラミングモデルは、（i）すべてのタスクが同じコンテキスト内で実行され、メモリアドレス空間のセグメンテーションを持たないか、または（ii）すべてのプロセスがそれ自身のスレッドで実行され、独自のメモリスタックを持つかを定義します。
プログラミングモデルは、アプリケーション開発者が利用できるプログラミング言語にもリンクしています。

### 2.2 Comparing Existing Solutions:

これらの観察に基づいて、Contiki、Tiny OS、Linuxを比較し、IoT用のOSの第1の原則を導き出します。
Contikiは階層化されたアプローチに近いモジュラーコンセプトに従いますが、Tiny OSはLinuxのようなモノリシックカーネルで構成されています。
Contikiのスケジューリングは、FIFO戦略が使用されるTinyOSの場合と同様に、純粋にイベント駆動型です。一方、Linuxではスケジューラが使用されているため、処理時間の公平な配布が保証されています。
ContikiとTinyOSのプログラミングモデルは、部分的なマルチスレッドのサポートを提供していますが、すべてのタスクが同じコンテキスト内で実行されるようなイベント駆動モデルに基づいています。
ContikiはC言語のサブセットを使用していますが、一部のキーワードは使用できませんが、TinyOSはnesCというC言語で書かれています。
Linuxは、一方で、実際のマルチスレッドをサポートし、標準C言語で書かれており、さまざまなプログラミング言語やスクリプト言語をサポートしています。
これらの設計上のトレードオフのため、TinyOSとContikiには、標準的なC / C ++プログラミング容易性、標準的なマルチスレッド、リアルタイムサポート（表1を参照）など、いくつかの開発者にとって重要な機能が欠けています。
大規模なIoTデバイスの展開を成功させるOSは、これらの機能をサポートする必要があります。開発者の視点から見ると、ハードウェアに依存しない強力なAPIです。

### 2.3 RIOT Architecture Overview

RIOTアーキテクチャの概要：RIOT OSは、WSN用OSと現在インターネットホスト上で動作している従来の本格的OSとのギャップを埋めることを目指しています。
これは、基本的なハードウェアとは独立した、エネルギー効率、小さなメモリフットプリント、モジュール性、統一されたAPIアクセスなどの設計目標に基づいています。
RIOTはFireKernel [4]から継承されたマイクロカーネルアーキテクチャを実装しており、標準APIでマルチスレッドをサポートしています。
FireKernel独自の機能に加えて、RIOTはWiselibアルゴリズムフレームワークなどの強力なライブラリを可能にするC ++のサポートを追加し、TCP / IPネットワークスタックを提供します。
RIOTアーキテクチャの利点は、（i）高い信頼性、（ii）開発者に優しいAPIです。
RIOTのモジュール式マイクロカーネル構造は、単一コンポーネントのバグに対してロバストになります。
たとえば、デバイスドライバやファイルシステムの障害は、システム全体を害することはありません。
RIOTは、開発者が必要な数のスレッドを作成できるようにし、カーネルメッセージAPIを使用して分散システムを簡単に実装することができます。
スレッドの量は、各スレッドの使用可能なメモリとスタックサイズによってのみ制限されますが、計算およびメモリのオーバーヘッドは最小限に抑えられます。

### 2.4 RIOT Technical Details

強いリアルタイム要件を満たすために、RIOTは、カーネルタスク（例えば、スケジューラ実行、プロセス間通信、タイマー操作）のための一定期間を強制する。
O（1）の保証されたランタイムの重要な前提条件は、カーネル内の静的メモリ割り当ての排他的使用です。
それでも、アプリケーションには動的メモリ管理が提供されます。
スレッドの固定サイズ循環リンクリストを使用して、スケジューラの実行時間を一定にします。
タイマ動作の一定の実行時間は、MCUが通常複数の比較レジスタを提供するという事実を利用することによって得られる。

より強力なIoTデバイスに対してもエネルギー効率を実現するために、ディープスリープモードで費やされる時間を最大限にすることが必須であるため、RIOTは定期的なイベントなしで動作するスケジューラを導入しています。
保留中のタスクがない場合は、RIOTは使用中の周辺デバイスに応じて可能な限り最も深いスリープモードを決定するアイドルスレッドに切り替えます。
割込み（外部またはカーネル生成）のみがシステムをアイドル状態から起動します。

カーネル機能の複雑さが低いことは、OSのエネルギー効率の主な要因です。
したがって、コンテキスト切り替えの持続時間および発生を最小限に抑えなければならない。
RIOTコンテキスト切り替えは、（i）対応するカーネル操作そのもの、例えばミューテックスロックまたは新しいスレッドの作成、または（ii）割り込みがスレッドスイッチを引き起こすという2つのケースで行われる。
最初のケースはほとんど起こりません。
たとえば、すべてのスレッドは通常1回作成されます。 したがって、スレッド切り替えの場合の処理時間を短縮することが重要です。
したがって、RIOTのカーネルは割り込みサービス・ルーチンから呼び出されると、スケジューラーの最小化を実現します。
その場合、古いスレッドのコンテキストを保存する必要はないので、タスクスイッチは非常に少ないクロックサイクルで実行できます。

## 3. AVAILANLE CODE & FUTURE WORK

この洗練されたアーキテクチャと効率的なスケジューリングにもかかわらず、RIOTのメモリフットプリントは低いです。
利用可能なオープンソースのRIOTコード[5]は、例えば、MSP430上の基本的なアプリケーションのために、5kByte未満のROMと1.5kByte未満のRAMを必要とします。
RIOTは、16ビットマイクロコントローラから32ビットプロセッサまで、標準のANSI Cコードと共通のPOSIXライクなAPIと組み合わせてマルチスレッドプログラミングモデルを使用しています。 したがって、異機種のIoTハードウェアを使用するプロジェクトでは、RIOTでソフトウェアシステム全体を構築し、既存のライブラリを簡単に採用することができます。
さらに、制約付きシステムをインターネット（6LoWPAN、RPL）に接続するためのIETFの最新規格を含むいくつかのネットワーキングプロトコルが利用可能になり、RIOT IoT対応になります。 進行中の作業には、完全なPOSIX準拠と様々なIoTプラットフォームへの移植が含まれます。

