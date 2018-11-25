# note-8
&ensp;&ensp&ensp;&ensp; 67 dhcp服务器端口  68 客户端端口   69 tftp端口   80http服务
网卡引导之后，会向网络中发送dhcp请求，当服务器收到之后，会把配置好的地址利用DHCP协议发送给主机，
还有thtp服务器的地址（标记了服务器上哪个文件是做引导系统的），当客户机收到之后，会根据地址利用
udp协议69端口链接tftp服务器，从服务器下载启动文件（类似grub，从哪加载应答文件），
利用文件在去下载内核文件，利用应答文件找到yum源，安装相应的包，利用应答文件定义的方法，安装包
实验：实现centos7实现基于PXE安装centos 7
1安装相关包
yum install  dhcp tftp-server  syslinux
systemctl enable http  dhcp tftp-server  syslinux下次开启启动
2启动httpd服务，配置yum源
mkdir /var/www/html/centos/{6,7}/os/x86_64/ -pv
mount /dev/sr0  centos/7/os/x86_64/
拷贝应答文件
cp anaconda-ks.cfg /var/www/html/ksdir/ks7-mini.cfg
vim  /var/www/html/ksdir/ks7-mini.cfg修改文件

auth --enableshadow --passalgo=sha512
# Use CDROM installation media
url --url=http://192.168.66.128/centos/7/os/x86_64/（路径）
# Use graphical install
text（文本安装）
# Run the Setup Agent on first boot
firstboot --enable
ignoredisk --only-use=sda
# Keyboard layouts
keyboard --vckeymap=us --xlayouts='us'
# System language
network  --hostname=CentOS7
# System services
firewall --disabled关闭防火墙
selinux --disabled禁用selinux
services --disabled="chronyd"
# System timezone
timezone Asia/Shanghai --isUtc --nontp
# X Window System configuration information
#xconfig  --startxonboot
# System bootloader configuration
bootloader --append=" crashkernel=auto" --location=mbr --boot-drive=sda
# Partition clearing information
clearpart --all --initlabel
zerombr清除mbr分区
reboot重启
# Disk partitioning information
part /boot --fstype="xfs" --ondisk=sda --size=1024
part swap --fstype="swap" --ondisk=sda --size=4096
part /data --fstype="xfs" --ondisk=sda --size=30720
part / --fstype="xfs" --ondisk=sda --size=51200

%packages
@core
%end   
修改权限                                                             
 chmod +r /var/www/html/ksdir/ks7-mini.cfg
 3配置dhcp服务
cp   /usr/share/doc/dhcp-4.2.5/dhcpd.conf.example      /etc/dhcp/dhcpd.conf 
修改文件 /etc/dhcp/dhcpd.conf 
subnet 192.168.66.0 netmask 255.255.255.0 {
        range 192.168.66.10 192.168.66.200;
        option routers 192.168.66.1;
        option domain-name-servers 8.8.8.8;
        next-server 192.168.66.128;
        filename "pxelinux.0";        
启动服务systemctl start dhcpd
4准备pxe的相关文件
mkdir  /var/lib/tftpboot/pxelinux.cfg/
cp /usr/share/syslinux/{menu.c32,pxelinux.0}  /var/lib/tftpboot/
cp  /misc/cd/isolinux/{vmlinuz,initrd.img}  /var/lib/tftpboot/
cp  /misc/cd/isolinux/isolinux.cfg  /var/lib/tftpboot/pxelinux.cfg/default
/var/lib/tftpboot/
├── initrd.img
├── menu.c32
├── pxelinux.0
├── pxelinux.cfg
│   └── default
└── vmlinuz
5准备修改菜单 /var/lib/tftpboot/pxelinux.cfg/default
default menu.c32
timeout 600

menu title PXE Install  CentOS 
label mini
   menu label ^Auto Install Mini Centos 7
   kernel vmlinuz
   append initrd=initrd.img ks=http://192.168.66.128/ksdir/ks7-mini.cfg

label desktop（图形）
   menu label ^Auto Install desktop Centos 7
   kernel vmlinuz
   append initrd=initrd.img ks=http://192.168.66.128/ksdir/ks7-mini.cfg 

label local
  menu default
  menu label Boot from ^local drive
  localboot 0xffff
  发布环境（根据类型或者地区）
灰度发布（金丝雀发布）：每次只发布一部分机器，把一部分机器下线，
利用调度器把这部分机器设为down 状态，把这部分机器更新软件（修改软连接），升级完上线，
让百分之10访问（vip），在使用软件的时候有问题，反馈，修复，如果没有问题，在上线一部分机器，
知道全部发布完.
蓝绿发布：（主备两套环境）
活动环境（绿色）对外提供服务，备用是非活动（蓝色），先升级备用环境，
把备用环境变化成主要环境，用户访问升级环境，如果发现问题，可以切换回备用环境
file模块管理远程主机的文件，设置权限
在远程主机data目录下创建新文件dest  name  path  都是指定目标文件
ansible  all   -m   file   -a  ‘path=/data/file.txt(指定目标文件）    state=touch(执行创建命令）'    
ansible  all   -m   file   -a  ‘path=/data/file.txt(指定目标文件）    state=absent(执行删除命令）'
指定创建软连接
ansible all   -m  file  -a  'src=/data/fstab(源文件）   path=/data/fstab.link(要生成的文件）  state=link执行创建软连接命令'
指定创建硬连接
ansible all   -m  file  -a  'src=/data/fstab(源文件）   path=/data/fstab.link(要生成的文件）  state=hard执行创建软连接命令'
创建文件夹
ansible  all   -m  file   -a  'dest=/data/dir1(指定目标文件）   state=directory（定义类型，文件夹）'   
如果目录是挂载点，会删除里面的数据，但不会删除目录
ansible  all   -m  file   -a  'dest=/data/  state=absent'
查看服务端口
ansible  websrvs  -m shell   -a 'ss -ntl'
启动服务httpd
ansible websrvs -m service  -a 'name=httpd state=started'
关闭服务端口
ansible websrvs -m service  -a 'name=httpd state=stopped'
设置开机启动         systemctl  is-enabled  httpd查看开机是否启动
ansible websrvs -m service  -a 'name=httpd enabled=yes'
centos6版本查看开机是否启动  chkconfig  --list  httpd
rpm -q httpd --scripts查看包的信息
账户管理
创建用户    创建系统组加上  system=yes    500以内
ansible  all  -m  user  -a  'name=test(用户名）  comment="test user" (描述）  uid=2000   home=/data/testhome(指定家目录）  group=chen(设置主组）  groups=nobody(设置辅助组）  shell=/sbin/nolgin(指定shell 类型）‘
删除用户(不会删除用户的家目录）
ansible   all   -m   user  -a  'name=test  state=absent'
删除用户和家目录
ansible  all  -m  user  -a  'name=test  state=absent   remove=yes'
