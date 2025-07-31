---
title: "【連載】Cybozu.comクラウド基盤の全貌 第5回 Neco のストレージ"
source: "https://blog.cybozu.io/entry/2025/07/18/113000"
author:
  - "[[anqou]]"
published: 2025-07-18
created: 2025-07-28
description: "はじめに こんにちは、クラウド基盤本部の伴野です。「【連載】Cybozu.comクラウド基盤の全貌」では、私たちが運用しているクラウド基盤を連載形式で紹介しています。今回の記事では、インフラ基盤 Neco のストレージについて説明します。Neco では様々なアプリケーションや、それらを支えるミドルウェアが動いています。この記事では、それぞれのソフトウェアの要件に合わせ、特性が異なる複数のストレージを Neco で提供していることを紹介します。 Kubernetes 上のストレージ Neco のストレージについて説明する前に、そもそも Kubernetes においてどのようにストレージがサポート…"
tags:
  - "clippings"
---
## はじめに

こんにちは、クラウド基盤本部の伴野です。 [「【連載】Cybozu.comクラウド基盤の全貌」](https://blog.cybozu.io/entry/2025/03/19/112000) では、私たちが運用しているクラウド基盤を連載形式で紹介しています。今回の記事では、インフラ基盤 Neco のストレージについて説明します。Neco では様々なアプリケーションや、それらを支えるミドルウェアが動いています。この記事では、それぞれのソフトウェアの要件に合わせ、特性が異なる複数のストレージを Neco で提供していることを紹介します。

## Kubernetes 上のストレージ

Neco のストレージについて説明する前に、そもそも Kubernetes においてどのようにストレージがサポートされているのかについて簡単に説明します。Kubernetes では、ファイルシステムやブロックストレージは主に PV（PersistentVolume）というリソースで表現されます。PV はクラスタ上にデプロイされた単一のストレージを表します。例えばノード上のディスクは一つの PV を構成し得ますし、LVM で切り出された LV（Logical Volume）も PV になり得ます。また NFS や iSCSI のようなネットワークストレージが提供するボリュームも PV として扱えます。

ユーザが Kubernetes 上で PV を使用したい場合、ユーザは PVC（PersistentVolumeClaim）というリソースを作成します。PVC には使用したいストレージの要件（サイズや種類など）が書かれており、Kubernetes によって適切な PV と紐付けられます。ユーザは、紐付けられた PV を PVC 経由で Pod に指定して使用します。

様々なストレージシステムを柔軟に扱うために、Kubernetes は CSI（Container Storage Interface）をサポートしています。CSI は、Kubernetes のようなコンテナ・オーケストレーション・システムが外部ストレージシステムと連携するための標準規格です。CSI driver と呼ばれるコンポーネントを実装することで、 好きなストレージを PV として Kubernetes 上で利用できるようになります。この後で見るように、Neco でも複数の CSI driver を活用しています。

![](https://cdn-ak.f.st-hatena.com/images/fotolife/a/anqou/20250613/20250613133331.png)

Kubernetes のストレージの仕組み

## Neco が提供するストレージ

Neco のノードには NVMe SSD と HDD が搭載されており、各ディスクは Linux の dm-crypt によって data at rest の暗号化が行われています（ [参考記事](https://blog.cybozu.io/entry/2019/03/08/170000) ）。Neco では、これらのディスクをソフトウェア的に組み合わせて Kubernetes 上のストレージとして提供しています。

Neco 上で動作するソフトウェアは様々で、それぞれストレージに求める要件が異なります。例えば、アプリケーションがファイルシステムに直接ファイルを読み書きする場合、そのボリュームの冗長化はストレージ側で行う必要があるでしょう。他方で、データベースなどのミドルウェアではデータの冗長性をミドルウェア自身が担保できることがあります。この場合、ストレージレイヤでの冗長化は必ずしも必要ではありませんが、代わりに IO に対する性能要件がシビアになりがちです。またその他のソフトウェアでは、データをオブジェクトストレージに保存することを要求するものもあります。

以上のような要件を満たせるように、Neco では特性の異なる複数種類のストレージを提供しています。まず、冗長性は不要な一方で高速な IO が必要な用途に向けて、ノード上の NVMe SSD を提供しています。これには自社で開発した [TopoLVM](https://github.com/topolvm/topolvm) という CSI driver を使用しています。TopoLVM によって払い出された PV は、ノード上のディスクに直接読み書きが行えるため、高速な IO が期待できます。一方で TopoLVM はノードに乗っているディスクをそのまま使用するため、ノードが故障した際には載っていたデータも一緒に使えなくなってしまいます。

そこで Neco では、より障害に強くスケーラブルなストレージとして、 [Ceph](https://ceph.io/en/) を使用したブロックストレージ（RBD：RADOS Block Device）と Amazon S3 API 互換なオブジェクトストレージ（RGW：RADOS GateWay）も提供しています。分散ストレージである Ceph の機能により、仮にノードやラックが故障した場合でもこれらのストレージは一貫性を保ったまま動作を続けられます。そのため、アプリケーションレベルでデータの冗長性を担保しないソフトウェアでも高可用な運用が可能になります。

ところで Ceph 自体は Kubernetes ネイティブなソフトウェアではありません。Ceph を Kubernetes 上で動作させるために、Neco では [Rook](https://rook.io/) を使用しています（この構成を以下では Rook/Ceph と呼びます）。Rook を使うことで Ceph を容易に管理できる上、Ceph の機能を Kubernetes に統合できます。

以下では TopoLVM と Rook/Ceph について、より詳細に説明します。

## TopoLVM

TopoLVM は Kubernetes 向けの CSI driver です。Cybozu が自社で開発し、OSS として GitHub で公開しています。過去に Cybozu Inside Out でも紹介したことがあります。

[blog.cybozu.io](https://blog.cybozu.io/entry/2019/11/08/090000)

また先日の KubeCon + CloudNativeCon Japan 2025 では、TopoLVM やその開発から生まれた KEP について、クラウド基盤本部の大神が発表しました：

![](https://www.youtube.com/watch?v=HXDIukPX6AY)[www.youtube.com](https://www.youtube.com/watch?v=HXDIukPX6AY)

TopoLVM の主な仕事は、サーバのローカルなボリュームを LVM で LV として切り出し、PV として提供することです。この PV をマウントした Pod は、切り出された LV に対して直接（ネットワークなどを介さず）ディスク IO を行うことが可能となります。そのため、高い IOPS と低い IO レイテンシを期待できます。

![](https://cdn-ak.f.st-hatena.com/images/fotolife/a/anqou/20250613/20250613133554.png)

TopoLVM の仕組み

TopoLVM は LVM を利用しているため、ローカルディスクだけでは不可能なことも可能になっています。例えば LVM を使うと複数のディスクを束ねて巨大な一つのボリューム（VG：Volume Group）として見せられます。TopoLVM では、そうやって束ねられたボリュームから PV を切り出すことができます。また、LVM が提供する LV のリサイズ機能を用いることで、払い出した PV の容量拡張にも TopoLVM は対応しています。

さらに TopoLVM は [dynamic provisioning](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/) ・ [topology-aware volume provisioning](https://kubernetes.io/blog/2018/10/11/topology-aware-volume-provisioning-in-kubernetes/) ・ [storage capacity tracking](https://kubernetes.io/docs/concepts/storage/storage-capacity/) に対応している上、オプショナルな機能として [scheduler extender](https://github.com/kubernetes/design-proposals-archive/blob/main/scheduling/scheduler_extender.md) も備えています。これらを使うことで、TopoLVM を使用する PVC が作成された際にオンデマンドに適切なノードを選択し、その上で動的に LV を切り出し PV を作成・紐付けられます。これにより柔軟なボリュームの払い出しが可能になっています。

一方で、TopoLVM が払い出すボリュームはノードローカルなものになります。つまり LVM によって作られた LV はそのノード上でのみ有効で、ノードやディスクが故障すると使えなくなってしまいます。そのため TopoLVM が払い出した PV をワークロードが使用する際には、ボリューム上のデータをワークロードが自らレプリケーションして冗長性を確保するか、あるいは消失しても良い一時ファイルなどを置くための使用に限る必要があります。

ノードローカルなストレージとしての別の弱みとして、使用できるボリュームサイズの上限がノードに搭載されたディスクの合計サイズで自動的に決まってしまうという問題もあります。例えば、ノードに搭載されたディスクの合計サイズが 1 TiB なら、そのノードで払い出せる最大の PV サイズは 1 TiB です。PVC で 10 TiB のストレージを要求され、仮に他のノードのディスクと合わせると 10 TiB 以上の空き容量があるとしても、そのような PVC に PV を払い出すことはできません。

以上のような特性を踏まえ、Neco では、NVMe SSD で構築した VG からボリュームを切り出すためにTopoLVM を使用しています。切り出されたボリュームは MySQL や Elasticsearch などのミドルウェアに使用されます。これらのミドルウェアはそれ自身がデータ冗長化のための仕組み（レプリケーションなど）を持っているため、ノード故障が発生して PV が使えなくなっても、他のノードへフェイルオーバーを行い動作を継続できます。

TopoLVM のその他の使い方として、消失しても問題ないようなボリュームが必要な場合に、TopoLVM から [generic ephemeral volume](https://kubernetes.io/docs/concepts/storage/ephemeral-volumes/#generic-ephemeral-volumes) を払い出しています。通常このような場合には `emptyDir` を用いると思いますが、 `emptyDir` に保存できるデータ量には上限があるため、その上限を超えてデータを保存したい場合に使用されています。

さて、上記の通り TopoLVM には幾つか欠点があるため、残念ながら全てのワークロードが TopoLVM で十分というわけではありません。ノードローカルでないボリュームが必要な場合、Neco では、以下で見る Rook/Ceph から PV を払い出すことになります。

## Rook/Ceph

Ceph はオープンソースの分散ストレージシステムです。Ceph の大きな特徴は、複数台のサーバをクラスタ化して使用することにより、スケーラブルなストレージプールを提供できるという点です。すなわち、Ceph が提供するストレージには仕様上のサイズの制限はなく、サーバを追加すればするだけ大きなストレージを扱えます。また Ceph は耐障害性を重視して設計されており、仮に一部の機材が故障したとしてもダウンタイムなくストレージを提供し続けられるようにデータが冗長化されています。従って、ユーザはデータの冗長性などを気にすることなく、高可用なストレージとして Ceph を使用できます。

![](https://cdn-ak.f.st-hatena.com/images/fotolife/a/anqou/20250613/20250613133651.png)

Ceph の仕組み

Ceph については、以前にも Inside Out で紹介したことがあります。

[blog.cybozu.io](https://blog.cybozu.io/entry/2018/12/13/103039)

Ceph を Kubernetes 上で管理・運用できるように開発されたのが Rook です。Rook も OSS であり、その開発には Cybozu も積極的に参加しています。特に、クラウド基盤本部のメンバである sat は Rook のメンテナでもあり、先日の KubeCon + CloudNativeCon Japan 2025 で登壇しました：

![](https://www.youtube.com/watch?v=LmL2wyMWTXo)[www.youtube.com](https://www.youtube.com/watch?v=LmL2wyMWTXo)

Rook は `CephCluster` といった CRD（Custom Resource Definition）と、そのリソースをリコンサイルする Kubernetes オペレータ（コントローラ）から主に構成されます。ユーザがカスタムリソースを apply すると、Rook がそれを見て必要な Pod などを自動的にデプロイし、Kubernetes 上で Ceph が使えるようになります。

さて Ceph のコアの部分は巨大なストレージプールを提供しており、ユーザはこれを様々な形で利用できます。具体的にはブロックストレージやファイルシステムとして使用できる RBD・CephFS の他、Amazon S3 API 互換のオブジェクトストレージである RGW などのインタフェースが提供されています。Rook もこれらの機能をサポートしており、それぞれに必要な CRD や Kubernetes とのインテグレーションが備わっています。Neco では Rook/Ceph が提供する機能のうち、主に RBD と RGW を使用しています。

### RBD

RBD は Ceph が提供する高可用なブロックストレージです。RBD 自体はブロックストレージですが、その上に一般的な（ext4 などの）ファイルシステムを構築することで、ファイルシステムとして使用できます。先に述べたように Ceph がデータの冗長化を担ってくれるため、RBD から払い出すボリュームについて利用者はデータの冗長化を気にする必要がありません。またボリュームのサイズに制限はなく、しかも RBD はシンプロビジョニングに対応しているため、ユーザは柔軟にボリュームを扱うことができます。

Rook を通して RBD を使用する場合、RBD 用の CSI driver の管理は Rook が行なってくれます。そのため Rook をセットアップした後は、ユーザは PVC を apply するだけで RBD ボリュームを PV として利用できるようになります。

RBD ボリュームはネットワーク経由でアクセスしなければならないため、ローカルディスクほど IO 性能は高くありません。そこで Neco では RBD を NVMe SSD によって構築し、IO 性能のペナルティを緩和しています。RBD から払い出されたボリュームは、ファイルシステムにデータを永続化する必要があるがソフトウェア自身に冗長化の機能がないものに使われています。

### RGW

Neco で利用しているもう一つの Rook/Ceph の機能である RGW は高可用なオブジェクトストレージです。RGW は Amazon S3 API 互換の API を備えています。そのため、RGW のユーザは Amazon S3 のクライアントライブラリを利用することで RGW にオブジェクトを出し入れできます。この特徴は、自社で開発するソフトウェアだけではなく OSS を動かす場合にもメリットになります。例えばログ基盤として有名な [Loki](https://grafana.com/oss/loki/) はデータの保存先として Amazon S3 をサポートしています。そこで Neco では RGW をデータ保存先として Loki を動かしています。

RGW も Rook によって CRD が用意されており、PV・PVC と同じような感覚で利用できるようになっています。ユーザが RGW 上にバケットを作成したい場合、まず `ObjectBucketClaim` リソースを作成します。すると、Rook が RGW 上にバケットを作成した後、それを `ObjectBucket` リソースとして Kubernetes 上で管理できるようにします。その後、ユーザが作成した `ObjectBucketClaim` と Rook が作成した `ObjectBucket` が紐付けられます。この際、バケットに接続するために必要な認証情報（ `AWS_ACCESS_KEY_ID` や `AWS_SECRET_ACCESS_KEY` ）は `Secret` リソースの形で Rook によって作成されます。ユーザはこの値を読み込んでバケットに接続できます。

Neco では [local PV](https://kubernetes.io/docs/concepts/storage/volumes/#local) として認識させた HDD の上に RGW を構築し、先に述べたログデータのほか、バックアップデータの保存に利用しています。また、お客様がアップロードした添付ファイルや画像などのデータの保存も RGW で実施しています。Cybozu.com ではこのようなデータの保存に SSD ではなく HDD を用いることで、運用にかかるコストを抑えています。

## おわりに

この記事では、Neco でサポートしている各種ストレージについて紹介しました。動作するソフトウェアに応じて最適なものを使い分けられるように、Neco では特性の異なる複数種類のストレージを提供しています。具体的には、ノードローカルなストレージとして TopoLVMを、障害耐性のあるストレージとして Rook/Ceph の RBD と RGW を用いています。提供されるストレージは全て Kubernetes の仕組みと統合されており、バックエンドの仕組みを気にすることなく共通のインターフェースを通じて利用できます。これにより、クラウド基盤のユーザが容易にストレージを利用できるようになっています。

「【連載】Cybozu.comクラウド基盤の全貌」では、私たちが運用しているクラウド基盤を連載形式で紹介しています。次回は Neco のデータベースについてご紹介する予定です。ご期待ください！

### あわせて読みたい

- [サイボウズサマーインターン2022 報告 〜 Kubernetes基盤開発コース&ストレージコース](https://blog.cybozu.io/entry/2022/09/07/135414)
- [トラブルの芽を摘むための一歩進んだOSSのアップグレード戦略](https://blog.cybozu.io/entry/2022/07/01/131640)
- [サイボウズサマーインターン2021 報告 〜 Kubernetes基盤開発コース](https://blog.cybozu.io/entry/2021/09/10/170132)
- [サイボウズの新しいインフラ基盤を支えるストレージの本番適用に向けた取り組み](https://blog.cybozu.io/entry/2021/02/10/191729)
- [分散ストレージCephのオーケストレータRookのデータ破壊バグを修正しました](https://blog.cybozu.io/entry/2021/01/28/065400)

[« プロダクト分散開発を支えるE2Eテスト基盤](https://blog.cybozu.io/entry/2025/07/23/170000) [サイボウズで利用可能な AI コーディング… »](https://blog.cybozu.io/entry/2025/07/17/170000)