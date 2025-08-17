# atomic変数アクセスのARMv7とARMv8比較
## コンパイラ
ARMv7：arm-linux-gnueabi-g++ -O2 -mcpu=cortex-a9 でコンパイルする。
```bash
$ arm-linux-gnueabi-g++ --version
arm-linux-gnueabi-g++ (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0
Copyright (C) 2023 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```
ARMv8：aarch64-linux-g++ -O2でコンパイルする。
```bash
$ aarch64-linux-g++ --version
aarch64-linux-g++.br_real (Buildroot 2021.11-12449-g1bef613319) 13.3.0
Copyright (C) 2023 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

## 単独のatomic変数のstore/loadがどのような命令にコンパイルされるか調べる。
デフォルトのmemory_order_seq_cstでどのような命令に展開されるか調べる。
- memory_orderについて  
https://cpprefjp.github.io/reference/atomic/memory_order.html  
memory_order_seq_cst: sequential consistency ＝ 順序一貫性。最も強い保証。

### C++ソース
```cpp
#include <atomic>

namespace {
std::atomic<int> atomicInt{};
}	// namespace

void	atomicStore(int iVal)
{
	atomicInt.store(iVal);
}
int	atomicLoad(void)
{
	return atomicInt.load();
}
```

### atomic store, loadの命令（ARMv7）
ARMv7：`dmb ish` → `str`,`ldr` → `dmb ish`になった。
```armasm
00010a28 <_Z11atomicStorei>:
   10a28:	e30230d0 	movw	r3, #8400	@ 0x20d0
   10a2c:	f57ff05b 	dmb	ish
   10a30:	e3403001 	movt	r3, #1
   10a34:	e583001c 	str	r0, [r3, #28]
   10a38:	f57ff05b 	dmb	ish
   10a3c:	e12fff1e 	bx	lr

00010a40 <_Z10atomicLoadv>:
   10a40:	e30230d0 	movw	r3, #8400	@ 0x20d0
   10a44:	f57ff05b 	dmb	ish
   10a48:	e3403001 	movt	r3, #1
   10a4c:	e593001c 	ldr	r0, [r3, #28]
   10a50:	f57ff05b 	dmb	ish
   10a54:	e12fff1e 	bx	lr
```

- `dmb ish`とは？  
https://developer.arm.com/documentation/dui0802/b/A32-and-T32-Instructions/DMB--DSB--and-ISB?lang=en  
  - dmb: data memory barrier.  
    プログラム順でdmb命令＜前＞のすべての明示的なメモリアクセスが、  
    プログラム順でdmb命令＜後＞のすべての明示的なメモリアクセスよりも先に観測されることを保証する。
  - ish: inner shareable domain.  
    [Memory types and attributes and the memory order model](https://developer.arm.com/documentation/ddi0406/c/Application-Level-Architecture/Application-Level-Memory-Model/Memory-types-and-attributes-and-the-memory-order-model?lang=en) からリンクされている [Normal memory](https://developer.arm.com/documentation/ddi0406/c/Application-Level-Architecture/Application-Level-Memory-Model/Memory-types-and-attributes-and-the-memory-order-model/Normal-memory?lang=en) の「Shareable, Inner Shareable, and Outer Shareable Normal memory」参照。  
    Inner Shareableは、同一クラスター内で、memory orderingが保証される。  
    例）Cortex-A9 MP x4 マルチコア内でmemory orderingが保証される。
  - 下記の例の場合、CPUコア1でアドレス1から値2が読めたら、アドレス0から値1が読めることが保証される。
    ```bash
    CPUコア0            CPUコア1
    str 値1 アドレス0  
    dmb ish  
    str 値2 アドレス1                   # アトミック変数へのstore  
                        ldr アドレス1   # アトミック変数からのload。ここで値2が読めたら...
                        dmb ish
                        ldr アドレス0   # ここで値1が読めることが保証される。
    ```
    この例は、  
    　　`atomic::store`に`std::memory_order_release`を指定して`str`の＜前だけ＞に`dmb`が追加され、  
    　　`atomic::load` に`std::memory_order_acquire`を指定して`ldr`の＜後だけ＞に`dmb`が追加され  
    ている状況と同じだが、デフォルトの最も強い保証の`std::memory_order_seq_cst`に含まれる保証。  
    デフォルトの`std::memory_order_seq_cst`だと、アトミック変数のアドレスへの`str`,`ldr`の＜前にも後にも＞`dmb`が追加される。  
    理由・効果は要調査。たとえばアトミック変数へのstoreの＜後＞にあるメモリアクセス命令が、アトミック変数のstore後に実行されることを保証している(確かに順序の保証は強くなっている。。。)が、それが何を意味するのか、など。

- 余談：コンパイルオプションでCPU指定をしなかったら、`dmb ish`の代わりに、`__sync_synchronize`サブルーチン呼び出しになり、命令数も２倍くらい(スタックへのpush/pop等が増えた)になった。

### atomic store, loadの命令（ARMv8）
ARMv8：`stlr`(Store Release Register), `ldar`(Load Acquire Register)になった。
```armasm
0000000000000ea0 <_Z11atomicStorei>:
 ea0:	f0000001 	adrp	x1, 3000 <__data_start>
 ea4:	91006021 	add	x1, x1, #0x18
 ea8:	91008021 	add	x1, x1, #0x20
 eac:	889ffc20 	stlr	w0, [x1]
 eb0:	d65f03c0 	ret
 eb4:	d503201f 	nop
 eb8:	d503201f 	nop
 ebc:	d503201f 	nop

0000000000000ec0 <_Z10atomicLoadv>:
 ec0:	f0000000 	adrp	x0, 3000 <__data_start>
 ec4:	91006000 	add	x0, x0, #0x18
 ec8:	91008000 	add	x0, x0, #0x20
 ecc:	88dffc00 	ldar	w0, [x0]
 ed0:	d65f03c0 	ret
 ed4:	d503201f 	nop
 ed8:	d503201f 	nop
 edc:	d503201f 	nop

```
[Load-Acquire and Store-Release instructions](https://developer.arm.com/documentation/102336/0100/Load-Acquire-and-Store-Release-instructions)
- stlr = Store Release Register  
  プログラム順でstlr命令＜前＞のすべての明示的なメモリアクセスが、stlr命令より前に観測されることを保証する。  
  大雑把には、ARMv7の dmb → str。
- ldar = Load Aqcuire Register  
  プログラム順でldar命令＜後＞のすべての明示的なメモリアクセスが、ldar命令より後に観測されることを保証する。  
  大雑把には、ARMv7の ldr → dmb。


### 単独のatomic変数のstore/loadの調査結果
「メモリバリア」を伴うstore/load命令にコンパイルされる。
前提となる知識として、CPU等は、必ずしもプログラム順＝アセンブリ言語の順にメモリアクセスするとは限らず、効率化のためにメモリアクセスの順序を入れ替える（reorderingする）ことがある。
[Memory ordering](https://developer.arm.com/documentation/102336/0100/Memory-ordering)参照。  
CPU等によるメモリアクセスのreorderingを抑止するのが、メモリバリア命令。  

ここまでは、atomic変数を使うことで、メモリバリアを伴う命令にコンパイルされ、CPU等によるメモリアクセスのreorderingが抑止されることにより、順序保証されることを見てきた。  
ここから、C++のソースコードで、C++のアトミック変数アクセスを利用することで、どのようにプログラム順(アセンブリ言語の順番)が変わるかを実験してみる。

## 非atomic変数とatomic変数が混在するstoreがどのような命令にコンパイルされるか調べる。
### C++ソース
対比用のatomic変数(`a`)・非atomic変数(`nonAtomic`)のオフセットアドレスを、他の２つの非atomic変数の間にすることで、最適化されることを期待する。
```cpp
#include <atomic>

struct AtomicStruct {
	int				      	v{};
	std::atomic<int>	a{};
	int		      			w{};
};

struct NonAtomicStruct {
	int	              v{};
	int	              notAtomic{};
	int	              w{};
};

namespace {
AtomicStruct	atomicStruct{};
NonAtomicStruct	nonAtomicStruct{};
}	// namespace

void setAtomicStruct(void)
{
	atomicStruct.v = 1;
	atomicStruct.w = 2;
	atomicStruct.a.store(3);
}
void setNonAtomicStruct(void)
{
	nonAtomicStruct.v = 1;
	nonAtomicStruct.w = 2;
	nonAtomicStruct.notAtomic = 3;
}

```

### 非atomic変数・atomic変数混在と、非atomic変数のみとの比較(ARMv7)
```armasm
00010854 <_Z15setAtomicStructv>:
   10854:	e30230d0 	movw	r3, #8400	@ 0x20d0
   10858:	e3a00001 	mov	r0, #1
   1085c:	e3403001 	movt	r3, #1
   10860:	e3a01002 	mov	r1, #2
   10864:	e3a02003 	mov	r2, #3
   10868:	e5830000 	str	r0, [r3]        ※r0の値(1)をオフセット+0にstore。atomicStruct.v = 1。
   1086c:	e5831008 	str	r1, [r3, #8]    ※r1の値(2)をオフセット+8にstore。atomicStruct.w = 2。
   10870:	f57ff05b 	dmb	ish             ※メモリバリア
   10874:	e5832004 	str	r2, [r3, #4]    ※r2の値(3)をオフセット+4にstore。atomicStruct.a.store(3)。
   10878:	f57ff05b 	dmb	ish
   1087c:	e12fff1e 	bx	lr

00010880 <_Z18setNonAtomicStructv>:
   10880:	e30230d0 	movw	r3, #8400	@ 0x20d0
   10884:	e3a02002 	mov	r2, #2
   10888:	e3403001 	movt	r3, #1
   1088c:	e3a00001 	mov	r0, #1
   10890:	e3a01003 	mov	r1, #3
   10894:	e5832018 	str	r2, [r3, #24]     ※r2の値(2)を、オフセット+24にstore。nonAtomicStruct.w = 2。
   10898:	e1c301f0 	strd	r0, [r3, #16]   ※store double: r0, r1の値(1,3)を、オフセット+16, +20にstore。nonAtomicStruct.v = 1 と nonAtomicStruct.nonAtomic = 3。
   1089c:	e12fff1e 	bx	lr
```
- setAtomicStructのコンパイル結果
  atomic変数のstoreは、C++ソースの通り、最後となった。また、メモリバリア命令が挿入された。  
  また、オフセットアドレスが連続しているのにも関わらず、atomic変数と非atomic変数の２つの変数アクセスの命令はまとめられなかった。
- setNonAtomicStructのコンパイル結果
  メモリアクセスのプログラム順（アセンブリ言語順）は、C++ソースと順番が入れ替わった。  
  また、オフセットアドレスが連続する非atomicの２つの変数のアクセスが、１つのstore double命令にまとめられた。

### 非atomic変数・atomic変数混在と、非atomic変数のみとの比較(ARMv8)
```armasm
0000000000000cc0 <_Z15setAtomicStructv>:
 cc0:	f0000001 	adrp	x1, 3000 <__data_start>
 cc4:	91006020 	add	x0, x1, #0x18
 cc8:	52800023 	mov	w3, #0x1                   	// #1
 ccc:	52800042 	mov	w2, #0x2                   	// #2
 cd0:	b9001823 	str	w3, [x1, #24] ※w3の値(1)をオフセット+24にstore。        atomicStruct.v = 1。
 cd4:	b9000802 	str	w2, [x0, #8]  ※w2の値(2)をオフセット+0x18+8=+32にstore。atomicStruct.w = 2。
 cd8:	52800061 	mov	w1, #0x3                   	// #3
 cdc:	91001000 	add	x0, x0, #0x4
 ce0:	889ffc01 	stlr	w1, [x0]    ※stlr命令＝メモリバリアを伴う。w1の値(3)をオフセット+0x18 +0x04=+28にstore。atomicStruct.a.store(3)。
 ce4:	d65f03c0 	ret
 ce8:	d503201f 	nop
 cec:	d503201f 	nop

0000000000000cf0 <_Z18setNonAtomicStructv>:
 cf0:	d2800022 	mov	x2, #0x1                   	// #1
 cf4:	52800041 	mov	w1, #0x2                   	// #2
 cf8:	f2c00062 	movk	x2, #0x3, lsl #32
 cfc:	f0000000 	adrp	x0, 3000 <__data_start>
 d00:	91006000 	add	x0, x0, #0x18
 d04:	f9000802 	str	x2, [x0, #16]   ※64bitのx2の値(0x00000003'00000001)をオフセット+16にstore。 nonAtomicStruct.v = 1 とnonAtomicStruct.nonAtomic = 3。
 d08:	b9001801 	str	w1, [x0, #24]   ※32bitのw1の値(2)をオフセット+24にstore。                   nonAtomicStruct.w = 2。
 d0c:	d65f03c0 	ret
```
- setAtomicStructのコンパイル結果
  atomic変数のstoreは、C++ソースの通り、最後となった。また、メモリバリアを伴う命令（stlr = Store Release Register）になった。  
  また、オフセットアドレスが連続しているのにも関わらず、atomic変数と非atomic変数の２つの変数アクセスの命令はまとめられなかった。
- setNonAtomicStructのコンパイル結果
  メモリアクセスのプログラム順（アセンブリ言語順）は、C++ソースと順番が入れ替わった。  
  また、オフセットアドレスが連続する非atomicの２つの変数のアクセスが、１つの64bit store命令にまとめられた。

### 非atomic変数とatomic変数が混在するstoreの調査結果
C++ソース上でatomic変数のstoreアクセスよりも前のメモリアクセスは、atomic変数storeよりも前に命令が並ぶようにコンパイルされた。  
アドレスが連続していて最適化されそうなメモリアドレス順のアクセスであっても、atomic変数へのアクセスは独立した命令にコンパイルされた。他のメモリアクセスとまとめて１つの命令にはならなかった。  
atomic変数アクセスがメモリバリアを伴う命令にコンパイルされた。前述のとおり。
