# Android Experiments

```shell
sudo dnf groupinstall -y "Development Tools" "Development Libraries"
sudo dnf install -y libtalloc-devel
```

```shell
apk add build-base bsd-compat-headers libarchive-dev linux-headers py3-docutils talloc-dev talloc-static uthash-dev
```

```shell
git clone git@github.com:termux/proot.git
cd proot/src
make LDFLAGS="-static -ltalloc"
```

```diff
diff --git a/src/GNUmakefile b/src/GNUmakefile
index 92a4935..6b89828 100644
--- a/src/GNUmakefile
+++ b/src/GNUmakefile
@@ -14,8 +14,8 @@ STRIP    ?= $(CROSS_COMPILE)strip
 OBJCOPY  ?= $(CROSS_COMPILE)objcopy
 OBJDUMP  ?= $(CROSS_COMPILE)objdump

-CPPFLAGS += -D_FILE_OFFSET_BITS=64 -D_GNU_SOURCE -I. -I$(VPATH)
-CFLAGS   += -Wall -Wextra -O2
+CPPFLAGS += -D_FILE_OFFSET_BITS=64 -D_LARGEFILE64_SOURCE -D_GNU_SOURCE -I. -I$(VPATH)
+CFLAGS   += -D_FILE_OFFSET_BITS=64 -D_LARGEFILE64_SOURCE -D_GNU_SOURCE -Wall -Wextra -O2
 LDFLAGS  += -ltalloc -Wl,-z,noexecstack

 OBJECTS += \
@@ -55,7 +55,6 @@ OBJECTS += \
 	ptrace/user.o		\
 	ptrace/wait.o		\
 	extension/extension.o	\
-	extension/ashmem_memfd/ashmem_memfd.o \
 	extension/kompat/kompat.o \
 	extension/fake_id0/chown.o \
 	extension/fake_id0/chroot.o \
diff --git a/src/cli/proot.c b/src/cli/proot.c
index 29365e8..92178b7 100644
--- a/src/cli/proot.c
+++ b/src/cli/proot.c
@@ -293,18 +293,6 @@ static int handle_option_link2symlink(Tracee *tracee, const Cli *cli UNUSED, con
 	return 0;
 }

-static int handle_option_ashmem_memfd(Tracee *tracee, const Cli *cli UNUSED, const char *value UNUSED)
-{
-	int status;
-
-	/* Initialize the ashmem-memfd extension.  */
-	status = initialize_extension(tracee, ashmem_memfd_callback, NULL);
-	if (status < 0)
-		note(tracee, WARNING, INTERNAL, "ashmem-memfd not initialized");
-
-	return 0;
-}
-
 static int handle_option_sysvipc(Tracee *tracee, const Cli *cli UNUSED, const char *value UNUSED)
 {
 	int status;
diff --git a/src/cli/proot.h b/src/cli/proot.h
index 3f61931..95cca0b 100644
--- a/src/cli/proot.h
+++ b/src/cli/proot.h
@@ -61,7 +61,6 @@ static int handle_option_i(Tracee *tracee, const Cli *cli, const char *value);
 static int handle_option_R(Tracee *tracee, const Cli *cli, const char *value);
 static int handle_option_S(Tracee *tracee, const Cli *cli, const char *value);
 static int handle_option_link2symlink(Tracee *tracee, const Cli *cli, const char *value);
-static int handle_option_ashmem_memfd(Tracee *tracee, const Cli *cli, const char *value);
 static int handle_option_sysvipc(Tracee *tracee, const Cli *cli, const char *value);
 static int handle_option_kill_on_exit(Tracee *tracee, const Cli *cli, const char *value);
 static int handle_option_L(Tracee *tracee, const Cli *cli, const char *value);
@@ -252,14 +251,6 @@ Copyright (C) 2015 STMicroelectronics, licensed under GPL v2 or later.",
 \tsyscalls inside proot. IPC is handled inside proot and launching 2 proot instances\n\
 \twill lead to 2 different IPC Namespaces",
 	},
-	{ .class = "Extension options",
-	  .arguments = {
-		{ .name = "--ashmem-memfd", .separator = '\0', .value = NULL },
-		{ .name = NULL, .separator = '\0', .value = NULL } },
-          .handler = handle_option_ashmem_memfd,
-          .description = "Emulate memfd_create support through ashmem and simulate fstat.st_size for ashmem",
-          .detail = "",
-	},
         { .class = "Extension options",
           .arguments = {
                 { .name = "-H", .separator = '\0', .value = NULL },
diff --git a/src/extension/ashmem_memfd/ashmem_memfd.c b/src/extension/ashmem_memfd/ashmem_memfd.c
deleted file mode 100644
index df3f3b5..0000000
--- a/src/extension/ashmem_memfd/ashmem_memfd.c
+++ /dev/null
@@ -1,238 +0,0 @@
-#include <stdlib.h>
-#include <signal.h>
-#include <unistd.h>
-#include <sys/syscall.h>  /* __NR_memfd_create,  */
-#include <linux/ashmem.h> /* ASHMEM_GET_SIZE,  */
-#include <linux/memfd.h>  /* MFD_CLOEXEC  */
-
-#include <talloc.h>
-
-#include "extension/extension.h"
-#include "path/path.h"
-#include "tracee/mem.h"
-#include "tracee/seccomp.h"
-#include "syscall/chain.h"
-#include "syscall/syscall.h" /* set_sysarg_data,  */
-
-enum AshmemMemfdChainState {
-	CS_IDLE,
-	CS_STAT_ENTERED,
-	CS_STAT_CHAINED_IOCTL
-};
-
-typedef struct {
-	bool memfd_supported;
-	enum AshmemMemfdChainState chain_state;
-	int fd;
-	word_t addr;
-} AshmemMemfdState;
-
-static FilteredSysnum filtered_sysnums[] = {
-	{ PR_memfd_create,		0 },
-	{ PR_ftruncate,		0 },
-	{ PR_fstat,		0 },
-	FILTERED_SYSNUM_END,
-};
-
-static int detect_memfd_support() {
-	const char *assume_unsupported = getenv("PROOT_ASSUME_MEMFD_UNSUPPORTED");
-	if (assume_unsupported != NULL) {
-		if (0 == strcmp(assume_unsupported, "1")) {
-			return 0;
-		}
-		if (0 == strcmp(assume_unsupported, "0")) {
-			return 1;
-		}
-	}
-
-	int reply_pipe[2];
-	int status = pipe(reply_pipe);
-	if (status < 0) {
-		return -1;
-	}
-
-	status = fork();
-	if (status < 0) {
-		close(reply_pipe[0]);
-		close(reply_pipe[1]);
-		return -1;
-	}
-
-	if (status == 0) {
-		/** Child process. Close readable end of pipe.  */
-		close(reply_pipe[0]);
-
-		/** Attempt creating memfd.  */
-		signal(SIGSYS, SIG_DFL);
-		int memfd = syscall(__NR_memfd_create, "support_probe", 0);
-
-		/** Send message to parent on success.  */
-		if (memfd >= 0) {
-			write(reply_pipe[1], "\x01", 1);
-			close(memfd);
-		}
-		close(reply_pipe[1]);
-		_exit(0);
-	}
-
-	/** Parent process.  */
-	close(reply_pipe[1]);
-	char reply_value = 0;
-	read(reply_pipe[0], &reply_value, 1);
-	close(reply_pipe[0]);
-	return reply_value == 1;
-}
-
-static bool is_ashmem_fd(Tracee *tracee, int fd) {
-	char path[PATH_MAX] = {};
-	if (readlink_proc_pid_fd(tracee->pid, fd, path) < 0) {
-		return false;
-	}
-	return 0 == strcmp(path, "/dev/ashmem");
-}
-
-static void ashmem_memfd_handle_stat(Extension *extension, Tracee *tracee, int fd, Reg stat_reg) {
-	if (is_ashmem_fd(tracee, fd)) {
-		AshmemMemfdState *state = talloc_get_type_abort(extension->config, AshmemMemfdState);
-		state->chain_state = CS_STAT_ENTERED;
-		state->fd = fd;
-		state->addr = peek_reg(tracee, CURRENT, stat_reg) + offsetof(struct stat, st_size);
-		tracee->restart_how = PTRACE_SYSCALL;
-	}
-}
-
-static int ashmem_memfd_handle_memfd_create(Extension *extension, Tracee *tracee, bool from_sigsys) {
-	AshmemMemfdState *state = talloc_get_type_abort(extension->config, AshmemMemfdState);
-	if (!state->memfd_supported) {
-		word_t flags = peek_reg(tracee, CURRENT, SYSARG_2);
-		set_sysnum(tracee, PR_openat);
-		set_sysarg_data(tracee, "/dev/ashmem", 12, SYSARG_2);
-		poke_reg(tracee, SYSARG_1, AT_FDCWD);
-		poke_reg(tracee, SYSARG_3, O_RDWR | ((flags & MFD_CLOEXEC) ? O_CLOEXEC : 0));
-		poke_reg(tracee, SYSARG_4, 0);
-		if (from_sigsys) {
-			restart_syscall_after_seccomp(tracee);
-			/* Skip further processing (such as forcing syscall result) from SIGSYS handler.  */
-			return 2;
-		}
-	}
-	return 0;
-}
-
-static void ashmem_memfd_handle_syscall(Extension *extension) {
-	Tracee *tracee = TRACEE(extension);
-	switch (get_sysnum(tracee, CURRENT)) {
-	case PR_memfd_create:
-	{
-		ashmem_memfd_handle_memfd_create(extension, tracee, false);
-		break;
-	}
-	case PR_ftruncate:
-	case PR_ftruncate64:
-	{
-		int fd = peek_reg(tracee, CURRENT, SYSARG_1);
-		if (is_ashmem_fd(tracee, fd)) {
-			set_sysnum(tracee, PR_ioctl);
-			poke_reg(tracee, SYSARG_3, peek_reg(tracee, CURRENT, SYSARG_2));
-			poke_reg(tracee, SYSARG_2, ASHMEM_SET_SIZE);
-		}
-	}
-	case PR_fstat:
-		ashmem_memfd_handle_stat(extension, tracee, peek_reg(tracee, CURRENT, SYSARG_1), SYSARG_2);
-		break;
-	case PR_fstatat64:
-	{
-		int fd = peek_reg(tracee, CURRENT, SYSARG_1);
-		if (
-				fd >= 0 &&
-				/** Is path argument an empty string?  */
-				!(peek_word(tracee, peek_reg(tracee, CURRENT, SYSARG_2)) & 0xFF)
-		   ) {
-			ashmem_memfd_handle_stat(extension, tracee, fd, SYSARG_3);
-		}
-		break;
-	}
-	default:
-		break;
-	}
-}
-
-static void ashmem_memfd_handle_stat_exit(Tracee *tracee, AshmemMemfdState *state) {
-	if (peek_reg(tracee, CURRENT, SYSARG_RESULT) || peek_word(tracee, state->addr)) {
-		state->chain_state = CS_IDLE;
-		return;
-	}
-
-	register_chained_syscall(tracee, PR_ioctl, state->fd, ASHMEM_GET_SIZE, 0, 0, 0, 0);
-	state->chain_state = CS_STAT_CHAINED_IOCTL;
-}
-
-int ashmem_memfd_callback(Extension *extension, ExtensionEvent event, intptr_t data1, intptr_t data2)
-{
-	switch (event) {
-	case INITIALIZATION: {
-		extension->config = talloc_zero(extension, AshmemMemfdState);
-		AshmemMemfdState *state = talloc_get_type_abort(extension->config, AshmemMemfdState);
-
-		memset(state, 0, sizeof(*state));
-		state->memfd_supported = detect_memfd_support();
-
-		extension->filtered_sysnums = filtered_sysnums;
-
-		return 0;
-	}
-	case INHERIT_PARENT: /* Inheritable for sub reconfiguration ...  */
-		return 1;
-
-	case INHERIT_CHILD: {
-		/* Create configuration in child.  */
-		Extension *parent = (Extension *) data1;
-		extension->config = talloc_zero(extension, AshmemMemfdState);
-		if (extension->config == NULL)
-			return -1;
-
-		AshmemMemfdState *old_state = talloc_get_type_abort(parent->config, AshmemMemfdState);
-		AshmemMemfdState *state = talloc_get_type_abort(extension->config, AshmemMemfdState);
-		state->memfd_supported = old_state->memfd_supported;
-	}
-
-	case SYSCALL_ENTER_END:
-		ashmem_memfd_handle_syscall(extension);
-		return 0;
-
-	case SYSCALL_EXIT_START:
-	{
-		AshmemMemfdState *state = talloc_get_type_abort(extension->config, AshmemMemfdState);
-		switch (state->chain_state) {
-			case CS_IDLE:
-			case CS_STAT_CHAINED_IOCTL:
-				break;
-			case CS_STAT_ENTERED:
-				ashmem_memfd_handle_stat_exit(TRACEE(extension), state);
-				break;
-		}
-		return 0;
-	}
-	case SIGSYS_OCC:
-	{
-		Tracee *tracee = TRACEE(extension);
-		if (get_sysnum(tracee, CURRENT) == PR_memfd_create) {
-			return ashmem_memfd_handle_memfd_create(extension, tracee, true);
-		}
-		return 0;
-	}
-	case SYSCALL_CHAINED_EXIT:
-	{
-		AshmemMemfdState *state = talloc_get_type_abort(extension->config, AshmemMemfdState);
-		if (state->chain_state == CS_STAT_CHAINED_IOCTL) {
-			state->chain_state = CS_IDLE;
-			Tracee *tracee = TRACEE(extension);
-			poke_uint32(tracee, state->addr, peek_reg(tracee, CURRENT, SYSARG_RESULT));
-			poke_reg(tracee, SYSARG_RESULT, 0);
-		}
-		return 0;
-	}
-	default:
-		return 0;
-	}
-}
diff --git a/src/extension/extension.h b/src/extension/extension.h
index 24409b9..27c0bcd 100644
--- a/src/extension/extension.h
+++ b/src/extension/extension.h
@@ -205,7 +205,6 @@ extern int hidden_files_callback(Extension *extension, ExtensionEvent event, int
 extern int port_switch_callback(Extension *extension, ExtensionEvent event, intptr_t d1, intptr_t d2);
 extern int link2symlink_callback(Extension *extension, ExtensionEvent event, intptr_t d1, intptr_t d2);
 extern int fix_symlink_size_callback(Extension *extension, ExtensionEvent event, intptr_t d1, intptr_t d2);
-extern int ashmem_memfd_callback(Extension *extension, ExtensionEvent event, intptr_t d1, intptr_t d2);
 extern int mountinfo_callback(Extension *extension, ExtensionEvent event, intptr_t d1, intptr_t d2);

 #endif /* EXTENSION_H */
diff --git a/src/extension/sysvipc/sysvipc_msg.c b/src/extension/sysvipc/sysvipc_msg.c
index 1fc290a..75c6ca8 100644
--- a/src/extension/sysvipc/sysvipc_msg.c
+++ b/src/extension/sysvipc/sysvipc_msg.c
@@ -175,37 +175,17 @@ static int sysvipc_do_msgrcv(Tracee *tracee, struct SysVIpcConfig *config, size_
 		return -EINVAL;
 	}

-	if ((config->msgrcv_msgflg & ~(IPC_NOWAIT | MSG_NOERROR | MSG_COPY | MSG_EXCEPT)) != 0) {
+	if ((config->msgrcv_msgflg & ~(IPC_NOWAIT | MSG_NOERROR | MSG_EXCEPT)) != 0) {
 		return -EINVAL;
 	}

-	bool copy = (config->msgrcv_msgflg & MSG_COPY) != 0;
-	if (copy) {
-		if ((config->msgrcv_msgflg & IPC_NOWAIT) == 0) {
-			return -EINVAL;
-		}
-		if ((config->msgrcv_msgflg & MSG_EXCEPT) != 0) {
-			return -EINVAL;
-		}
-	}
-
 	struct SysVIpcMsgQueueItem *found_item = NULL;
 	struct SysVIpcMsgQueueItem *candidate_item;
-	if (copy) {
-		int index = 0;
-		STAILQ_FOREACH(candidate_item, queue->items, link) {
-			if (index == config->msgrcv_msgtyp) {
-				found_item = candidate_item;
-				break;
-			}
-			index += 1;
-		}
-	} else {
-		STAILQ_FOREACH(candidate_item, queue->items, link) {
-			if (sysvipc_msg_match(candidate_item->mtype, config->msgrcv_msgtyp, config->msgrcv_msgflg)) {
-				found_item = candidate_item;
-				break;
-			}
+
+	STAILQ_FOREACH(candidate_item, queue->items, link) {
+		if (sysvipc_msg_match(candidate_item->mtype, config->msgrcv_msgtyp, config->msgrcv_msgflg)) {
+			found_item = candidate_item;
+			break;
 		}
 	}

@@ -224,7 +204,7 @@ static int sysvipc_do_msgrcv(Tracee *tracee, struct SysVIpcConfig *config, size_

 	int status = sysvipc_msg_deliver(tracee, config, queue, found_item, current_time);

-	if (status >= 0 && !copy) {
+	if (status >= 0) {
 		queue->stats.msg_qnum -= 1;
 		queue->stats.msg_cbytes -= found_item->mtext_length;
 		STAILQ_REMOVE(queue->items, found_item, SysVIpcMsgQueueItem, link);
diff --git a/src/extension/sysvipc/sysvipc_shm.c b/src/extension/sysvipc/sysvipc_shm.c
index 6cb56e1..c3cc9b2 100644
--- a/src/extension/sysvipc/sysvipc_shm.c
+++ b/src/extension/sysvipc/sysvipc_shm.c
@@ -684,7 +684,7 @@ void sysvipc_shm_helper_main() {
 	write(1, addr.sun_path, SYSVIPC_SHMHELPER_SOCKET_LEN);
 	for (;;) {
 		struct SysVIpcShmHelperRequest request;
-		int status = TEMP_FAILURE_RETRY(read(0, &request, sizeof(request)));
+		int status = read(0, &request, sizeof(request));
 		if (status == 0) {
 			break;
 		}
```

```shell
sudo install ./proot /usr/local/bin
```

```shell
curl -Lo alpine-proot.sh git.io/alpine-proot.sh
rm -rf ~/.alpinelinux_container
./alpine-proot.sh

# Doesn't work properly on `glibc` hosts just yet:

# proot warning: signal 4 received from process 4198418
# ######################################################################################## 100.0%
# (loops)

# We should probably follow https://git.alpinelinux.org/aports/tree/testing/proot/APKBUILD to build instead with upstream proot
```
