---
title: "How Can MicroPython Run Like A OS?"
date: 2022-04-16 09:48:10
tags:
---

在本文中记录我们以 MicroPython 和 REPL 的交互为例，探索 MicroPython 为什么能和一个 OS 一样做到被事件驱动，而非顺序执行。

<!-- more -->

## 现象

- 在 esp32 上执行如下代码
    
```c
import webrepl
webrepl.start()
while True:
    print("loop")
```
    
后，使用 webrepl 客户端进行连接，无法在板上 serial port 和 webrepl console 中看到任何相关内容
    
- 在执行完 `webrepl.start()` 后，没有开始循环时，使用 webrepl 客户端进行连接，连接成功后，再开始循环，`loop` 会输出到 serial port 和 webrepl console 中
- 如果执行的是这样的代码
    
    ```c
    import webrepl
    import time
    webrepl.start()
    while True:
        print("loop")
        time.sleep_ms(1000)
    ```
    
    则即使在循环开始以后，也能够通过 webrepl 连接上
    
- 在循环开始后，使用 serial 连接的终端模拟器可以直接通过 `Ctrl + C` 而 webrepl console 中就无法做到
- 在加入了 `time.sleep_ms(1000)` 的代码被加入后，在 webrepl 中也能够使用 `Ctrl + C` 来将循环打断


## 一些前期猜想

- MicroPython 是直接跑在 MicroPython 上的，通过中断进行一切事件驱动（无法解释死循环中无法响应 webrepl 请求）
- MicroPython 是跑在 RTOS 上的，通过 RTOS 的线程级调度来实现并发，这样也能够做到事件驱动（还是无法解释死循环中无法响应 webrepl 请求）
- MicroPython 是个单线程程序，直接由自身通过轮询去处理所有的事件（这样的确可以在要输出的时候去查看所有连接上的 console，但是能输出不能接收又显得太过古怪，至少在输出的时候可以通过查看所有事件来发现要接收的信号）

到这里我们已经觉得把 MicroPython 当成黑箱去分析得出的结果可能非常不靠谱，所以我们从源码入手，尝试解释我们实验中遇到的现象。

## 实际实现

我们烧录进去的二进制文件包括了 RTOS 和 MicroPython，它们一共使用三种方式来实现事件驱动：

### Baremetal 进行中断响应
通过设置片上中断来进行中断响应，达到打断程序的目的。这样可以非常自由地进行操作，例如收到了 uart 的输入后，可以直接将这些输入送给 MicroPython 的 stream。这里的响应是最底层的。在这一层，MCU 和我们使用的计算机十分相似，我们完全可以用理解 PC 中断的方式来理解 esp32 的中断。
    
在 MicroPython 最先开始在 FreeRTOS 上运行的时候，进行了一些初始化操作，其中进行了中断设置。我们发现

```c
void app_main(void) {
    // Hook for a board to run code at start up.
    // This defaults to initialising NVS.
    MICROPY_BOARD_STARTUP();

    // Create and transfer control to the MicroPython task.
    xTaskCreatePinnedToCore(mp_task, "mp_task", MP_TASK_STACK_SIZE / sizeof(StackType_t), NULL, MP_TASK_PRIORITY, &mp_main_task_handle, MP_TASK_COREID);
}
```

函数启动了主线程函数 `mp_task`，在其开始运行的时候

```c
void mp_task(void *pvParameter) {
    volatile uint32_t sp = (uint32_t)get_sp();
    #if MICROPY_PY_THREAD
    mp_thread_init(pxTaskGetStackStart(NULL), MP_TASK_STACK_SIZE / sizeof(uintptr_t));
    #endif
    #if CONFIG_USB_ENABLED
    usb_init();
    #elif CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG
    usb_serial_jtag_init();
    #else
    uart_stdout_init();
    #endif
    machine_init();
    ...
```

进行了这些初始化操作，而通过串口输出的 `loop` 会因为 `Ctrl + C` 而停止，键盘输入没有因为死循环而被禁用，就是因为这里调用 `uart_stdout_init` 进行了初始化

```c
void uart_stdout_init(void) {
	uart_config_t uartcfg = {
		.baud_rate = MICROPY_HW_UART_REPL_BAUD,
		.data_bits = UART_DATA_8_BITS,
		.parity = UART_PARITY_DISABLE,
		.stop_bits = UART_STOP_BITS_1,
		.flow_ctrl = UART_HW_FLOWCTRL_DISABLE,
		.rx_flow_ctrl_thresh = 0
	};
	uart_param_config(MICROPY_HW_UART_REPL, &uartcfg);

	const uint32_t rxbuf = 129; // IDF requires > 128 min
	const uint32_t txbuf = 0;

	uart_driver_install(MICROPY_HW_UART_REPL, rxbuf, txbuf, 0, NULL, 0);

	uart_isr_handle_t handle;
	uart_isr_free(MICROPY_HW_UART_REPL);
	uart_isr_register(MICROPY_HW_UART_REPL, uart_irq_handler, NULL, ESP_INTR_FLAG_LOWMED | ESP_INTR_FLAG_IRAM, &handle);
	uart_enable_rx_intr(MICROPY_HW_UART_REPL);
}
```

而这其中的函数 `uart_driver_install` 是一个 `esp-idf` 库函数，我们可以在官方文档中找到它的功能

![官方文档中的函数声明](Untitled4.png)

![官方文档中对函数功能的描述](Untitled5.png)

这个函数将对应的中断响应程序 install 到内存的某个部分来等待中断。
    
### RTOS 进行线程级调度
    
此外，MicroPython 本身并不是一个单线程的程序，它也会通过 FreeROTS 的线程来执行自己的函数。比如在上面提到的，在一个没有 delay 的循环中，输出的内容仍然会被送到 webrepl 客户端，这是通过在 FreeRTOS 上 spawn 出来的线程来发送的。

```c
STATIC mp_obj_t ppp_connect_py(size_t n_args, const mp_obj_t *args, mp_map_t *kw_args) {
	enum { ARG_authmode, ARG_username, ARG_password };
		...
	if (xTaskCreatePinnedToCore(pppos_client_task, "ppp", 2048, self, 1, (TaskHandle_t *)&self->client_task_handle, MP_TASK_COREID) != pdPASS) {
		mp_raise_msg(&mp_type_RuntimeError, MP_ERROR_TEXT("failed to create worker task"));
	}

	return mp_const_none;
}
```

在这里 spawn 出了一个线程，执行的是 `pppos_client_task` ，这个 task

```c
static void pppos_client_task(void *self_in) {
	ppp_if_obj_t *self = (ppp_if_obj_t *)self_in;
	uint8_t buf[256];

	while (ulTaskNotifyTake(pdTRUE, 0) == 0) {
		int err;
		int len = mp_stream_rw(self->stream, buf, sizeof(buf), &err, 0);
		if (len > 0) {
			pppos_input_tcpip(self->pcb, (u8_t *)buf, len);
		}
	}

	self->client_task_handle = NULL;
	vTaskDelete(NULL);
}
```

的功能就是不断从 MicroPython 的 I/O stream 中取东西，交由 PPP 来进行处理。由于是另一个线程，会被 FreeRTOS 调度出来，也就不会被 MicroPython 主线程上的循环给 block 住了。

### MicroPython 进行事件响应
    
那么为什么对 webrepl server 的连接请求又会被 block 呢？MicroPython 内部还有一套原生的 Event 系统，这些 events 会在特定的时候被放到 MicroPython 的 foreground 来运行。还是以 webrepl 为例，等待 webrepl client 的连接请求显然是被放到了 background 中的，而无法在没有 sleep 的循环中直接被响应又告诉我们这并不在另一个线程上运行，也并非中断。

MicroPython 本身维护了一个链表，记录着那些需要被处理的回调函数

```python
def setup_conn(port, accept_handler):
	global listen_s
	listen_s = socket.socket()
	listen_s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

	ai = socket.getaddrinfo("0.0.0.0", port)
	addr = ai[0][4]

	listen_s.bind(addr)
	listen_s.listen(1)
	if accept_handler:
		listen_s.setsockopt(socket.SOL_SOCKET, 20, accept_handler)
	for i in (network.AP_IF, network.STA_IF):
		iface = network.WLAN(i)
		if iface.active():
			print("WebREPL daemon started on ws://%s:%d" % (iface.ifconfig()[0], port))
	return listen_s
```

这段 Python 代码调用了一个用 C 语言编写的函数

```c
STATIC mp_obj_t socket_setsockopt(size_t n_args, const mp_obj_t *args) {
	...
	#if MICROPY_PY_USOCKET_EVENTS
	// level: SOL_SOCKET
	// special "register callback" option
	case 20: {
		if (args[3] == mp_const_none) {
			if (self->events_callback != MP_OBJ_NULL) {
				usocket_events_remove(self);
				self->events_callback = MP_OBJ_NULL;
			}
		} else {
			if (self->events_callback == MP_OBJ_NULL) {
				usocket_events_add(self);
			}
			self->events_callback = args[3];
		}
		break;
	}
	#endif
	...
	return mp_const_none;
}
```

这里将自己这个 socket 对象添加到 events 链表中，调用的是函数 `usocket_events_add`.

```c
STATIC void usocket_events_add(socket_obj_t *sock) {
	sock->events_next = usocket_events_head;
	usocket_events_head = sock;
}
```

将事件添加到链表头。而处理这些回调函数（事件）的，是在主线程中主动调用的函数，使用了类似轮询的方式，不断查看有没有需要处理的事件。比如处理 `usocket` 事件的

```c
void usocket_events_handler(void) {
	if (usocket_events_head == NULL) {
		return;
	}
	if (--usocket_events_divisor) {
		return;
	}
	usocket_events_divisor = USOCKET_EVENTS_DIVISOR;

	fd_set rfds;
	FD_ZERO(&rfds);
	int max_fd = 0;

	for (socket_obj_t *s = usocket_events_head; s != NULL; s = s->events_next) {
		FD_SET(s->fd, &rfds);
		max_fd = MAX(max_fd, s->fd);
	}

	// Poll the sockets
	struct timeval timeout = { .tv_sec = 0, .tv_usec = 0 };
	int r = select(max_fd + 1, &rfds, NULL, NULL, &timeout);
	if (r <= 0) {
		return;
	}

	// Call the callbacks
	for (socket_obj_t *s = usocket_events_head; s != NULL; s = s->events_next) {
		if (FD_ISSET(s->fd, &rfds)) {
			mp_call_function_1_protected(s->events_callback, s);
		}
	}
}
```

就在每次被调用到的时候取处理我们添加到链表中的 events. 而在很多函数中，我们都能够找到这个函数以宏的形式得到了调用

![对改 Handler 的使用](Untitled6.png)

```c
#define MICROPY_EVENT_POLL_HOOK \
do { \
	extern void mp_handle_pending(bool); \
	mp_handle_pending(true); \
	MICROPY_PY_USOCKET_EVENTS_HANDLER \
	MP_THREAD_GIL_EXIT(); \
	ulTaskNotifyTake(pdFALSE, 1); \
	MP_THREAD_GIL_ENTER(); \
} while (0);
```

![添加该函数 Hook 的位置](Untitled7.png)

在这些地方，都会检查是否有还未被处理的 event，然后直接由主线程进行处理。较为典型的，有在 repl 获取命令时的 `mp_task() --> pyexec_friendly_repo() --> readline() --> mp_hal_stdin_rx_chr()` 中，就使用了这个宏

```c
int mp_hal_stdin_rx_chr(void) {
	for (;;) {
		int c = ringbuf_get(&stdin_ringbuf);
		if (c != -1) {
			return c;
		}
		MICROPY_EVENT_POLL_HOOK
	}
}
```

我们在来看 `time.sleep_ms` 的实现

```c
STATIC mp_obj_t time_sleep_ms(mp_obj_t arg) {
	mp_int_t ms = mp_obj_get_int(arg);
	if (ms >= 0) {
		mp_hal_delay_ms(ms);
	}
	return mp_const_none;
}
```

而这里调用的 `mp_hal_delay_ms` 中，又有查询 socket 中事件的宏

```c
void mp_hal_delay_ms(uint32_t ms) {
	uint64_t us = ms * 1000;
	uint64_t dt;
	uint64_t t0 = esp_timer_get_time();
	for (;;) {
		mp_handle_pending(true);
		MICROPY_PY_USOCKET_EVENTS_HANDLER
		MP_THREAD_GIL_EXIT();
		uint64_t t1 = esp_timer_get_time();
		dt = t1 - t0;
		if (dt + portTICK_PERIOD_MS * 1000 >= us) {
			// doing a vTaskDelay would take us beyond requested delay time
			taskYIELD();
			MP_THREAD_GIL_ENTER();
			t1 = esp_timer_get_time();
			dt = t1 - t0;
			break;
		} else {
			ulTaskNotifyTake(pdFALSE, 1);
			MP_THREAD_GIL_ENTER();
		}
	}
	if (dt < us) {
		// do the remaining delay accurately
		mp_hal_delay_us(us - dt);
	}
}
```

所以我们在进行 `time.sleep_ms` 的时候，也能够响应 socket 的事件。

综上，MicroPython 用三种不同的机制来实现了类似事件驱动的效果。感觉这样做似乎并不太 elegant，但可能出于性能考虑选取的最优解，三种方式的开销也确实一种比一种小。本次阅读 MicroPython 源码使我们获益良多。（虽然他们把 initializer 拼成 initialiser）


## boot.py, main.py 的运行

运行这两个文件的代码也在 `mp_task` 中

```c
void mp_task(void *pvParameter) {
		...
    // run boot-up scripts
    pyexec_frozen_module("_boot.py");
    pyexec_file_if_exists("boot.py");
    if (pyexec_mode_kind == PYEXEC_MODE_FRIENDLY_REPL) {
        int ret = pyexec_file_if_exists("main.py");
        if (ret & PYEXEC_FORCED_EXIT) {
            goto soft_reset_exit;
        }
    }
		...
    goto soft_reset;
}
```

最后这个文件中读出来的内容都去调了这个函数

```c
int pyexec_file(const char *filename) {
    return parse_compile_execute(filename, MP_PARSE_FILE_INPUT, EXEC_FLAG_SOURCE_IS_FILENAME);
}
```

和 REPL 中我们的输入共用一个函数，只是改变了一个参数，这个参数仅带来 parse 的不同

```c
mp_parse_tree_t mp_parse(mp_lexer_t *lex, mp_parse_input_kind_t input_kind) {
		...
    // work out the top-level rule to use, and push it on the stack
    size_t top_level_rule;
    switch (input_kind) {
        case MP_PARSE_SINGLE_INPUT:
            top_level_rule = RULE_single_input;
            break;
        case MP_PARSE_EVAL_INPUT:
            top_level_rule = RULE_eval_input;
            break;
        default:
            top_level_rule = RULE_file_input;
    }
    push_rule(&parser, lex->tok_line, top_level_rule, 0);
		...
```

运行这两个文件的方式不过是读出来然后直接解释执行。