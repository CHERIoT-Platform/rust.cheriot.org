---
layout: post
title:  "Status update #1"
author: Sarah Harris
date: 2026-09-23
---

Much the core of the compiler and standard library is now working (for example, 100% or 2,756 of the tests for the core library now pass), and we've started introducing some of the infrastructure to allow Rust code to make use of CHERIoT-specific features.
The focus of this article will be the new Memory Mapped IO (MMIO) attribute, which makes it possible to access embedded peripherals with pure Rust.

This means it is now possible to write drivers for CHERIoT hardware peripherals, everything from single pins to Ethernet, without having to resort to C shims to get access to the devices.
That's good news if you want to take advantage of Rust's safety features in driver code, or if you just want to have one less build step that can break!

Do keep in mind, though, that this is only a low level interface to the hardware, and will probably need some sort of wrapper or abstraction to be useful.
MMIO registers are by nature `unsafe`, and need to be handled with care.
In particular, creating a normal reference (`&` or `&mut`) to them is likely to be undefined behaviour, because reads and writes need to happen at specific times, and the values in memory may change unexpectedly.
You will likely want to use the pointer API, especially [`std::ptr::read_volatile()`/`write_volatile()`](https://doc.rust-lang.org/stable/std/ptr/fn.read_volatile.html) and the [raw borrow operators](https://doc.rust-lang.org/reference/expressions/operator-expr.html?highlight=question#r-expr.borrow.raw) (`&raw const` and `&raw mut`), or a more convenient wrapper like the [`derive_mmio` crate](https://crates.io/crates/derive-mmio).

As an example, here's a fairly crude example of what access to a hardware UART without using any helper libraries looks like.
This uses [Sonata](https://lowrisc.github.io/sonata-system/) as the target hardware.

```Rust
#![crate_type = "lib"]
#![no_std]
#![feature(cheriot_attributes)]

use core::ptr::{read_volatile, write_volatile};

/// This type defines the layout of the UART's registers in memory.
/// This layout is a limited view of Sonata's UART: https://opentitan.org/book/hw/ip/uart/doc/registers.html
#[repr(C)] // prevent reordering of fields.
struct RawUART {
	interrupt_state: u32,
	interrupt_enable: u32,
	interrupt_test: u32,
	alert_test: u32,
	control: u32,
	status: u32,
	read: u32,
	write: u32,
}

// Flags in `RawUART::control`:
// Enable receiver.
const CONTROL_RECEIVE_ENABLE: u32 = 1<<1;
// Enable transmitter.
const CONTROL_TRANSMIT_ENABLE: u32 = 1<<0;
const ENABLE_BITS: u32 = CONTROL_RECEIVE_ENABLE | CONTROL_TRANSMIT_ENABLE;

// Flags in `RawUART::status`:
/// Receive FIFO is empty.
const STATUS_RECEIVE_EMPTY: u32 = 1<<5;
/// Transmit FIFO is full.
const STATUS_TRANSMIT_FULL: u32 = 1<<0;

/// These statics define the actual instances of UART hardware on the chip.
/// There will often be more than one available.
/// We use the new MMIO attribute to tell the compiler what hardware we mean,
/// and what access to it we need.
/// In this case, read and write.
unsafe extern "Rust" {
	#[cheriot_mmio(name = "uart", permissions = "RW")]
	static mut UART0: RawUART;
	#[cheriot_mmio(name = "uart1", permissions = "RW")]
	static mut UART1: RawUART;
	#[cheriot_mmio(name = "uart2", permissions = "RW")]
	static mut UART2: RawUART;
}
```

Just like other embedded devices, CHERIoT devices expose hardware peripherals by mapping their registers into memory at a fixed address.
CHERIoT differs from other devices in that it provides very granular control over what code has access to what devices, and in what ways.
Instead of just creating a pointer to some interesting register whenever we feel like it, say `volatile uint32_t *self_destruct_device = (volatile uint32_t *)0xdeadbeef;`, we need to use compiler features to request access for our code.
This is what the `cheriot_mmio` attribute does, with the convenient side effect that we can request devices by name, instead of having to carefully copy a raw address.
The details of the device are retrieved from the board specification file.

We do still need to tell the type system how to use the memory though; we define the struct `RawUART` and carefully control its layout so that it matches the locations of the UART registers in memory.
We then bind it to multiple instances of UART hardware that exist at different offsets in memory.

The string of letters in the attribute describe the access we're asking for.
This corresponds to a subset of the permission bits in a capability, and covers simple things like read and write (so we could choose to only observe a device without asking for control of it), through to more advanced features like the ability to load a capability.

The permissions available and their corresponding letters are:
* `R` allow reading (corresponds to the `PERMIT_LOAD` capability permission)
* `W` allow writing (corresponds to the `PERMIT_STORE` capability permission)
* `c` allow reading and writing of capabilities (corresponds to the `PERMIT_LOAD_STORE_CAPABILITY` capability permission)
* `g` allow reading capabilities with the global permission (corresponds to the `PERMIT_LOAD_GLOBAL` capability permission)
* `m` allow reading capabilities with the write permission (corresponds to the `PERMIT_LOAD_MUTABLE` capability permission)

To make the above usable from safe Rust, we might go on to write an abstraction like this:

```Rust
/// Here's a sketch of the safe interface our driver might present to other
/// code.
/// Ideally, this should make use of compartments for better code reuse and
/// security, but that's out of scope for this example.
pub struct UART {
	raw: *mut RawUART,
}
impl UART {
	/// Initialise a given UART.
	pub fn new(uart_number: usize) -> UART {
		// Figure out which UART to use.
		let raw = match uart_number {
			0 => &raw mut UART0,
			1 => &raw mut UART1,
			2 => &raw mut UART2,
			_ => panic!("invalid UART number"),
		};

		unsafe {
			assert!(read_volatile(&raw const (*raw).control) & ENABLE_BITS == 0, "UART already in use");

			// TODO: you would need to configure the UART here (connect pins, set baud rate, etc).

			// Enable UART.
			let control = read_volatile(&raw const (*raw).control);
			write_volatile(&raw mut (*raw).control, control | ENABLE_BITS);
		}

		UART { raw }
	}
	/// Read bytes until all of `data` has been overwritten.
	/// TODO: for a real application, you would probably want to implement `std::io::Read` and `std::io::Write` and add error handling.
	pub fn read(&mut self, data: &mut [u8]) {
		for index in 0..data.len() {
			unsafe {
				// Wait for data, we might want to sleep here instead of busy-waiting.
				while read_volatile(&raw const (*self.raw).status) & STATUS_RECEIVE_EMPTY != 0 {}

				// Read more data.
				data[index] = read_volatile(&raw const (*self.raw).read) as u8;
			}
		}
	}
	/// Write bytes until all of `data` has been queued up to transmit.
	pub fn write(&mut self, data: &[u8]) {
		for byte in data.iter().copied() {
			unsafe {
				// Wait until hardware is ready for more data.
				while read_volatile(&raw const (*self.raw).status) & STATUS_TRANSMIT_FULL != 0 {}
				// Queue more data.
				write_volatile(&raw mut (*self.raw).write, byte as u32);
			}
		}
	}
}
impl Drop for UART {
	fn drop(&mut self) {
		unsafe {
			// Disable UART.
			write_volatile(&raw mut (*self.raw).control, 0);
		}
	}
}
```

For more details about writing CHERIoT device drivers and how the C++ API works, see [this section in the CHERIoT programmer's guide](https://cheriot.org/book/drivers.html).

Under the covers, most of the complicated part of MMIO attributes is handled by LLVM.
The changes to the Rust compiler add the new attribute, parsing to validate the permissions, and then forward the information to LLVM.
If you want the gritty details, see [this pull request](https://github.com/CHERIoT-Platform/cheri-rust/pull/245).

Speaking of pull requests, the MMIO attribute has also paved the way for us to [add attributes for shared objects](https://github.com/CHERIoT-Platform/cheri-rust/pull/253) imported from other compartments.
It may be a small feature on its own, but it's another step toward full compartment support in Rust!
