HW2 documentation for modifying firewall rules and intergrating snort into bastion host enviroment.

Xiangqian Zhang

Steps:
1. firewall rull that forward ssh traffic from bastion host port 22 to the Ubuntu system port 22. 
	ext_if="em0"
	int_if="em1"

	icmp_types = "{ echoreq, unreach }"	
	services = "{ ssh, domain, http, ntp, https }"
	server = "192.168.33.111"
	ssh_rdr = "2222"
	table <rfc6890> { 0.0.0.0/8 10.0.0.0/8 100.64.0.0/10 127.0.0.0/8 169.254.0.0/16          \
                  172.16.0.0/12 192.0.0.0/24 192.0.0.0/29 192.0.2.0/24 192.88.99.0/24    \
                  192.168.0.0/16 198.18.0.0/15 198.51.100.0/24 203.0.113.0/24            \
                  240.0.0.0/4 255.255.255.255/32 }
	table <bruteforce> persist


	#options                                                                                                                         
	set skip on lo0

	#normalization
	scrub in all fragment reassemble max-mss 1440

	#NAT rules
	nat on $ext_if from $int_if:network to any -> ($ext_if)

	#RDR rules
	rdr pass on $ext_if proto tcp from any to $ext_if port ssh -> $server port ssh

	#blocking rules
	antispoof quick for $ext_if
	block in quick on egress from <rfc6890>
	block return out quick on egress to <rfc6890>
	block log all

	#pass rules
	pass in quick on $int_if inet proto udp from any port = bootpc to 255.255.255.255 port = bootps keep state label "allow access to DHCP server"
	pass in quick on $int_if inet proto udp from any port = bootpc to $int_if:network port = bootps keep state label "allow access to DHCP server"
	pass out quick on $int_if inet proto udp from $int_if:0 port = bootps to any port = bootpc keep state label "allow access to DHCP server"

	pass in quick on $ext_if inet proto udp from any port = bootps to $ext_if:0 port = bootpc keep state label "allow access to DHCP client"
	pass out quick on $ext_if inet proto udp from $ext_if:0 port = bootpc to any port = bootps keep state label "allow access to DHCP client"

	pass in on $ext_if proto tcp to port { ssh, $ssh_rdr } keep state (max-src-conn 15, max-src-conn-rate 3/1, overload <bruteforce> flush global)
	pass out on $ext_if proto { tcp, udp } to port $services
	pass out on $ext_if inet proto icmp icmp-type $icmp_types
	pass in on $int_if from $int_if:network to any
	pass out on $int_if from $int_if:network to any

	pass out on $int_if proto tcp to port ssh

#	$OpenBSD: sshd_config,v 1.104 2021/07/02 05:11:21 dtucker Exp $

Step2: modified the local ssh server on the firewall to move it to a different port for management puposes
 
	# This is the sshd server system-wide configuration file.  See
	# sshd_config(5) for more information.

	# This sshd was compiled with PATH=/usr/bin:/bin:/usr/sbin:/sbin

	# The strategy used for options in the default sshd_config shipped with
	# OpenSSH is to specify options with their default value where
	# possible, but leave them commented.  Uncommented options override the
	# default value.

	# Note that some of FreeBSD's defaults differ from OpenBSD's, and
	# FreeBSD has a few additional options.

	Port 2222
	#Port 22
	#AddressFamily any
	#ListenAddress 0.0.0.0
	#ListenAddress ::

	#HostKey /etc/ssh/ssh_host_rsa_key
	#HostKey /etc/ssh/ssh_host_ecdsa_key
	#HostKey /etc/ssh/ssh_host_ed25519_key

	# Ciphers and keying	
	#RekeyLimit default none

	# Logging
	#SyslogFacility AUTH
	#LogLevel INFO

	# Authentication:

	#LoginGraceTime 2m
	PermitRootLogin yes
	#StrictModes yes
	#MaxAuthTries 6
	#MaxSessions 10

	#PubkeyAuthentication yes

	# The default is to check both .ssh/authorized_keys and .ssh/authorized_keys2
	# but this is overridden so installations will only check .ssh/authorized_keys
	AuthorizedKeysFile	.ssh/authorized_keys

	#AuthorizedPrincipalsFile none
	
	#AuthorizedKeysCommand none
	#AuthorizedKeysCommandUser nobody

	# For this to work you will also need host keys in /etc/ssh/ssh_known_hosts
	#HostbasedAuthentication no
	# Change to yes if you don't trust ~/.ssh/known_hosts for
	# HostbasedAuthentication
	#IgnoreUserKnownHosts no
	# Don't read the user's ~/.rhosts and ~/.shosts files
	#IgnoreRhosts yes

	# Change to yes to enable built-in password authentication.
	# Note that passwords may also be accepted via KbdInteractiveAuthentication.
	#PasswordAuthentication no
	#PermitEmptyPasswords no

	# Change to no to disable PAM authentication
	#KbdInteractiveAuthentication yes

	# Kerberos options
	#KerberosAuthentication no
	#KerberosOrLocalPasswd yes
	#KerberosTicketCleanup yes
	#KerberosGetAFSToken no

	# GSSAPI options
	#GSSAPIAuthentication no
	#GSSAPICleanupCredentials yes

	# Set this to 'no' to disable PAM authentication, account processing,
	# and session processing. If this is enabled, PAM authentication will
	# be allowed through the KbdInteractiveAuthentication and
	# PasswordAuthentication.  Depending on your PAM configuration,
	# PAM authentication via KbdInteractiveAuthentication may bypass
	# the setting of "PermitRootLogin prohibit-password".
	# If you just want the PAM account and session checks to run without
	# PAM authentication, then enable this but set PasswordAuthentication
	# and KbdInteractiveAuthentication to 'no'.
	#UsePAM yes

	#AllowAgentForwarding yes
	#AllowTcpForwarding yes
	#GatewayPorts no
	#X11Forwarding no
	#X11DisplayOffset 10
	#X11UseLocalhost yes
	#PermitTTY yes
	#PrintMotd yes
	#PrintLastLog yes
	#TCPKeepAlive yes
	#PermitUserEnvironment no
	#Compression delayed
	#ClientAliveInterval 0
	#ClientAliveCountMax 3
	#UseDNS yes
	#PidFile /var/run/sshd.pid
	#MaxStartups 10:30:100
	#PermitTunnel no
	#ChrootDirectory none
	#UseBlacklist no
	#VersionAddendum FreeBSD-20231004

	# no default banner path
	#Banner none

	# override default of no subsystems
	Subsystem	sftp	/usr/libexec/sftp-server

	# Example of overriding settings on a per-user basis
	#Match User anoncvs
	#	X11Forwarding no
	#	AllowTcpForwarding no
	#	PermitTTY no
	#	ForceCommand cvs server


Step3:

	snort3 installation: pkg install snort3
	configuration: 	
	hostname="sq3"
	ifconfig_em0="DHCP"
	sshd_enable="YES"
	moused_nondefault_enable="NO"
	# Set dumpdev to "AUTO" to enable crash dumps, "NO" to disable
	dumpdev="AUTO"
	zfs_enable="YES"
	gateway_enable="YES"
	ifconfig_em1="inet 192.168.33.1 netmask 255.255.255.0"
	dnsmasq_enable="YES"
	pf_enable="YES"
	pflog_enable="YES"
	snort_enable="YES"
	snort_interface="em0"
	snort_conf="/usr/local/etc/snort/snort.lua"

Step4: Ensure snort can protect agains SMBGhost.
	
		

	
