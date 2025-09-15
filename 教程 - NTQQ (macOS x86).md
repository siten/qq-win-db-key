## 0. 背景信息

* 系统：`macOS Sequoia 15.5`
* 架构：x86_64
* QQ：[6.9.58-28971 版本](https://qq.en.uptodown.com/mac/download/1033570073)
   * 发布日期：2024 年 10 月 28 号
   * sha256：`47a09e5ea5211012ca0f7e7ef634aef0c4a3fe87c27bac61ac0b74afdbd577d1`
   * 备份链接：[共享“qq-6-9-58.dmg” - GoogleDrive](https://drive.google.com/file/d/1ktW7jg1ZdkOrnmEyWAT0LUyyO3eBgXr_/view?usp=sharing)
   * 备注：请比对好 sha256，或者直接从官网下载最新版。最新版可能有变动

特别注意：这篇教程是写给 x86 架构的 macOS 系统用户写的，如果是 ARM 架构的 macOS 用户，请参考[这篇文章](https://gilbert.vicp.io/2024/03/20/Unlocking-Memories-Extracting-QQ-NT-Chat-History-from-Encrypted-Databases/)。


## 1. 准备阶段

关闭 SIP，具体参考[这篇资料](https://gist.github.com/imhet/c96889529826915343d7b41a4c6ec770)。

## 2. 获取 key

### 2.1 获取 nt_sqlite3_key_v2 函数的基础地址

```bash
cp /Applications/QQ.app/Contents/Resources/app/wrapper.node .

objdump -d wrapper.node | grep -B 20 "nt_sqlite3_key_v2"

 332cdd7:	48 8d 35 73 cb 57 00	leaq	0x57cb73(%rip), %rsi ## literal pool for: "main"
 332cdde:	48 89 df	movq	%rbx, %rdi
 332cde1:	4c 89 fa	movq	%r15, %rdx
 332cde4:	44 89 f1	movl	%r14d, %ecx
 332cde7:	48 83 c4 08	addq	$0x8, %rsp
 332cdeb:	5b	popq	%rbx
 332cdec:	41 5e	popq	%r14
 332cdee:	41 5f	popq	%r15
 332cdf0:	5d	popq	%rbp
 332cdf1:	e9 00 00 00 00	jmp	0x332cdf6
 332cdf6:	55	pushq	%rbp
 332cdf7:	48 89 e5	movq	%rsp, %rbp
 332cdfa:	41 57	pushq	%r15
 332cdfc:	41 56	pushq	%r14
 332cdfe:	41 54	pushq	%r12
 332ce00:	53	pushq	%rbx
 332ce01:	41 89 ce	movl	%ecx, %r14d
 332ce04:	49 89 d7	movq	%rdx, %r15
 332ce07:	49 89 f4	movq	%rsi, %r12
 332ce0a:	48 89 fb	movq	%rdi, %rbx
 332ce0d:	48 8d 35 cb a3 65 00	leaq	0x65a3cb(%rip), %rsi ## literal pool for: "nt_sqlite3_key_v2: db=%p zDb=%s"
```

分析：

1. `jmp` 指令无条件地将程序的执行流转移到了 `0x332cdf6`。这是一种尾调用优化
2. 在 `0x332cdf6` 这个地址，我们看到了 x86-64 架构下标准的函数序言指令
    1. pushq	%rbp        ; 保存调用者的栈底指针
    2. movq	%rsp, %rbp    ; 建立本函数新的栈帧

综合上面的信息，`nt_sqlite3_key_v2` 函数的第一条指令位于地址 `0x332cdf6`

### 2.2 通过 lldb 获取 key

sqlite3_key_v2 方法定义：

```bash
int sqlite3_key_v2(
  sqlite3 *db,                   // 解密的数据库，第一个参数，寄存器：rdi
  const char *zDbName,           // 解密的数据库，第二个参数，寄存器：rsi
  const void *pKey, int nKey     // 第三个参数密钥：rdx、第四个参数密钥长度：rcx
);
```


```bash
ps aux | grep 'QQ$' | awk '{print $2}'
2349

# 开始调试
lldb --attach-pid 37456

# 获取 wrapper.node 在内存的起始位置
(lldb) image list -o -f | grep /Applications/QQ.app/Contents/Resources/app/wrapper.node
[  0] 0x0000000110068000 /Applications/QQ.app/Contents/Resources/app/wrapper.node

# 根据第一步获取的地址，计算函数在当前环境下的真实地址
(lldb) expr 0x0000000110068000 + 0x332cdf6
(long) $0 = 4617489910

# 断点
(lldb) br s -a 4617489910

# 继续运行 QQ
(lldb) c

# 这时候，点击 QQ 的任意聊天窗口，让其运行。
# 成功的话，会在我们的断点停下

# key 长度
# 如果这里读取到 0x10，基本就是对的。代表 key 的长度为 16（十进制）
(lldb) register read rcx
      rcx = 0x0000000000000010

# 读取 key 的地址
(lldb) register read rdx
      x2 = 0x0000010805f442c0

# 读取 key。这里的【Z[12?_]7OMsX?X22】就是 key
# 不同环境下，这个值是不同的，仅供参考
(lldb) memory read --format c --count 16 --size 1 0x0000010805f442c0
0x10805f442c0: Z[12?_]7OMsX?X22

# 退出调试
(lldb) detach
(lldb) exit

```

## 3. 数据数据库

```bash
# 数据库的位置：~/Library/Containers/com.tencent.qq/Data/Library/Application Support/QQ/nt_qq_{MD5}/nt_db
# 参考值：~/Library/Containers/com.tencent.qq/Data/Library/Application Support/QQ/nt_qq_cc067b8bcbf8980fabd93574e09d9efa/nt_db
# 复制里面的 profile_info.db，拿来测试

cat ./profile_info.db| tail -c +1025 > profile_info_clean.db
sqlcipher ./profile_info_clean.db

PRAGMA key = "your_key";
PRAGMA kdf_iter = 4000;
PRAGMA cipher_page_size = 4096;
PRAGMA cipher_hmac_algorithm = HMAC_SHA1;
PRAGMA cipher_default_kdf_algorithm = PBKDF2_HMAC_SHA512;

sqlite> .tables
buddy_list                   profile_info_v6
```

至此解密成功