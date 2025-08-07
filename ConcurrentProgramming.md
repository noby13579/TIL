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

## デフォルトのmemory_order_seq_cstでどのような命令に展開されるか調べる。
- memory_orderについて
https://cpprefjp.github.io/reference/atomic/memory_order.html
memory_order_seq_cst: sequential consistency ＝ 順序一貫性。最も強い保証。

### C++ソース
```cpp
#include <atomic>
#include <iostream>

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

int	atomicFetchAdd(int iAddVal)
{
	return 	atomicInt.fetch_add(iAddVal);
}
int	atomicFetchOr(int iOrVal)
{
	return 	atomicInt.fetch_or(iOrVal);
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
    dmb命令＜前＞のすべての明示的なメモリアクセスが、  
    dmb命令＜後＞のすべての明示的なメモリアクセスよりも実行されることを保証する。
  - ish: inner shareable domain.  
    [Memory types and attributes and the memory order model](https://developer.arm.com/documentation/ddi0406/c/Application-Level-Architecture/Application-Level-Memory-Model/Memory-types-and-attributes-and-the-memory-order-model?lang=en) からリンクされている [Normal memory](https://developer.arm.com/documentation/ddi0406/c/Application-Level-Architecture/Application-Level-Memory-Model/Memory-types-and-attributes-and-the-memory-order-model/Normal-memory?lang=en) の「Shareable, Inner Shareable, and Outer Shareable Normal memory」参照。  
    Inner Shareableは、同一クラスター内で、整合性が保証される。  
    例）Cortex-A9 MP x4 マルチコア内で整合性が保証される。
  - 下記の例の場合、CPUコア1からアドレス1から値2が読めたら、アドレス0から値1が読めることが保証される。
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
    理由・効果は要調査。アトミック変数へのstoreの＜後＞のメモリアクセス命令が、アトミック変数のstore後に実行されることを保証している(確かに順序の保証は強くなっている。。。)が、それが何を意味するのか、など。

- 余談：コンパイルオプションでCPU指定をしなかったら、`dmb ish`の代わりに、`__sync_synchronize`サブルーチン呼び出しになり、命令数も２倍くらい(スタックへのpush/pop等が増えた)になった。

### atomic store, loadの命令（ARMv8）
ARMv8：`stlr`, `ldar`になった。
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
todo: 記載。メモリオーダーを実現するいい感じのload/store命令があるらしい。
stlr: store release  
https://developer.arm.com/documentation/ddi0602/2025-06/Base-Instructions/STLR--Store-release-register-?lang=en  
ldar: load acquire  
https://developer.arm.com/documentation/ddi0602/2025-06/Base-Instructions/LDAR--Load-acquire-register-?lang=en


下記はARMv8のメモリ属性の解説
https://developer.arm.com/documentation/100941/0101/Memory-attributes
