## 参考ドキュメント
####[前回ドキュメント](http://qiita.com/akachochin/items/e7d176d241d5c19bf991)
####[ASADA氏の公開記事](http://syuu1228.github.io/howto_implement_hypervisor/)
この記事から引用したときは「ChapterXX」とする。例えば第15回の記事なら(Chapter15)となる。

####[Intel ® Virtualization Technology for Directed I/O Architecture Specification](http://www.intel.co.jp/content/www/jp/ja/intelligent-systems/intel-technology/vt-directed-io-spec.html) 
このドキュメントから引用したときは(仕様書:XXX)とする。

## mappingの開始
次に、iommu_init()に戻り、iommu_create_mapping()を呼び出し、先に割り当てたドメイン(のページテーブル)に対してマッピングを行う。
そのドメインはホスト用の(特殊な)ドメインであることに注意する。
このドメイン自体がホストの物理アドレス空間を示すので、そのマッピングは、ホストドメインを示す物理アドレス空間 == ホストの物理アドレス空間となる1:1のマッピングである。

```c
	/*
	 * Create 1:1 mappings from '0' to 'maxaddr' for devices assigned to
	 * the host
	 */
	iommu_create_mapping(host_domain, 0, 0, maxaddr);
```
## maxaddrについて
ところで、maxaddrとはなんだろうか。これは、先に書いたIOMMU_CREATE_DOMAIN()を割り当てる直前に以下の関数を呼び出すことで行っている。

```c
	maxaddr = vmm_mem_maxaddr();
```

vmm_mem_maxaddr()は以下の関数でグローバル変数Maxmemをページ単位からアドレス単位にした値を返している。

```c
vm_paddr_t
vmm_mem_maxaddr(void)
{

	return (ptoa(Maxmem));
}
```

Maxmemはハイパーバイザとは関係なく、機種(amd64)依存のカーネルブートコードに存在する変数であり、以下のように初期化される。

```c
static void
getmemsize(caddr_t kmdp, u_int64_t first)
{
(省略)
	/*
	 * Maxmem isn't the "maximum memory", it's one larger than the
	 * highest page of the physical address space.  It should be
	 * called something like "Maxphyspage".  We may adjust this
	 * based on ``hw.physmem'' and the results of the memory test.
	 */
	Maxmem = atop(physmap[physmap_idx + 1]);
```

上記のコードから、初期値として、ホストの物理アドレス空間の最上位のアドレスに対応したページ番号 + 1した値が格納されていることがわかる。

## ページテーブルの全体図
IOMMUを使って、アドレス変換をするために、以下のようなページテーブルを作る必要がある。

![table-iommu.jpg](https://qiita-image-store.s3.amazonaws.com/0/19975/a3b01089-9cdd-2487-944d-e7d13e5c88f2.jpeg)

前回説明していなかったが、前回書いたvtd_create_domain()内のdomain構造体割り当て処理で割り当てたページテーブルは図中のPage Map(AGAWが48bitの場合)もしくはPage Directory Pointer(AGAWが39bit)である。

そして、以下で書くiommu_create_mapping()では、引数で指定されたアドレス範囲をマッピングするのに必要な範囲でPage Tableまでの割り当てと設定を行う。

## iommu_create_mapping()
この関数では、gpa(ゲスト物理アドレス。ここではホスト用のドメインの物理アドレス空間)とhpa(ホスト物理アドレス)のマッピングを指定された長さ分だけ行う。
おおよそ以下のようなことを実施する。
(1)すでに割り当て済みの最上位のページテーブル(Page MapもしくはPage Directory Pointer)にマッピングに必要な設定をする。
(2)それ以下のページテーブル(Page Directory Pointer,Page Directory,Page Table)を必要な範囲で割り当ててそれぞれにマッピングに必要な設定をする。

マッピングを行っているのは実際にはIOMMU_CREATE_MAPPING(),すなわちvtd_create_mapping()である。

なお、先の図中のRoot-TableとContext-Tableはここでは何も扱わない。
Root-TableとContext-Tableは特定のPCIデバイスが使うゲスト物理アドレスとホスト物理アドレスの対応を知る(=ページテーブルを特定する)ために存在するものだからである。

```c
void
iommu_create_mapping(void *dom, vm_paddr_t gpa, vm_paddr_t hpa, size_t len)
{
	uint64_t mapped, remaining;

	remaining = len;

	while (remaining > 0) {
		mapped = IOMMU_CREATE_MAPPING(dom, gpa, hpa, remaining);
		gpa += mapped;
		hpa += mapped;
		remaining -= mapped;
	}
}
```

vtd_create_mapping()はvtd_update_mapping()を呼んでいるだけなのでvtd_update_mapping()を見る。

```c
static uint64_t
vtd_update_mapping(void *arg, vm_paddr_t gpa, vm_paddr_t hpa, uint64_t len,
		   int remove)
{
(省略)
```

まずは、ホスト用の物理アドレス空間と1:1対応の変換を行うために十分なアドレス幅(シフト数)を求める。
これは、ページテーブルの深さを決めるために必要である。

```c
	/*
	 * Compute the size of the mapping that we can accomodate.
	 *
	 * This is based on three factors:
	 * - supported super page size
	 * - alignment of the region starting at 'gpa' and 'hpa'
	 * - length of the region 'len'
	 */
	spshift = 48;
	for (i = 3; i >= 0; i--) {
		spsize = 1UL << spshift;
		if ((dom->spsmask & (1 << i)) != 0 &&
		    (gpa & (spsize - 1)) == 0 &&
		    (hpa & (spsize - 1)) == 0 &&
		    (len >= spsize)) {
			break;
		}
		spshift -= 9;
	}
```

ここで、ホスト物理アドレスと1:1対応したマッピングを行う。マッピングは最上位のページテーブルから行い、順次下位に向かってページテーブルを割り当てていくようなイメージで行う。

```c
	ptp = dom->ptp;
	nlevels = dom->pt_levels;
	while (--nlevels >= 0) {
```

そのために、現在書き込み対象になっているページテーブルのどこに書き込むか、gpaからインデックスを生成する。

```c
		ptpshift = 12 + nlevels * 9;
		ptpindex = (gpa >> ptpshift) & 0x1FF;
```

もちろん、末端のページテーブル(Page Table)まですべてマッピングが完了したら、そこで終了である。

```c
		/* We have reached the leaf mapping */
		if (spshift >= ptpshift) {
			break;
		}
```

先の図の末端のページテーブル(Page Table)まで辿り着かない限り、ひたすらPage Tableに近い側に下っていく。
その過程で、ページテーブル未割り当てな箇所があれば、ページテーブルを割り当てその物理アドレスを記録する。

```c
		/*
		 * We are working on a non-leaf page table page.
		 *
		 * Create a downstream page table page if necessary and point
		 * to it from the current page table.
		 */
		if (ptp[ptpindex] == 0) {
			void *nlp = malloc(PAGE_SIZE, M_VTD, M_WAITOK | M_ZERO);
			ptp[ptpindex] = vtophys(nlp)| VTD_PTE_RD | VTD_PTE_WR;
		}
```

そして、CPUから該当ページテーブルにアクセスできるように、ページテーブルエントリ内の物理アドレスを仮想アドレスに変換する。

```c
		ptp = (uint64_t *)PHYS_TO_DMAP(ptp[ptpindex] & VTD_PTE_ADDR_M);
	}
```

ここまでの文書とソースを読んだ人は「あれ、末端のページテーブル(Page Table)のエントリをまだ設定できていないのでは？」と思うだろう。
その処理が以下の処理である。
今回は登録する処理なので、removeフラグはfalseである。

```c
	/*
	 * Update the 'gpa' -> 'hpa' mapping
	 */
	if (remove) {
		ptp[ptpindex] = 0;
	} else {
		ptp[ptpindex] = hpa | VTD_PTE_RD | VTD_PTE_WR;

		if (nlevels > 0)
			ptp[ptpindex] |= VTD_PTE_SUPERPAGE;
	}
```

これで、ホストの物理アドレス空間と1:1対応したドメインのマップが作成されたことになる。

そして、vtd_update_mapping()を抜ける。
今回はここまで。
次は、iommu_init()の後半処理について書く予定。
大まかに言うと以下二点。
(1)デバイスのコンテキストテーブルとページテーブルの紐付け(IOMMU_ADD_DEVICE)
(2)IOMMUを有効にする(IOMMU_ENABLE)

