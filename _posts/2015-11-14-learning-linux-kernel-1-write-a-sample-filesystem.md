---
layout: post
title: Linux�ں�֮�ļ�ϵͳѧϰ�ʼǣ�1����ʶ�ļ�ϵͳ
categories: Linux_kernel
tags: Linux kernel filesystem
---

��Linux�±�̸��ں˴򽻵�����Ƶ���ķ�ʽ���ƾ��ǲ����ļ��ˡ�˭��Linux��ѭ��һ�н��ļ�����˼���ء�

��ˣ�����ѧϰLinux�ں˵ĵ�һ�������Լ�������дһ���򵥵��ļ�ϵͳsamplefs��ϣ������ܹ�����Linux�ں˵Ĵ��š�

# ʲô���ļ�ϵͳ

�й��ļ�ϵͳ�ĸ�������ںܶ����ϵͳ�Ľ̲��Ͽ�����һ�����������ܵģ�

**����**�Ӽ�����洢�Ĳ�νṹ���潲�𣬴洢���ʰ������ٶȣ������ڽ��ʺ�Ѱַ��ʽ���������η�Ϊ��

CPU�ڲ��Ĵ��� --> CPU�ڲ����棨�༶�� --> ���ڴ� --> �����ⲿ�洢�����̡�flash�����̵ȣ�

�ڴ��ǰ��ֽڴ洢���ݵģ�CPUֱ�ӽ�Ҫ���ʵ��ڴ��ַ�͵���ַ�����ϼ��ɣ��������Ĺ���

���ⲿ�洢�Ͳ�һ���ˣ����һ����������ļ���Ҳ������������;������Linux swap����������ͬ�Ĳ���ϵͳ֧�ֵ��ļ����Ͳ�һ����
���Ҳ�ͬ���ļ���СҲ��һ����ÿ������������ϵ��ļ�����Ҫ��Ǹ��ļ������͡���С�����λ�õȵ��ļ���Ϣ���������������Ե����ݣ�Ԫ���ݣ�����ЩԪ����ͬ��Ҫ����������ϡ�

��ˣ���ν��ļ����뻮һ�ı��浽����豸�ϣ��ǲ���ϵͳ����Ҫ��������⡣���������Ĵ𰸾��ǡ����ļ�ϵͳ��

## �ļ�ϵͳ�Ĺ���

���û��򿪼����������ϵͳ���ڵ�½shell�л��Զ����뵽����Ŀ¼��/home/xxx/���¡��û����Կ����Ӹ�Ŀ¼��/����ʼ���ļ�Ŀ¼�������Խ��뵽�κ�һ��Ŀ¼���򿪡��޸Ļ�ɾ���ļ���

����ļ�ϵͳ�ĵ�һ�����ܾ��ǣ������ļ�ϵͳ���ļ��ķ��ʲ�����

���⣬���ָ��û����ļ�Ŀ¼��Ϣ�������ں�����ʱ���ⲿ�洢�豸�ж�ȡ�����ġ�������Ҫ���Ƚ��ļ�ϵͳ����װ�����������豸�ϣ���Linux��ʹ��mkfs.xxx���ߣ���windows�н�������ʽ������

�ڰ�װLinuxϵͳʱ����Ҫ�Դ��̽��з��������ұ���ҪΪ��/����Ŀ¼���������������������֮�󣬾Ϳ��Խ��������Ѱ�װ���ļ�ϵͳ���豸mount����Ŀ¼�ṹ�е�ĳ��·����������mount point������

����˵���ٶ���Ϊ�ҵ��������ubuntu 10.04���������һ��Ӳ�̣���һ������ΪӲ�̷�����������������

```sh
bookxiaow@ubuntu1004:~$ sudo fdisk -l /dev/sdb

Disk /dev/sdb: 2147 MB, 2147483648 bytes
255 heads, 63 sectors/track, 261 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x00000000

Disk /dev/sdb doesn't contain a valid partition table

bookxiaow@ubuntu1004:~$ sudo fdisk /dev/sdb
Device contains neither a valid DOS partition table, nor Sun, SGI or OSF disklabel
Building a new DOS disklabel with disk identifier 0xc5c04c60.
Changes will remain in memory only, until you decide to write them.
After that, of course, the previous content won't be recoverable.

Warning: invalid flag 0x0000 of partition table 4 will be corrected by w(rite)

WARNING: DOS-compatible mode is deprecated. It's strongly recommended to
         switch off the mode (command 'c') and change display units to
         sectors (command 'u').
		 
Command (m for help): m
Command action
   a   toggle a bootable flag
   b   edit bsd disklabel
   c   toggle the dos compatibility flag
   d   delete a partition
   l   list known partition types
   m   print this menu
   n   add a new partition
   o   create a new empty DOS partition table
   p   print the partition table
   q   quit without saving changes
   s   create a new empty Sun disklabel
   t   change a partition's system id
   u   change display/entry units
   v   verify the partition table
   w   write table to disk and exit
   x   extra functionality (experts only)
   
Command (m for help): n
Command action
   e   extended
   p   primary partition (1-4)
p
Partition number (1-4): 1
First cylinder (1-261, default 1): 
Using default value 1
Last cylinder, +cylinders or +size{K,M,G} (1-261, default 261): 
Using default value 261

Command (m for help): p

Disk /dev/sdb: 2147 MB, 2147483648 bytes
255 heads, 63 sectors/track, 261 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0xc5c04c60

   Device Boot      Start         End      Blocks   Id  System
/dev/sdb1               1         261     2095458+  83  Linux

Command (m for help): w
The partition table has been altered!
```

���������Ѿ������÷������ˣ�����ֻ����һ����/dev/sdb1�����������ǰ�װext4�ļ�ϵͳ��

```sh
bookxiaow@ubuntu1004:~$ sudo mkfs.ext4 -c /dev/sdb1
mke2fs 1.41.11 (14-Mar-2010)
Filesystem label=
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
131072 inodes, 523864 blocks
26193 blocks (5.00%) reserved for the super user
First data block=0
Maximum filesystem blocks=536870912
16 block groups
32768 blocks per group, 32768 fragments per group
8192 inodes per group
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912

Checking for bad blocks (read-only test): done                                
Writing inode tables: done                            
Creating journal (8192 blocks): done
Writing superblocks and filesystem accounting information: done

This filesystem will be automatically checked every 30 mounts or
180 days, whichever comes first.  Use tune2fs -c or -i to override.
```

��󣬾Ϳ��Թ��ص���Ŀ¼����

```sh
bookxiaow@ubuntu1004:~$ sudo mkdir /mnt/test
bookxiaow@ubuntu1004:~$ sudo mount -t ext4 /dev/sdb1 /mnt/test
bookxiaow@ubuntu1004:~$ cd /mnt/test/
bookxiaow@ubuntu1004:/mnt/test$ ls
lost+found
```

���Ŀ¼/mnt/test�Ѿ������ˣ����������Ѿ��������ļ�����ô����֮��ԭ�����ļ����ݾͱ������ء������ˣ���unmount֮����ٱ�¶���������С�����ǲ��ǿ��������ض����أ�A_A

# Linux�ļ�ϵͳ�Ľṹ

Linux�У����û�ֱ�ӽ�������VFS��Virtual Filesystem���������û��ṩ��ͳһ���ļ������ӿڣ�open/create/close/write/read/...��������ĳЩ������ĳЩ�ļ�ϵͳ�и�����֧�֡�

[VFS����ṹ]��/image/VFS.jpg��

�������ͼ�У�һ�������ļ��ض�λ��ĳ����װ���ض��ļ�ϵͳ�������飩�ķ����У����Ҷ�Ӧһ�� inode �ṹ������һ��inode���ܻ��Ӧ���Ŀ¼��dentry������˵Ӳ���ӣ�ע�ⲻ��cp����
��һ��Ŀ¼���ͬ�Ľ��̿��ܻ�ͬʱ���������Ҳ�Ͳ����˶��`struct file`�ṹ��������ͬ�Ľ��̾Ϳ��������Լ��Ĵ򿪷�ʽ�Ͷ�дƫ��������ĳЩ����£�Ҳ����ͬ�����̻�ͬ������
�Ķ��fdָ��ͬһ��`struct file`�ṹ������˵`dup()/dup2()`��`fork()`֮���ӽ��̡�

##�����е��ļ���ʾ

�Խ��̶��ԣ�һ���򿪵��ļ���Ӧ��һ��������Ψһ������fd�����fd��Ȼ��ĳ�����������ֵ��

��һ��`struct task_struct`�����ݣ�

```c
struct task_struct {
	/* open file information */
	struct files_struct *files;
};
```

������`struct files_struct`�Ķ��壺

```c
struct files_struct {
	atomic_t count;
	struct fdtable *fdt;
	struct fdtable fdtab;
	fd_set close_on_exec_init;
	fd_set open_fds_init;
	struct file * fd_array[NR_OPEN_DEFAULT]; /*on i386, NR_OPEN_DEFAULT is 32*/
	spinlock_t file_lock; 
};
```

�����и� fd_array���飬��Ԫ������ָ��`struct file`��ָ�롣��ˣ����Բ²���̻���fd������ȥ������������`struct file`���������Сֻ��32�������ܽ���ֻ�ܴ�32���ļ��ɣ�

��ô��ʵ�����ʲô�����أ������Ǹ�һ��writeϵͳ���õ�ʵ�ֿ���������ô��fd�ҵ�fileָ��İɡ�

```c
asmlinkage ssize_t sys_read(unsigned int fd, char __user * buf, size_t count)
{
	int fput_needed;
	file = fget_light(fd, &fput_needed);
}
```

Ҫ����`fget_light`�Ĵ��룬Ҫ�����[RCUԭ��](http://blog.chinaunix.net/uid-12260983-id-2952617.html)����Ϊ�ں�ʹ��RCU������`files_struct`�ĸ��¡�

���ٴ���Ľ���ܲ��죬`struct file*`���鲢����`fd_array`������`fdt->fd`������`fd_array`��ɶ�ã���ʱ���������

## �򿪵��ļ�����`struct file`

һ���򿪵��ļ������ں��ж�Ӧһ��`struct file`�ṹ�壬����[slab������](http://www.ibm.com/developerworks/cn/linux/l-linux-slab-allocator/)������������չ�����

�ļ������Կɷ�Ϊ��̬���ԺͶ�̬���ԡ���̬�������ļ��Ĺ������ԣ������ļ�·�����ļ������ߡ��ļ����͵ȵȣ�����̬�������ڴ��ļ�ʱ�������壬�����ļ���ģʽ��ֻ��/ֻд/��д�����ļ���ǰƫ�ơ�

�ļ�·��������Ŀ¼��dentry���dentry�ֱ�����ָ���ļ�Ԫ���ݵ�inodeָ�룬���file�ṹֻ��Ҫ����ָ��dentry�ṹ��ָ�뼴�ɣ�`struct dentry *f_dentry����

ͬʱ��file�ṹ��ά���ļ��Ķ�̬���ԣ�

```c
struct file {
	struct file_operation *f_op; /*һ���ļ���������ָ��*/
	atomic_t f_count; /*���ṹ���ü���*/
	unsigned_int f_flags; /* �ļ��򿪷�ʽ */
	mode_t f_mode; /* �ļ��Ƿ�ɶ����д*/
	loff_t f_pos; /*��ǰƫ����*/
};
```

�û��ռ���ļ���read/write�������ն��ǵ���`f_op`��ĺ�������fd��file�ṹֻ��Ҫ����һ��ת����������������ײ��ļ�ϵͳûʲô��ϵ����ʵ�ؼ��������`f_op`�ˡ�

��ͬ�ĵײ��ļ�ϵͳ���ڴ����ļ���file�ṹ��ʱ��������`f_op`Ϊָ���ض���`struct file_operation`��ָ�롣

## ���ļ�����inode����������Ŀ¼��dentry

�����ᵽ��`struct file`��ֻ������ָ��`struct dentry`��ָ�룬��˶��ڴ򿪵��ļ���`struct dentry`��Ҫ����ָ��`struct inode`��ָ�롣

��һ���棬��openʱ���ں���Ҫ�����ļ���pathname���ҵ����ļ���Ӧ��inode���ܻ�ȡ���ļ����ݡ�������½��ļ���Ҳ��Ҫ�ҵ����ļ�����Ŀ¼��inode�����ܴ������ļ���

��ˣ���δ�pathname��λ��inode��Ҳ��Ҫ����`struct dentry`��

�й��ļ�·���������ľ���ʵ�֣����Բο�������ϣ�����ֻ������һ�´��¹��̣�

1. ���ȿ��ļ�·�����Ǿ���·����/path/to/file���������·����filename������������ǴӸ�Ŀ¼���Ǵӵ�ǰĿ¼��ʼ����
2. ��ǰ���̵ĸ�Ŀ¼�͵�ǰĿ¼��Ϣ��������`struct tast_struct`��`struct fs_struct *fs`����

```c
struct fs_struct {
	atomic_t count; // ���ü���
	rwlock_t lock; //������
	int umask;
	struct dentry *root, *pwd, *altroot;
	struct vfsmount *rootmnt, *pwdmnt, *altrootmnt;
};
```

3. dentry�ṹ����һ���汣�����Լ����ļ�����Ŀ¼�����ƣ�`d_iname`���������Ŀ¼�Ļ��ֱ�����ָ��Ŀ¼�������ļ�����Ŀ¼����dentry��ָ�루`d_child`���������Ӹ�Ŀ¼����ǰĿ¼����ʼ���ȶԣ�
�����ҵ��ļ���dentry�ṹ�ˡ�

## �����ļ����ݵ����ݡ���inode

