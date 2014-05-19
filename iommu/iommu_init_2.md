## 参考ドキュメント
####[前回ドキュメント](http://qiita.com/akachochin/items/3638733711b3c41c189c)
####[ASADA氏の公開記事](http://syuu1228.github.io/howto_implement_hypervisor/)
この記事から引用したときは「ChapterXX」とする。例えば第15回の記事なら(Chapter15)となる。

####[Intel ® Virtualization Technology for Directed I/O Architecture Specification](http://www.intel.co.jp/content/www/jp/ja/intelligent-systems/intel-technology/vt-directed-io-spec.html) 
このドキュメントから引用したときは(仕様書:XXX)とする。

## domainを作る
今回はiommu_init()の以下処理から読んでみる。

```c
	/*
	 * Create a domain for the devices owned by the host
	 */
	maxaddr = vmm_mem_maxaddr();
	host_domain = IOMMU_CREATE_DOMAIN(maxaddr);
	if (host_domain == NULL)
		panic("iommu_init: unable to create a host domain");
```

IOMMU_INIT()経由でvtd_create_domain()が呼ばれる。

## AGAW
```c
	/*
	 * Calculate AGAW.
	 * Section 3.4.2 "Adjusted Guest Address Width", Architecture Spec.
	 */
```

この処理では、最初に引数で与えられた最大物理アドレスのAGAW(Adjusted Guest Address Width)を計算する。
要するに「アドレス幅がどのくらいか」である。

上記引用コメント以下でごにょごにょと計算しているが、要するに以下表のようなagaw値を作りたいのである。

| Address範囲|AGAW|
|:-----------:|------------:|
| 0x00000000_00001000 - 0x00000000_001fffff |21|
| 0x00000000_00200000 - 0x00000000_3fffffff |30|
| 0x00000000_40000000 - 0x0000007f_ffffffff |39|
| 0x00000080_00001000 - 0x0000ffff_ffffffff |48|

なぜ、AGAWの計算が必要なのか。それは以下コメントにあるとおり、ページテーブルの深さが決まるためである。

```c
	/*
	 * Select the smallest Supported AGAW and the corresponding number
	 * of page table levels.
	 */
```

次に、IOMMU側でサポートされているAGAWを取得する。

```c
	tmp = VTD_CAP_SAGAW(vtdmap->cap);
```

## ここでちょっとCAPについて
CAPについては、仕様書10.4.2「Capability Register」を見て欲しい。これはその名前のとおり、「このremapping H/Wでできることを調べる」ためのレジスタである。
このレジスタのSAGAW(Supported Adjusted Guest Address Width)は5bitのビットフィールドでこのremapping H/Wでサポートするゲスト物理アドレスのアドレス長を表す。
現時点では、bit1,2がそれぞれ39bit Address Widthと48bit Address Widthを表す。

この関数のループでは、計算から求められたAGAWとハードがサポートしているAGAWの両方を満たすAGAWの中で一番最小のものを選ぶ。おそらく、できる限りページテーブルを小さくするためだと思われる。

## domainについて
そして、domain構造体を割り当てる。仕様書3.2「Domains and Address Translation」の記述によると、「プラットホーム内の独立した環境に対して割り当てられたホストの物理アドレスのサブセット」であり、あるデバイスが使うアドレス空間を表現するためのものである。
※ただし、今回作成するドメインはホストの物理アドレス空間そのものと1:1対応する特殊なドメインを作成する。
IOMMU_CREATE_DOMAIN()を呼び出すところに以下コメントがあったことを思い出すと良い。

```c
	/*
	 * (省略)
	 * Create a domain for the devices owned by the host
	 * (省略)
	 */
struct domain {
	uint64_t	*ptp;		/* first level page table page */
	int		pt_levels;	/* number of page table levels */
	int		addrwidth;	/* 'AW' field in context entry */
	int		spsmask;	/* supported super page sizes */
	u_int		id;		/* domain id */
	vm_paddr_t	maxaddr;	/* highest address to be mapped */
	SLIST_ENTRY(domain) next;
};
```

そして、domain構造体を初期化する。
ここで一点注意することがある。
domain構造体のptpメンバはゲストのアドレス空間を表現するためのページテーブルの仮想アドレスが格納される。
(次回あたり書くが、今回はホストドメインという「ホストの物理アドレス空間」を表現するための特殊なドメインを作成する)
この時点でptpが指すページテーブルはAGAWによって決まるLevel分の段数をすべて割り当てるのではなく、ルートとなるページテーブル(Page Map)をただひとつだけ割り当てることである。

今回はここまで。次回は物理アドレス間のマップ(Page Table)について書こうと思う。

