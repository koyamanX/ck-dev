---
title: "My RISC-V debug feature part7 (examine関数(riscv013)の読解, DMの実装)"
date: 2021-05-16T20:56:07+09:00
author: "@koyamanX"
categories: ["RISC-V"]
tags: ["RISC-V", "FPGA", "JTAG"]
draft: true
---

今回から、`examine`関数を読み、初期化のフローを理解する。
DMを実装をして、`examine`で初期化をできるようにしていく。
<!--more-->

## examine関数
`examine`関数はriscv013用のインターフェイスより呼び出される。

`src/target/riscv/riscv-013.c`
```c
struct target_type riscv013_target = {
	.name = "riscv",

	.init_target = init_target,
	.deinit_target = deinit_target,
	.examine = examine,

	.poll = &riscv_openocd_poll,
	.halt = &riscv_halt,
	.step = &riscv_openocd_step,

	.assert_reset = assert_reset,
	.deassert_reset = deassert_reset,

	.write_memory = write_memory,

	.arch_state = arch_state
};
```

### examine関数の実体
`src/target/riscv/riscv-013.c:examine`
```c
static int examine(struct target *target)
{
	/* Don't need to select dbus, since the first thing we do is read dtmcontrol. */

	uint32_t dtmcontrol = dtmcontrol_scan(target, 0);
	LOG_DEBUG("dtmcontrol=0x%x", dtmcontrol);
	LOG_DEBUG("  dmireset=%d", get_field(dtmcontrol, DTM_DTMCS_DMIRESET));
	LOG_DEBUG("  idle=%d", get_field(dtmcontrol, DTM_DTMCS_IDLE));
	LOG_DEBUG("  dmistat=%d", get_field(dtmcontrol, DTM_DTMCS_DMISTAT));
	LOG_DEBUG("  abits=%d", get_field(dtmcontrol, DTM_DTMCS_ABITS));
	LOG_DEBUG("  version=%d", get_field(dtmcontrol, DTM_DTMCS_VERSION));

```
まずは、`dtmcontrol_scan`にて、DTM内の`dtmcs`をスキャンする。

### dmicsレジスタ
`dtmcs`レジスタのフォーマットは以下の通り。
アドレス(IR)は`0x10`である。
{{<figure src="./image00.png" >}}

各フィールドの意味を以下に示す。(riscv-debug-spec p.88)

{{<figure src="./image01.png" >}}
#### dmihardreset
`1`をこのフィールドに書き込むことで、DTMをハードリセットする。
ハードリセットは実行中のDMIトランザクションを終了させる。また、すべてのDTM内部状態、レジスタを初期状態にする。
#### dmireset
`1`を書き込むことで、`sticky error`状態をクリアする。また、DMIトランザクションはキャンセルしない。

#### idle
デバッカに与えるヒントで、DMI scan後に必要なRun-Test/Idleサイクル数を保持する。
必要に応じて、デバッカは`dmics.dmistat`を確認する。
```
0: 不要
1: 1
2: 2
...
```

#### dmistat
```
0: no error
1: reserved(same as 2)
2: An operation failed
3: An operation was attempted while DMI access was still in progress
```

{{<figure src="./image02.png" >}}
#### abits
The size of address in dmi

#### version
```
0: version 0.11
1: version 0.13 and 1.0
15: version not described in any spec
```

### examine続き

`src/target/riscv/riscv-013.c:examine`

```c
	if (dtmcontrol == 0) {
		LOG_ERROR("dtmcontrol is 0. Check JTAG connectivity/board power.");
		return ERROR_FAIL;
	}
	if (get_field(dtmcontrol, DTM_DTMCS_VERSION) != 1) {
		LOG_ERROR("Unsupported DTM version %d. (dtmcontrol=0x%x)",
				get_field(dtmcontrol, DTM_DTMCS_VERSION), dtmcontrol);
		return ERROR_FAIL;
	}
```
読み出した、`dtmcontrol(dtmcs)`を確認する。
`riscv013`では、versionが`1`がリセット値であることが保証されているので、`dtmcontrol`が0であるとき、正しく読み出せていないことになる。

つぎに、`dtmcs.version`を読み出し、`1(version 0.13 and 1.0)`であるか確認している。


`src/target/riscv/riscv-013.c:examine`
```c
	riscv013_info_t *info = get_info(target);
	/* TODO: This won't be true if there are multiple DMs. */
	info->index = target->coreid;
	info->abits = get_field(dtmcontrol, DTM_DTMCS_ABITS);
	info->dtmcs_idle = get_field(dtmcontrol, DTM_DTMCS_IDLE);

	/* Reset the Debug Module. */
	dm013_info_t *dm = get_dm(target);

```

読み出した、`index`, `abits`, `idle`は`info`構造体へ保存する。

#### get\_info関数
`get_info`関数では、`riscv`のバージョン固有の`info`構造体を取得する。
今回の場合は、`riscv013`用のinfo構造体(`riscv013_info_t`)を取得する。

`src/target/riscv/riscv-013.c`
```c
static riscv013_info_t *get_info(const struct target *target)
{
	riscv_info_t *info = (riscv_info_t *) target->arch_info;
	assert(info);
	assert(info->version_specific);
	return (riscv013_info_t *) info->version_specific;
}
```


`src/target/riscv/riscv-013.c`
`riscv013_info_t`抜粋
```c
typedef struct {
	/* The indexed used to address this hart in its DM. */
	unsigned index;
	/* Number of address bits in the dbus register. */
	unsigned abits;
	/* Number of abstract command data registers. */
	unsigned datacount;
	/* Number of words in the Program Buffer. */
	unsigned progbufsize;

	/* We cache the read-only bits of sbcs here. */
	uint32_t sbcs;

...

	dm013_info_t *dm;
} riscv013_info_t;
```
DTMやDMの情報を保持する。

#### get\_dm関数

この関数では、ターゲットの`DM`構造体を返す。
もし、見つからない場合は、作成し初期化する。
構造体としては、`dm013_info_t`と同じである。

### examine続き
`src/target/riscv/riscv-013.c:examine`
```c
	if (!dm)
		return ERROR_FAIL;
	if (!dm->was_reset) {
		dmi_write(target, DM_DMCONTROL, 0);
		dmi_write(target, DM_DMCONTROL, DM_DMCONTROL_DMACTIVE);
		dm->was_reset = true;
	}

	dmi_write(target, DM_DMCONTROL, DM_DMCONTROL_HARTSELLO |
			DM_DMCONTROL_HARTSELHI | DM_DMCONTROL_DMACTIVE |
			DM_DMCONTROL_HASEL);
	uint32_t dmcontrol;
	if (dmi_read(target, &dmcontrol, DM_DMCONTROL) != ERROR_OK)
		return ERROR_FAIL;

	if (!get_field(dmcontrol, DM_DMCONTROL_DMACTIVE)) {
		LOG_ERROR("Debug Module did not become active. dmcontrol=0x%x",
				dmcontrol);
		return ERROR_FAIL;
	}
```
`dm`が初期化されていない場合は、初期化をする。
初期では、`DM_DMCONTROL`(`DTM`でないことに注意)をまず、0で初期化する。
`dmi_write`は`dbus(DMI)`を経由して、`DM`のレジスタを書き込みする。
今回は、`DM_DMCONTORL`アドレスに、`0`を書き込む。


その後、`DMACTIVE`フィールドを1にする。
{{<figure src="./image03.png" >}}
このフィールドはDMのリセット信号として機能する。
このフィールドに書き込みをした場合は、次のアクションを行う前に、書き込んだ値が反映されているかポーリングが必要。
このフィールドが`0`の場合は、DMはすべて初期状態であることを示し、DMに対するアクセスはすべて失敗する。
また、`version`フィールドは正しい値を返さないかもしれない。
`1`の場合は、DMが正しく動作していることを示す。

次に、`DM_DMCONTROL_DMACTIVE`の他に、`WARL`フィールドに`-1`を書き込む。
読み出したっているビットを見ることで、DMがサポートする機能がわかる。
書き込むフィールドは`DM_DMCONTROL_HARTSEL`、`DM_DMCONTROL_HARTSELLO`、`DM_DMCONTROL_HARTSELHI`である。
これらは、hartの選択方法を規定するものである。（詳しくは後ほど)
```c
	dmi_write(target, DM_DMCONTROL, DM_DMCONTROL_HARTSELLO |
			DM_DMCONTROL_HARTSELHI | DM_DMCONTROL_DMACTIVE |
			DM_DMCONTROL_HASEL);
```

次に、`dmi_read`にて`DM_DMCONTROL`を読み出し、`DMACTIVE`ビットが1であるか検査をする。
`0`の場合は、DMはアクティブでない。

### 今日はここまで
今回はここまで。
ちなみに、`DMACTIVE`までは、DMを実装している。
そのため、`examine`は途中まで成功している。
次回は、まだ実装していない`hart`の選択方法から見ていく。
なんか、投稿のフォーマットがぐちゃぐちゃで見づらい説はある。
なんかいい方法ないかな。
あと、表をいい感じに書きたい。
