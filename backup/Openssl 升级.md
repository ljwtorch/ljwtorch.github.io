## 升级支持TLS 1.3

```shell
# 下载
wget https://www.openssl.org/source/openssl-3.0.13.tar.gz
# 解压
tar -zxvf openssl-3.0.13.tar.gz
# 配置、编译
./config  --prefix=/usr/local/openssl
make
# 检查是否出错
make test
# 安装
make install
```

安装完成后检查链接库是否正常，将缺少的文件直接软链接到系统的`/usr/lib64`目录下

```shell
cd /usr/local/openssl/bin
ldd openssl

linux-vdso.so.1 =>  (0x00007ffe3893b000)
libssl.so.3 => not found
libcrypto.so.3 => not found
libdl.so.2 => /lib64/libdl.so.2 (0x00007ff49e5e0000)
libpthread.so.0 => /lib64/libpthread.so.0 (0x00007ff49e3c4000)
libc.so.6 => /lib64/libc.so.6 (0x00007ff49dff6000)
/lib64/ld-linux-x86-64.so.2 (0x00007ff49e7e4000)

ln -s /usr/local/openssl/lib64/libssl.so.3 /usr/lib64/libssl.so.3
ln -s /usr/local/openssl/lib64/libcrypto.so.3 /usr/lib64/libcrypto.so.3

linux-vdso.so.1 =>  (0x00007fffd2ddb000)
libssl.so.3 => /lib64/libssl.so.3 (0x00007fbc48fc8000)
libcrypto.so.3 => /lib64/libcrypto.so.3 (0x00007fbc48954000)
libdl.so.2 => /lib64/libdl.so.2 (0x00007fbc48750000)
libpthread.so.0 => /lib64/libpthread.so.0 (0x00007fbc48534000)
libc.so.6 => /lib64/libc.so.6 (0x00007fbc48166000)
/lib64/ld-linux-x86-64.so.2 (0x00007fbc4926d000)
```

报错：

```shell
Can't locate IPC/Cmd.pm in @INC (@INC contains: /home/xxx/openssl-3.0.13/util/perl /usr/local/lib64/perl5 /usr/local/share/perl5 /usr/lib64/perl5/vendor_perl /usr/share/perl5/vendor_perl /usr/lib64/perl5 /usr/share/perl5 . /home/xxx/openssl-3.0.13/external/perl/Text-Template-1.56/lib) at /home/xxx/openssl-3.0.13/util/perl/OpenSSL/config.pm line 19.
BEGIN failed--compilation aborted at /home/xxx/openssl-3.0.13/util/perl/OpenSSL/config.pm line 19.
Compilation failed in require at /home/xxx/openssl-3.0.13/Configure line 23.
BEGIN failed--compilation aborted at /home/xxx/openssl-3.0.13/Configure line 23.

yum install perl-IPC-Cmd
```

