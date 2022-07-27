# Lustre-KMOD-2.12.9-with-ZFS-0.7.13-on-Centos-7.9

```text

rpm --version
RPM version 4.11.3

```
After compiling the KMOD modules of Lustre 2.12.9 with ZFS 0.7.13 on CentOS 7.9, kernel 3.10.0-1160.49.1. 
When trying to install the  compiled rpms with "yum" command, it report huge keym errors:

```text
           Requires: ksym(dmu_request_arcbuf) = 0x5b319f50
Error: Package: kmod-lustre-osd-zfs-2.12.9-1.el7.x86_64 (/kmod-lustre-osd-zfs-2.12.9-1.el7.x86_64)
           Requires: ksym(dmu_objset_space) = 0x9289dfa8
Error: Package: kmod-lustre-osd-zfs-2.12.9-1.el7.x86_64 (/kmod-lustre-osd-zfs-2.12.9-1.el7.x86_64)
           Requires: ksym(dmu_object_set_blocksize) = 0x931c2a43
Error: Package: kmod-lustre-osd-zfs-2.12.9-1.el7.x86_64 (/kmod-lustre-osd-zfs-2.12.9-1.el7.x86_64)
           Requires: ksym(sa_handle_get_from_db) = 0x6e885f53
Error: Package: kmod-lustre-osd-zfs-2.12.9-1.el7.x86_64 (/kmod-lustre-osd-zfs-2.12.9-1.el7.x86_64)
           Requires: ksym(dmu_tx_commit) = 0xf6382d3e
Error: Package: kmod-lustre-osd-zfs-2.12.9-1.el7.x86_64 (/kmod-lustre-osd-zfs-2.12.9-1.el7.x86_64)
           Requires: ksym(txg_wait_synced) = 0xcd874555
Error: Package: kmod-lustre-osd-zfs-2.12.9-1.el7.x86_64 (/kmod-lustre-osd-zfs-2.12.9-1.el7.x86_64)
           Requires: ksym(dmu_tx_hold_zap_by_dnode) = 0x88f962d2
Error: Package: kmod-lustre-osd-zfs-2.12.9-1.el7.x86_64 (/kmod-lustre-osd-zfs-2.12.9-1.el7.x86_64)
           Requires: ksym(dmu_tx_hold_zap) = 0x91cee74e
Error: Package: kmod-lustre-osd-zfs-2.12.9-1.el7.x86_64 (/kmod-lustre-osd-zfs-2.12.9-1.el7.x86_64)
           Requires: ksym(sa_replace_all_by_template) = 0xf501af39
```

It looks like it is caused just by the rpm/rpmbuild used, nothing related to the Lustre software itself.
 Since the same version of DKMS package works just fine on the same server.
 
# Solution 1:
 
The only option seems is NOT to generat these ksym module requirements at all(risky!).

One way is to comment line 106 to 107 of file : /usr/lib/rpm/redhat/find-requires.ksyms 
 
 ```text
 /usr/lib/rpm/redhat/find-requires.ksyms 
 
#    LANG=C join -t '\t' -j 1 -v 2 $symvers <(mod_requires "${modules[@]}") | LANG=C sort -u \
#    | awk '{ FS = "\t" ; OFS = "\t" } { print "ksym(" $1 ") = " $2 }'
```
 
 
 Or comment ling 156 of /usr/lib/rpm/redhat/find-requires

 
 ```text
  #     printf "%s\n" "${filelist[@]}" | /usr/lib/rpm/redhat/find-requires.ksyms
 
 ```
 
 Then run 
 ```text
 
 make rpms
 ```
 
 should work. 
 
 Note: this is only a temporary hack for rpm to let Lustre KMOD module can be istalled. It can be very risky.
 
# Solution 2:
 
 The ZFS/SPL packages can be built from source to include these kernel ksym export functions provided by the kernel modules. They have two building modes: generic and redhat. Which provide correlate spec files:
 ```text
 
 rpm/generic/spl-kmod.spec
 rpm/redhat/spl-kmod.spec
 and 
 
 rpm/generic/zfs.spec
 rpm/redhat/zfs.spec

```

You need to use the "--with-spec=[generic|redhat]" to choose which mode you want to use. The default mode us generic. 


The "redhat" mode by default will export all the provided ksy functions. The only issue for the redhat mode is it can only support the lastest Linux 
kernel installed. If you want to install on an older kernel(while you have a newer kernel on the same computer), the build will the fail. If that's the case, you can change one line of file:

```text
vi /usr/lib/rpm/redhat/macros
...
#change line 214 from:

%global kverrel %(%{kmodtool} verrel %{?kernel_version} 2>/dev/null) 

to 

%global kverrel %(%{kmodtool} verrel  2>/dev/null)

```
Then it will fix the issue.

For the generic mode, you need to add one line in the spec file(before %prep):

```text

%global _use_internal_dependency_generator 0

```

This way to force rpmbuild to scan and export the provided ksym functions of the compiled kernel module.





