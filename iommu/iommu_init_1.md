
##はじめに
Software Design誌でASADA氏が「ハイパーバイザの作り方」と題する記事を執筆している。
この中でbhyveというFreeBSD搭載のハイパーバイザ実装が紹介されている。

この記事にインスパイアされ、bhyveのソースを読んで調べたことなどを記録に残す。(ただ、一回では到底書ききれないので少しずつ書き溜めていこうと思う。)

##初期化処理周りを読んでみて
私は、あるソフト全体のデータ構造の雰囲気を掴みたいとき、初期化処理を読むやり方を割と好んでいる。
ということで、今回もご多分にもれず、初期化処理から入ってみる。

ただ、興味や私的な記録という性格上、必ずしも実行順にはならなかったり、解説が不十分なことは了解願いたい。
最初に記録に残すのは、IOMMUの箇所である。
ここは「ハイパーバイザの作り方」第15回と第16回に詳しい。かなりダブる箇所もあるがそれはそれということで。

##vmm.koがロードされると
vmm.koカーネルモジュールがロードされると、sys/amd64/vmm/vmm.c内のvmm_handler()がコールされる。
この中でcase MOD_LOADで書かれた処理がロード時処理である。

vmmdev_init()はmutexの初期化くらいなので、略。
次に、PCIパススルーデバイスが存在する場合、iommu_init()が呼び出される。

今回からしばらくはこの部分にスポットをあてて書いてみたいと思う。

##いきなり参考資料
####[ASADA氏の公開記事](http://syuu1228.github.io/howto_implement_hypervisor/)
この記事から引用したときは「ChapterXX」とする。例えば第15回の記事なら(Chapter15)となる。

####[Intel ® Virtualization Technology for Directed I/O Architecture Specification](http://www.intel.co.jp/content/www/jp/ja/intelligent-systems/intel-technology/vt-directed-io-spec.html) 
このドキュメントから引用したときは(仕様書:XXX)とする。

##基本的な用語のおさらいとデバイスの概要について
PCIパススルーとは、「ゲストOSから直接物理マシンのPCIデバイスにアクセスできるようにした仕組み。これによって、ハイパーバイザに制御を移さなくても済むのでパフォーマンスの向上が図れる」くらいにまずは捉えるのが良い。
なお「PCI PassThrou」なので、PPTとも呼ばれ、ソースの中ではこの略語で出てくる。

PCIメモリ空間にアクセスする際、NICのような扱うデータ量が大きいデバイスではDMAを使うのが普通。しかし、ゲストOSが"知っている"物理アドレスは本物の物理アドレスではない。よって、ゲストOSが"知っている"物理アドレスを使ってDMAをかけると他の領域を破壊する。

```
ゲストOS -(ゲスト物理)-> PCIデバイス -(ゲスト物理)-> 物理メモリ (意図しないアクセス!)
```
そこで、「物理メモリとPCIデバイスの間にMMUのような装置を置きアドレス変換を行う」(Chapter15)ハードウェアを置いてみる。


```
ゲストOS -(ゲスト物理)-> PCIデバイス -(ゲスト物理)-> IOMMU -(ホスト物理)-> 物理メモリ
```
IOMMUが単独であるわけでなく、Intel VT-dの主要な機能の一つである。
なお、(仕様書:2.5)によると、ゲスト物理アドレスからホスト物理の変換により、以下の旨みがあるとのこと。仕様書の記述は長いので以下サマリ。

####(1)OS保護：DMAが意図しない領域にアクセスするのを防ぐ
device driverによるバグなどで変な領域にDMAアクセスするのを防げる。
####(2)Legacy Deviceでバウンスバッファを使わなくても、Highmemにデータを転送できる
例えば、32bit PCIデバイスでは物理アドレスが32bit値なわけでして。
IOMMU使ってリマッピングすることで、面倒なくHIGHMEMに転送すればよい。
####(3)DMAごとの区画保護
device driverがバッファを登録しさえすればそれが物理的に細切れであってもIOMMUがよきに計らってくれるのだ。
####(4)プロセスの仮想アドレス空間を他のプロセスだけでなくデバイスと共有する。
(プロセスと共有するだけなら、CPUのMMUうまく使えばよいが、デバイスともなると物理メモリを共有しなければならず、したがって物理アドレス管理が必要)

##iommu_init()の実装
iommu_init()の実装を抜粋しつつ読んでみる。

```c
void
iommu_init(void)
(省略)
    if (vmm_is_intel())
        ops = &iommu_ops_intel;
```

当然、IOMMUの制御はIntelとAMDで異なるため、要所となる制御は関数ポインタで呼び分けするようにしている。

```c
    error = IOMMU_INIT();
```

この関数は

```c
static __inline int
IOMMU_INIT(void)
{
    if (ops != NULL)
        return ((*ops->init)());
    else
        return (ENXIO);
}
```

であり、先の関数ポインタテーブルを使った呼び分けをしている。IOMMU_XXXX()となっているものは基本的に同様である。
Intelの場合は、vtd_init()がコールされる。

vtd_init()では、

```c
    /*
     * (略)
     * Allow the user to override the ACPI DMAR table by specifying the
     * physical address of each remapping unit.
r)
     * (略)
     */
    for (units = 0; units < DRHD_MAX_UNITS; units++) {
        snprintf(envname, sizeof(envname), "vtd.regmap.%d.addr", units);
        if (getenv_ulong(envname, &mapaddr) == 0)
            break;
        vtdmaps[units] = (struct vtdmap *)PHYS_TO_DMAP(mapaddr);
    }
```
とあり、環境変数経由でvtdmapsを書き換えている。
この処理は「本来はACPI経由でDMA Remapping Reporting Structure(後述)の先頭物理アドレスを取得する。しかし、環境変数で指定されている場合、そちらを優先的に使う。そして、取得した情報内のRemapping Hardwareレジスタ空間物理アドレスを使い仮想アドレス空間を割り付けている(後述)」だと思われる。

```c
    /* Search for DMAR table. */
    status = AcpiGetTable(ACPI_SIG_DMAR, 0, (ACPI_TABLE_HEADER **)&dmar);
    if (ACPI_FAILURE(status))
        return (ENXIO);
```

環境変数で取得できない場合、本来の動作である「ACPI経由でDMA Remapping Reporting Structureの物理アドレス取得」を行う。
で、IOMMUの数だけ

```c
    vtdmaps[units++] = (struct vtdmap *)PHYS_TO_DMAP(drhd->Address);
```
を行い、Remapping Hardwareのレジスタ空間の物理アドレスを仮想アドレスに変換(というか特定の仮想アドレスに割り付けて)いる。

##DMA Remapping Reporting Structureとは
仕様書8.1に掲載されているデータ構造で先頭が"DMAR"で始まるヘッダ＋データテーブルの構造になっている(だからこそ、AcpiGetTable()の第一引数は"DMAR"である。)。これはRemapping Hardwareに関する情報が格納されており、BIOSが用意する。
このヘッダの中にレジスタ空間のアドレスがあり、これを使ってRemapping Hardwareの制御を行う。

次のドキュメントは「ドメインを作る」(IOMMU_CREATE_DOMAIN()以下)の予定である。

