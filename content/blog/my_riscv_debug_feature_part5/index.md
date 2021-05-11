---
title: "My RISC-V debug feature part5"
date: 2021-05-10T22:21:40+09:00
author: "@koyamanX"
categories: ["RISC-V"]
tags: ["RISC-V", "FPGA", "JTAG"]
draft: false
---

仕様を読んでいるが、よくわからない。
hartselとかwindowみたいなのがあってよく分かりづらい。
最小限の機能で、OpenOCDから制御できればいいので、
OpenOCDの初期化フローを読んで、それに応答するようなDM(Debug Module)を作ってみる。
<!--more-->

## OpenOCDの初期化

`src/openocd.c`
```c
/* normally this is the main() function entry, but if OpenOCD is linked
 * into application, then this fn will not be invoked, but rather that
 * application will have it's own implementation of main(). */
int openocd_main(int argc, char *argv[])
{
	int ret;

	/* initialize commandline interface */
	struct command_context *cmd_ctx;

	cmd_ctx = setup_command_handler(NULL);

	if (util_init(cmd_ctx) != ERROR_OK)
		return EXIT_FAILURE;

	if (ioutil_init(cmd_ctx) != ERROR_OK)
		return EXIT_FAILURE;

	if (rtt_init() != ERROR_OK)
		return EXIT_FAILURE;

	LOG_OUTPUT("For bug reports, read\n\t"
		"http://openocd.org/doc/doxygen/bugs.html"
		"\n");

	command_context_mode(cmd_ctx, COMMAND_CONFIG);
	command_set_output_handler(cmd_ctx, configuration_output_handler, NULL);

	server_host_os_entry();

	/* Start the executable meat that can evolve into thread in future. */
	ret = openocd_thread(argc, argv, cmd_ctx);

	flash_free_all_banks();
	gdb_service_free();
	server_free();

	unregister_all_commands(cmd_ctx, NULL);

	/* free all DAP and CTI objects */
	dap_cleanup_all();
	arm_cti_cleanup_all();

	adapter_quit();

	server_host_os_close();

	/* Shutdown commandline interface */
	command_exit(cmd_ctx);

	rtt_exit();
	free_config();

	if (ERROR_FAIL == ret)
		return EXIT_FAILURE;
	else if (ERROR_OK != ret)
		exit_on_signal(ret);

	return ret;
}
```
```c
/** OpenOCD runtime meat that can become single-thread in future. It parse
 * commandline, reads configuration, sets up the target and starts server loop.
 * Commandline arguments are passed into this function from openocd_main().
 */
static int openocd_thread(int argc, char *argv[], struct command_context *cmd_ctx)
{
	int ret;

	if (parse_cmdline_args(cmd_ctx, argc, argv) != ERROR_OK)
		return ERROR_FAIL;

	if (server_preinit() != ERROR_OK)
		return ERROR_FAIL;

	ret = parse_config_file(cmd_ctx);
	if (ret == ERROR_COMMAND_CLOSE_CONNECTION) {
		server_quit(); /* gdb server may be initialized by -c init */
		return ERROR_OK;
	} else if (ret != ERROR_OK) {
		server_quit(); /* gdb server may be initialized by -c init */
		return ERROR_FAIL;
	}

	ret = server_init(cmd_ctx);
	if (ERROR_OK != ret)
		return ERROR_FAIL;

	if (init_at_startup) {
		ret = command_run_line(cmd_ctx, "init");
		if (ERROR_OK != ret) {
			server_quit();
			return ERROR_FAIL;
		}
	}

	ret = server_loop(cmd_ctx);

	int last_signal = server_quit();
	if (last_signal != ERROR_OK)
		return last_signal;

	if (ret != ERROR_OK)
		return ERROR_FAIL;
	return ERROR_OK;
}
```
`openocd_thread`がメインになる様子。
まず、command line argumentsをパースする。
その後、コンフィグファイルをパースする。
このコンフィグファイルは`-f`オプションで渡されるものであり、コマンドを一行ずつ実行していく。

## コンフィグファイルのパースおよび実行

`src/helper/configuration.c`
```
int parse_config_file(struct command_context *cmd_ctx)
{
	int retval;
	char **cfg;

	if (!config_file_names) {
		command_run_line(cmd_ctx, "script openocd.cfg");
		return ERROR_OK;
	}

	cfg = config_file_names;

	while (*cfg) {
		retval = command_run_line(cmd_ctx, *cfg);
		if (retval != ERROR_OK)
			return retval;
		cfg++;
	}

	return ERROR_OK;
}
```

`tcl/target/rv32xsoc.cfg`の抜粋。
```bash
...
jtag newtap $_CHIPNAME cpu -irlen 10 -expected-id $_FPGATAPID
target create $_TARGETNAME riscv -endian $_ENDIAN -chain-position $_TARGETNAME
```
`parse_config_file`によりパースされ、一行ずつ実行する。
ここでは、`target_create`にフォーカスしてみていく。
`target create`のハンドラは以下のようになっている。

`src/target/target.c`
```c
static const struct command_registration target_command_handlers[] = {
	{
		.name = "targets",
		.handler = handle_targets_command,
		.mode = COMMAND_ANY,
		.help = "change current default target (one parameter) "
			"or prints table of all targets (no parameters)",
		.usage = "[target]",
	},
	{
		.name = "target",
		.mode = COMMAND_CONFIG,
		.help = "configure target",
		.chain = target_subcommand_handlers,
		.usage = "",
	},
	{
		.name = "enable_rtos_riscv",
		.handler = handle_enable_rtos_riscv_command,
		.mode = COMMAND_CONFIG,
		.usage = "enable_rtos_riscv",
		.help = "Allow the use of `-rtos riscv` for just a little longer, "
			"until it will be completely removed at the end of 2020."
	},
	COMMAND_REGISTRATION_DONE
};
```

つまり、`target`命令の場合は次にくる引数がサブコマンドになる。
サブコマンドは以下の通り。

`src/target/target.c`
```c
static const struct command_registration target_subcommand_handlers[] = {
	{
		.name = "init",
		.mode = COMMAND_CONFIG,
		.handler = handle_target_init_command,
		.help = "initialize targets",
		.usage = "",
	},
	{
		.name = "create",
		.mode = COMMAND_CONFIG,
		.jim_handler = jim_target_create,
		.usage = "name type '-chain-position' name [options ...]",
		.help = "Creates and selects a new target",
	},
	{
		.name = "current",
		.mode = COMMAND_ANY,
		.jim_handler = jim_target_current,
		.help = "Returns the currently selected target",
	},
	{
		.name = "types", .mode = COMMAND_ANY, .jim_handler = jim_target_types, .help = "Returns the available target types as " "a list of strings", }, {
		.name = "names",
		.mode = COMMAND_ANY,
		.jim_handler = jim_target_names,
		.help = "Returns the names of all targets as a list of strings",
	},
	{
		.name = "smp",
		.mode = COMMAND_ANY,
		.jim_handler = jim_target_smp,
		.usage = "targetname1 targetname2 ...",
		.help = "gather several target in a smp list"
	},

	COMMAND_REGISTRATION_DONE
};
```

今回は`target create`なので、`jim_target_create`関数が呼ばれることになる。
この関数では、命令の引数を準備し、`target_create`を呼び出す。
`src/target/target.c`
```c
static int jim_target_create(Jim_Interp *interp, int argc, Jim_Obj *const *argv)
{
	Jim_GetOptInfo goi;
	Jim_GetOpt_Setup(&goi, interp, argc - 1, argv + 1);
	if (goi.argc < 3) {
		Jim_WrongNumArgs(goi.interp, goi.argc, goi.argv,
			"<name> <target_type> [<target_options> ...]");
		return JIM_ERR;
	}
	return target_create(&goi);
}
```


`target create`の引数を処理する。
`src/target/target.c:create_target`
```c
static int target_create(Jim_GetOptInfo *goi)
{
	Jim_Obj *new_cmd;
	Jim_Cmd *cmd;
	const char *cp;
	int e;
	int x;
	struct target *target;
	struct command_context *cmd_ctx;

	cmd_ctx = current_command_context(goi->interp);
	assert(cmd_ctx != NULL);

	if (goi->argc < 3) {
		Jim_WrongNumArgs(goi->interp, 1, goi->argv, "?name? ?type? ..options...");
		return JIM_ERR;
	}

...
```


`src/target/target.c:create_target`続き
```c
	/* now does target type exist */
	for (x = 0 ; target_types[x] ; x++) {
		if (0 == strcmp(cp, target_types[x]->name)) {
			/* found */
			break;
		}

		/* check for deprecated name */
		if (target_types[x]->deprecated_name) {
			if (0 == strcmp(cp, target_types[x]->deprecated_name)) {
				/* found */
				LOG_WARNING("target name is deprecated use: \'%s\'", target_types[x]->name);
				break;
			}
		}
	}
```
この部分で、以下のtclのriscvを処理している。
`target create $_TARGETNAME riscv -endian $_ENDIAN -chain-position $_TARGETNAME`
`target_types[]`から`riscv`に対応する、ターゲットを探し出す。

`src/target/target.c`
```c
static struct target_type *target_types[] = {
	&arm7tdmi_target,
	&arm9tdmi_target,
	&arm920t_target,
	&arm720t_target,
	...
	&riscv_target,
	...	
```

`src/target/target.c:target_create`続き
```c
	/* Create it */
	target = calloc(1, sizeof(struct target));
	if (!target) {
		LOG_ERROR("Out of memory");
		return JIM_ERR;
	}

	/* set target number */
	target->target_number = new_target_number();

	/* allocate memory for each unique target type */
	target->type = malloc(sizeof(struct target_type));
	if (!target->type) {
		LOG_ERROR("Out of memory");
		free(target);
		return JIM_ERR;
	}

	memcpy(target->type, target_types[x], sizeof(struct target_type));
```

`target_configure`関数で`-endian`や`-chain-position`を処理をする。

`src/target/target.c:target_create`続き
```c
	e = target_configure(goi, target);

	if (e == JIM_OK) {
		if (target->has_dap) {
			if (!target->dap_configured) {
				Jim_SetResultString(goi->interp, "-dap ?name? required when creating target", -1);
				e = JIM_ERR;
			}
		} else {
			if (!target->tap_configured) {
				Jim_SetResultString(goi->interp, "-chain-position ?name? required when creating target", -1);
				e = JIM_ERR;
			}
		}
		/* tap must be set after target was configured */
		if (target->tap == NULL)
			e = JIM_ERR;
	}
```
また、この時点では(`target_create`)、TAPが設定されていることが期待されている(`-chain-position`で指定)。上のコードで確認している。

`src/target/target.c:target_create`続き
```c
	if (target->type->target_create) {
		e = (*(target->type->target_create))(target, goi->interp);
		if (e != ERROR_OK) {
			LOG_DEBUG("target_create failed");
			free(target->cmd_name);
			rtos_destroy(target);
			free(target->gdb_port_override);
			free(target->trace_info);
			free(target->type);
			free(target);
			return JIM_ERR;
		}
	}

	/* create the target specific commands */
	if (target->type->commands) {
		e = register_commands(cmd_ctx, NULL, target->type->commands);
		if (ERROR_OK != e)
			LOG_ERROR("unable to register '%s' commands", cp);
	}
```
ここで、ターゲット(riscv)の`target_create`を呼び出す。
その後、ターゲットspecificなコマンドを登録する。

riscv用の`target_create`を読んで見る。

以下が`target_types`から取り出した、ターゲットspecificな構造体の実態である。
`../src/target/riscv/riscv.c`
```c
struct target_type riscv_target = {
	.name = "riscv",

	.target_create = riscv_create_target,
	.init_target = riscv_init_target,
	.deinit_target = riscv_deinit_target,
	.examine = riscv_examine,
```

`riscv_create_target`が呼び出されることになる。
`../src/target/riscv/riscv.c`
```c
static int riscv_create_target(struct target *target, Jim_Interp *interp)
{
	LOG_DEBUG("riscv_create_target()");
	target->arch_info = calloc(1, sizeof(riscv_info_t));
	if (!target->arch_info)
		return ERROR_FAIL;
	riscv_info_init(target, target->arch_info);
	return ERROR_OK;
}
```

`../src/target/riscv/riscv.c`
```c
/*** RISC-V Interface ***/

void riscv_info_init(struct target *target, riscv_info_t *r)
{
	memset(r, 0, sizeof(*r));
	r->dtm_version = 1;
	r->registers_initialized = false;
	r->current_hartid = target->coreid;
	r->version_specific = NULL;

	memset(r->trigger_unique_id, 0xff, sizeof(r->trigger_unique_id));

	r->xlen = -1;

	r->mem_access_methods[0] = RISCV_MEM_ACCESS_PROGBUF;
	r->mem_access_methods[1] = RISCV_MEM_ACCESS_SYSBUS;
	r->mem_access_methods[2] = RISCV_MEM_ACCESS_ABSTRACT;

	r->mem_access_progbuf_warn = true;
	r->mem_access_sysbus_warn = true;
	r->mem_access_abstract_warn = true;

	INIT_LIST_HEAD(&r->expose_csr);
	INIT_LIST_HEAD(&r->expose_custom);
}
```


## initコマンド

ターゲットの作成が終わったら、サーバーを建てる。
そのあと、`init`コマンドを実行する。

`src/openocd.c:openocd_thread`抜粋
```c
	ret = server_init(cmd_ctx);
	if (ERROR_OK != ret)
		return ERROR_FAIL;

	if (init_at_startup) {
		ret = command_run_line(cmd_ctx, "init");
		if (ERROR_OK != ret) {
			server_quit();
			return ERROR_FAIL;
		}
	}

```

`src/openocd.c`
```c
static const struct command_registration openocd_command_handlers[] = {
	{
		.name = "version",
		.jim_handler = jim_version_command,
		.mode = COMMAND_ANY,
		.help = "show program version",
	},
	{
		.name = "noinit",
		.handler = &handle_noinit_command,
		.mode = COMMAND_CONFIG,
		.help = "Prevent 'init' from being called at startup.",
		.usage = ""
	},
	{
		.name = "init",
		.handler = &handle_init_command,
		.mode = COMMAND_ANY,
		.help = "Initializes configured targets and servers.  "
			"Changes command mode from CONFIG to EXEC.  "
			"Unless 'noinit' is called, this command is "
			"called automatically at the end of startup.",
		.usage = ""
	},
	{
		.name = "add_script_search_dir",
		.handler = &handle_add_script_search_dir_command,
		.mode = COMMAND_ANY,
		.help = "dir to search for config files and scripts",
		.usage = "<directory>"
	},
	COMMAND_REGISTRATION_DONE
};
```
`init`コマンドは`handle_init_command`が対処する。


`src/openocd.c`
```c
/* OpenOCD can't really handle failure of this command. Patches welcome! :-) */
COMMAND_HANDLER(handle_init_command)
{

	if (CMD_ARGC != 0)
		return ERROR_COMMAND_SYNTAX_ERROR;

	int retval;
	static int initialized;
	if (initialized)
		return ERROR_OK;

	initialized = 1;

	retval = command_run_line(CMD_CTX, "target init");
	if (ERROR_OK != retval)
		return ERROR_FAIL;

	retval = adapter_init(CMD_CTX);
	if (retval != ERROR_OK) {
		/* we must be able to set up the debug adapter */
		return retval;
	}

	LOG_DEBUG("Debug Adapter init complete");

	/* "transport init" verifies the expected devices are present;
	 * for JTAG, it checks the list of configured TAPs against
	 * what's discoverable, possibly with help from the platform's
	 * JTAG event handlers.  (which require COMMAND_EXEC)
	 */
	command_context_mode(CMD_CTX, COMMAND_EXEC);

	retval = command_run_line(CMD_CTX, "transport init");
	if (ERROR_OK != retval)
		return ERROR_FAIL;

	retval = command_run_line(CMD_CTX, "dap init");
	if (ERROR_OK != retval)
		return ERROR_FAIL;

	LOG_DEBUG("Examining targets...");
	if (target_examine() != ERROR_OK)
		LOG_DEBUG("target examination failed");

	command_context_mode(CMD_CTX, COMMAND_CONFIG);

	if (command_run_line(CMD_CTX, "flash init") != ERROR_OK)
		return ERROR_FAIL;

	if (command_run_line(CMD_CTX, "nand init") != ERROR_OK)
		return ERROR_FAIL;

	if (command_run_line(CMD_CTX, "pld init") != ERROR_OK)
		return ERROR_FAIL;
	command_context_mode(CMD_CTX, COMMAND_EXEC);

	/* initialize telnet subsystem */
	gdb_target_add_all(all_targets);

	target_register_event_callback(log_target_callback_event_handler, CMD_CTX);

	return ERROR_OK;
}
```
ここでは、以下の順でコマンドと関数を実行する。
0. `init`
0. `adapter_init();`
0. `transport init`
0. `dap init`
0. `target_examine();`
0. `flash init`
0. `nand init`
0. `pld init`

`init`では、`riscv`固有の初期化をする。
`adapter_init()`, `transport init`, `dap init`でJTAGの初期化をしている？
`target_examine`では、`RISC-V Debug DM`が存在し、正しいステータスを返すかテストしているっぽい。
次は、これらを詳しくみていく。
なお、`flash init`、`nand init`、`pld init`は見なくて良さそうなのでとりあえず放置する。
