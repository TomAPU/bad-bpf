# Bad BPF
A collection of malicious eBPF programs that make use of eBPF's ability to
read and write user data in between the usermode program and the kernel.

# Overview
See my blog for an overview on how these programs work and why
this is interesting: https://blog.tofile.dev/2021/....


# Build
## Dependecies
As this code makes use of CO-RE, it requires a recent version of Linux that has BTF Type information.
See [these notes in the libbpf README](https://github.com/libbpf/libbpf/tree/master#bpf-co-re-compile-once--run-everywhere)
for more information. For example Ubuntu requries `Ubuntu 20.10`+.

To build it requires these dependecies:
- zlib
- libelf
- libbfd
- clang 11
- make

On Ubuntu these can be installed by
```bash
sudo apt install build-essential clang-11 libelf-dev zlib1g-dev libbfd-dev libcap-dev libfd-dev
```

## Build
To use pre-build binaries, grab them from the [Releases](https://github.com/pathtofile/bad-bpf/releases) page.

To Build from source, recusivly clone the respository the run `make` in the `src` directory to build:
```bash
git clone --recusrive https://github.com/pathtofile/bad-bpf.git
cd bad-bpf/src
make
```
The binaries will built into `bad-bpf/src/bin`.

# Run
To run, launch each program as `root`. Every program has a `--help` option
that has required arguemnts and examples.

# Programs
## Common Arguments
As well as `--help`, every program also has a `--target-ppid`/`-t`.
This option restricts the programs' operation to only programs that are children
of the process matching this PID. This demonstrates to how affect some programs, but not others.

## BPF-Dos
```bash
sudo ./bpfdos
```
This program raises a `SIG_KILL` signal to any program attempting to use the `bpf` syscall.


## Pid-Hide
```
sudo ./pidhide --pid-to-hide 2222
```
This program hides the process matching this pid from tools such as `ps`.

It works by hooking the `getdents64` syscall, as `ps` works by looking for every sub-folder
of `/proc/`. PidHide unlinks the folder matching the PID, so `ps` only sees the folders before
and after it.


## Sudo-Add
```
sudo ./sudoadd --username lowpriv-user
```
This program allows a normally low-privledged user to use `sudo` to become root.

It works by intercepting `sudo`'s reading of the `/etc/sudoers` file, and overwriting the first line
with `<username> ALL=(ALL:ALL) NOPASSWD:ALL #`. This tricks sudo into thinking the user is allowed to become
root. Other programs such as `cat` or `sudoedit` are unnafected, so to those programs the file is unchanged
and the user does not have those privliges. The `#` at the end of the line ensures the rest of the line
is trated as a comment, so it doesn't currup the file's logic.

## Text-Replace
```
sudo ./textreplace --filename /path/to/file --input foo --replace bar
```
This program replaces all text matching `input` in the file with the `replace` text.
This has a number of uses, for example:

To hide kernel module `joydev` from tools such as `lsmod`:
```bash
./textreplace -f /proc/modules -i 'joydev' -r 'cryptd'
```

Spoof the MAC address of the `eth0` interface:
```bash
/textreplace -f /sys/class/net/eth0/address -i '00:15:5d:01:ca:05' -r '00:00:00:00:00:00'
```
Malware conducting anti-sandbox checks might check the MAC address to look for signs it is
running inside a Virtual Machine or Sandbox, and not on a 'real' machine.


**NOTE:** Both `input` and `replace` must be the same length, to avoid adding NULL characters to the
middle of a block of text. To enter a newline from a bash prompt, use `$'\n'`, e.g. `--replace $'text\n'`.