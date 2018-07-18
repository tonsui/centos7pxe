centos 7 lab server

intall
	01. install with gnome desktop(option)

gparted
	01. sudo yum install epel-release -y
	02. sudo yum install gparted -y

install services
	01. sudo yum install tftp-server dhcp syslinux vsftpd -y

dhcp
	01. sudo vi /etc/dhcp/dhcpd.conf
		ddns-update-style interim;
		ignore client-updates;
		authoritative;
		allow booting;
		allow bootp;
		allow unknown-clients;

		subnet 10.1.1.0 netmask 255.255.255.0 {
		range 10.1.1.10 10.1.1.250;
		option domain-name-servers 10.1.1.254;
		option domain-name "york.com";
		option routers 10.1.1.254;
		option broadcast-address 10.1.1.255;
		default-lease-time 86400;
		max-lease-time 86400;

		next-server 10.1.1.254;
		filename "pxelinux.0";
		}

tftp
	01. cd /usr/lib/systemd/system
	02. sudo cp tftp.service tftp.service.org
	03. sudo vi tftp.service
		[Unit]
		Description=Tftp Server
		Requires=tftp.socket
		Documentation=man:in.tftpd

		[Service]
		ExecStart=/usr/sbin/in.tftpd -s /tftp
		StandardInput=socket

		[Install]
		Also=tftp.socket

	04. sudo mkdir /tftp
	05. sudo semanage fcontext -a -t public_content_t "/tftp(/.*)?"
	06. sudo restorecon -F -R -v /tftp
	07. cd /usr/share/syslinux
	08. sudo cp pxelinux.0 menu.c32 memdisk mboot.c32 chain.c32 /tftp/
	09. sudo chmod 777 -R /tftp
	10. sudo chmod 777 /tftp
	11. sudo restorecon -F -R -v /tftp
	12. sudo mkdir /tftp/pxelinux.cfg
	13. sudo mkdir -p /tftp/netboot/
	14. sudo restorecon -R -F -v /tftp/
	15. sudo vi /tftp/pxelinux.cfg/default
		default menu.c32
		prompt 0
		timeout 50
		menu title PXE

		label CentOS 7

		menu label york
		kernel ipxe.lkrn
		append dhcp && initrd http://10.1.1.1/win10pe1803.iso && chain memdisk iso raw

set static ip
	01. sudo vi /etc/sysconfig/network-scripts/ifcfg-enp1s0
	[modify]
		BOOTPROTO=static
		ONBOOT=yes
		IPADDR=10.1.1.254
		NETMASK=255.255.255.0
	02. sudo systemctl restart network.service

enable services
	01. sudo systemctl enable vsftpd
	02. sudo systemctl start vsftpd
	03. sudo systemctl enable tftp
	04. sudo systemctl start tftp
	05. sudo systemctl daemon-reload
	06. sudo systemctl enable dhcpd
	07. sudo systemctl start dhcpd

set firewall
	01. sudo firewall-cmd --permanent --add-service=dhcp
	02. sudo firewall-cmd --permanent --add-service=ftp
	03. sudo firewall-cmd --permanent --add-service=tftp
	04. sudo firewall-cmd --reload

samba
	01. yum install samba samba-client samba-common
	02. cd /etc/samba
	03. sudo rm smb.conf
	04. sudo vi smb.conf
		[global]
		dns proxy = no
		map to guest = bad user
		netbios name = ssd
		security = user
		server string = Samba Server %v

		[samba]
		path = /samba
		read only = no
		valid users = ssd

	05. sudo systemctl enable smb.service
	06. sudo systemctl enable nmb.service
	07. sudo systemctl restart smb.service
	08. sudo systemctl restart nmb.service
	09. sudo firewall-cmd --permanent --zone=public --add-service=samba
	10. sudo firewall-cmd --reload
	11. sudo mkdir /samba
	12. sudo chmod -R 777 /samba
	13. sudo chcon -t samba_share_t /samba
	14. sudo smbpasswd -a ssd
	15. sudo systemctl restart smb.service
	16. sudo systemctl restart nmb.service
	17. testparm

mount
	01. sudo fdisk -l
	02. sudo vi /etc/fstab
		/dev/sda1 /samba ext4 defaults 0 2
	03. sudo reboot
	04. sudo chmod -R 777 /samba
	05. sudo chcon -t samba_share_t /samba
	06. sudo reboot

windows client
	01. run: \\ssd
	02. copy win10pe1803.iso
	03. copy host folder

end.
