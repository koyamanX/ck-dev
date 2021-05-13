---
title: "My RISC-V debug feature part6 (target\_examine, riscv\_examine)"
date: 2021-05-13T23:02:58+09:00
author: "@koyamanX"
categories: ["RISC-V"]
tags: ["RISC-V", "FPGA", "JTAG"]
draft: false
---

今回も引き続き、OpenOCDのコードを読んでいく。
`target_examine`と`riscv_examine`を読む。
<!--more-->

## target\_examine


`src/target/target.c`
```c
/* Targets that correctly implement init + examine, i.e.
 * no communication with target during init:
 *
 * XScale
 */
int target_examine(void)
{
	int retval = ERROR_OK;
	struct target *target;

	for (target = all_targets; target; target = target->next) {
		/* defer examination, but don't skip it */
		if (!target->tap->enabled) {
			jtag_register_event_callback(jtag_enable_callback,
					target);
			continue;
		}

		if (target->defer_examine)
			continue;

		int retval2 = target_examine_one(target);
		if (retval2 != ERROR_OK) {
			LOG_WARNING("target %s examination failed", target_name(target));
			retval = retval2;
		}
	}
	return retval;
}
/* Equivalent Tcl code arp_examine_one is in src/target/startup.tcl
 * Keep in sync */
int target_examine_one(struct target *target)
{
	target_call_event_callbacks(target, TARGET_EVENT_EXAMINE_START);

	int retval = target->type->examine(target);
	if (retval != ERROR_OK) {
		target_call_event_callbacks(target, TARGET_EVENT_EXAMINE_FAIL);
		return retval;
	}

	target_call_event_callbacks(target, TARGET_EVENT_EXAMINE_END);

	return ERROR_OK;
}

static int jtag_enable_callback(enum jtag_event event, void *priv)
{
	struct target *target = priv;

	if (event != JTAG_TAP_EVENT_ENABLE || !target->tap->enabled)
		return ERROR_OK;

	jtag_unregister_event_callback(jtag_enable_callback, target);

	return target_examine_one(target);
}
```

`target_examine_one`にて、ターゲット固有の`examine`を実行する。
RISC-Vの場合を見ていく。


`src/target/riscv/riscv.c`
```c
struct target_type riscv_target = {
	.name = "riscv",

	.target_create = riscv_create_target,
	.init_target = riscv_init_target,
	.deinit_target = riscv_deinit_target,
	.examine = riscv_examine,

```
riscv用のtarget\_type構造体の`examine`の実態は`riscv_examine`である。
```c
static int riscv_examine(struct target *target)
{
	LOG_DEBUG("riscv_examine()");
	if (target_was_examined(target)) {
		LOG_DEBUG("Target was already examined.");
		return ERROR_OK;
	}

	if (use_vjtag == true) {
		LOG_DEBUG("use_vjtag is enabled");
		riscv_tap_vjtag_init(target->tap);
	}

	/* Don't need to select dbus, since the first thing we do is read dtmcontrol. */

	RISCV_INFO(info);
	uint32_t dtmcontrol = dtmcontrol_scan(target, 0);
	LOG_DEBUG("dtmcontrol=0x%x", dtmcontrol);
	info->dtm_version = get_field(dtmcontrol, DTMCONTROL_VERSION);
	LOG_DEBUG("  version=0x%x", info->dtm_version);

	struct target_type *tt = get_target_type(target);
	if (tt == NULL)
		return ERROR_FAIL;

	int result = tt->init_target(info->cmd_ctx, target);
	if (result != ERROR_OK)
		return result;

	return tt->examine(target);
}
```
ここから、RISC-VのDMに対するアクセスが始まる。

## DMへのアクセス
なお、今回はVJTAGを用いてDMにアクセスをする。
まず最初にアクセスするのは、`dtmcontrol`である。
### dtmcontrol(dtmcs)
`dtmcontrol`には`dtmcontrol_scan`関数を用いてアクセスをする。

`src/target/riscv/riscv.c`
```c
uint8_t ir_dtmcontrol[4] = {DTMCONTROL};
struct scan_field select_dtmcontrol = {
	.in_value = NULL,
	.out_value = ir_dtmcontrol
};
uint8_t ir_dbus[4] = {DBUS};
struct scan_field select_dbus = {
	.in_value = NULL,
	.out_value = ir_dbus
};
```

`src/target/riscv/riscv.c`
```c
static uint32_t dtmcontrol_scan(struct target *target, uint32_t out)
{
	struct scan_field field;
	uint8_t in_value[4];
	uint8_t out_value[4] = { 0 };

	if (bscan_tunnel_ir_width != 0)
		return dtmcontrol_scan_via_bscan(target, out);


	buf_set_u32(out_value, 0, 32, out);

	if (use_vjtag)
		vjtag_vir_scan(target->tap, select_dtmcontrol.out_value[0]);
	else
		jtag_add_ir_scan(target->tap, &select_dtmcontrol, TAP_IDLE);

	field.num_bits = 32;
	field.out_value = out_value;
	field.in_value = in_value;
	jtag_add_dr_scan(target->tap, 1, &field, TAP_IDLE);

	if (use_vjtag)
		vjtag_vir_scan(target->tap, select_dbus.out_value[0]);
	else
		/* Always return to dbus. */
		jtag_add_ir_scan(target->tap, &select_dbus, TAP_IDLE);

	int retval = jtag_execute_queue();
	if (retval != ERROR_OK) {
		LOG_ERROR("failed jtag scan: %d", retval);
		return retval;
	}

	uint32_t in = buf_get_u32(field.in_value, 0, 32);
	LOG_DEBUG("DTMCONTROL: 0x%x -> 0x%x", out, in);

	return in;
}
```
まずは、VIRへDTMCONTROLを転送する。
`jtag_add_dr_scan(target->tap, 1, &field, TAP_IDLE);`
にて、DR scanを行い、DTMCONTROLの値を取得する。(32 bit)
結果は、`field`へ格納される。
なお、終了後のTAP状態はIDLEとなる。
```
	if (use_vjtag)
		vjtag_vir_scan(target->tap, select_dbus.out_value[0]);
```
この行では、DBUS(DMI)へVIRを変更している。
`int retval = jtag_execute_queue();`でJTAGのコマンドを実行する。

`uint32_t in = buf_get_u32(field.in_value, 0, 32);`
この行で、取得した、DTMCONTROLを取り出す。


`dtmcs(dtmcontrol)`のレジスタの構成は以下のようになっている。
{{<figure src="./image00.png" >}}
`version`フィールドを`get_field`にて取り出す。

つぎに、`get_target_type`を行い、ターゲットの種別を調べる。
`src/target/riscv/riscv.c`
```c
static struct target_type *get_target_type(struct target *target)
{
	riscv_info_t *info = (riscv_info_t *) target->arch_info;

	if (!info) {
		LOG_ERROR("Target has not been initialized");
		return NULL;
	}

	switch (info->dtm_version) {
		case 0:
			return &riscv011_target;
		case 1:
			return &riscv013_target;
		default:
			LOG_ERROR("Unsupported DTM version: %d", info->dtm_version);
			return NULL;
	}
}
```
バージョンに応じて、`riscv013_target`, `riscv011_target`を選択する。
今回は、`0b0001`としたので、riscv-debug-spec 0.13 and 1.0に準拠となる。

`riscv-debug/src/dtm.h`
```c
#define DTMCS_VERSION	4'b0001
#define DTMCS_ABITS		6'b100000
```

`src/target/riscv/riscv.c:riscv_examine`
```c
	int result = tt->init_target(info->cmd_ctx, target);
	if (result != ERROR_OK)
		return result;

	return tt->examine(target);
}
```
つぎに、ターゲット(`riscv_013`)固有な初期化を行う。
その後、`examine`をする。

`riscv-013`固有の`target_type`構造体は以下のとおりである。
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

`src/target/riscv/riscv-013.c`
```c
static int init_target(struct command_context *cmd_ctx,
		struct target *target)
{
	LOG_DEBUG("init");
	RISCV_INFO(generic_info);

	generic_info->get_register = &riscv013_get_register;
	generic_info->set_register = &riscv013_set_register;
	generic_info->get_register_buf = &riscv013_get_register_buf;
	generic_info->set_register_buf = &riscv013_set_register_buf;
	generic_info->select_current_hart = &riscv013_select_current_hart;
	generic_info->is_halted = &riscv013_is_halted;
	generic_info->resume_go = &riscv013_resume_go;
	generic_info->step_current_hart = &riscv013_step_current_hart;
	generic_info->on_halt = &riscv013_on_halt;
	generic_info->resume_prep = &riscv013_resume_prep;
	generic_info->halt_prep = &riscv013_halt_prep;
	generic_info->halt_go = &riscv013_halt_go;
	generic_info->on_step = &riscv013_on_step;
	generic_info->halt_reason = &riscv013_halt_reason;
	generic_info->read_debug_buffer = &riscv013_read_debug_buffer;
	generic_info->write_debug_buffer = &riscv013_write_debug_buffer;
	generic_info->execute_debug_buffer = &riscv013_execute_debug_buffer;
	generic_info->fill_dmi_write_u64 = &riscv013_fill_dmi_write_u64;
	generic_info->fill_dmi_read_u64 = &riscv013_fill_dmi_read_u64;
	generic_info->fill_dmi_nop_u64 = &riscv013_fill_dmi_nop_u64;
	generic_info->dmi_write_u64_bits = &riscv013_dmi_write_u64_bits;
	generic_info->authdata_read = &riscv013_authdata_read;
	generic_info->authdata_write = &riscv013_authdata_write;
	generic_info->dmi_read = &dmi_read;
	generic_info->dmi_write = &dmi_write;
	generic_info->read_memory = read_memory;
	generic_info->test_sba_config_reg = &riscv013_test_sba_config_reg;
	generic_info->hart_count = &riscv013_hart_count;
	generic_info->data_bits = &riscv013_data_bits;
	generic_info->print_info = &riscv013_print_info;
	generic_info->version_specific = calloc(1, sizeof(riscv013_info_t));
	if (!generic_info->version_specific)
		return ERROR_FAIL;
	generic_info->sample_memory = sample_memory;
	riscv013_info_t *info = get_info(target);

	info->progbufsize = -1;

	info->dmi_busy_delay = 0;
	info->bus_master_read_delay = 0;
	info->bus_master_write_delay = 0;
	info->ac_busy_delay = 0;

	/* Assume all these abstract commands are supported until we learn
	 * otherwise.
	 * TODO: The spec allows eg. one CSR to be able to be accessed abstractly
	 * while another one isn't. We don't track that this closely here, but in
	 * the future we probably should. */
	info->abstract_read_csr_supported = true;
	info->abstract_write_csr_supported = true;
	info->abstract_read_fpr_supported = true;
	info->abstract_write_fpr_supported = true;

	info->has_aampostincrement = YNM_MAYBE;

	return ERROR_OK;
}
```
`init`では、`riscv013`の処理用の関数を`generic_info`構造体へセットしている。
`generic_info`のRISC-V Debug specのバージョン非依存なインターフェイスを提供する。

今日はここまで。
次回は`examine`を読んで、`examine`の処理が通るようなDMを実装していく。
