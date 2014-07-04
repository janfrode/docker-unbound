
## Howto run unbound in docker, and route traffic to the containers using LVS

### Pull down unbound container

	docker pull janfrode/unbound

### Configure systemd to run containers

First we start two unbound instances on exposed on high ports:

	# vi /etc/systemd/system/docker-unbound-5354.service
	[Unit]
	Description=unbound in docker on port 5354
	After=docker.service
	Requires=docker.service

	[Service]
	Restart=always
	ExecStart=/usr/bin/docker run --privileged=true --rm=true -p 5354:53 -p 5354:53/udp janfrode/unbound

	[Install]
	WantedBy=multi-user.target

	# vi /etc/systemd/system/docker-unbound-5355.service
	[Unit]
	Description=unbound in docker on port 5355
	After=docker.service
	Requires=docker.service

	[Service]
	Restart=always
	ExecStart=/usr/bin/docker run --privileged=true --rm=true -p 5355:53 -p 5355:53/udp janfrode/unbound

	[Install]
	WantedBy=multi-user.target

	# systemctl daemon-reload
	# systemctl enable docker-unbound-5354
	# systemctl start docker-unbound-5354
	# systemctl enable docker-unbound-5355
	# systemctl start docker-unbound-5355

	# docker ps
	CONTAINER ID        IMAGE                     COMMAND                CREATED             STATUS              PORTS                                        NAMES
	fc73158dcd64        janfrode/unbound:latest   /usr/sbin/unbound -v   12 seconds ago      Up 11 seconds       0.0.0.0:5355->53/tcp, 0.0.0.0:5355->53/udp   condescending_almeida9   
	82501f484c2f        janfrode/unbound:latest   /usr/sbin/unbound -v   16 seconds ago      Up 15 seconds       0.0.0.0:5354->53/tcp, 0.0.0.0:5354->53/udp   tender_ritchie7          


And both works, good:

	# dig unbound.net -p 5354 @127.0.0.1 +short
	213.154.224.1
	# dig unbound.net -p 5355 @127.0.0.1 +short
	213.154.224.1


### Configure keepalived/LVS

Then we install keepalived to manage failover ip-addresses and LVS services:


	# yum install keepalived ipvsadm
	================================================================================
 	Package                Arch      Version           Repository             Size
	================================================================================
	Installing:
 	ipvsadm                x86_64    1.27-4.el7        rhel-7-server-rpms     44 k
 	keepalived             x86_64    1.2.10-2.el7      rhel-7-server-rpms    218 k
	Installing for dependencies:
 	lm_sensors-libs        x86_64    3.3.4-10.el7      rhel-7-server-rpms     40 k
 	net-snmp-agent-libs    x86_64    1:5.7.2-18.el7    rhel-7-server-rpms    698 k
 	net-snmp-libs          x86_64    1:5.7.2-18.el7    rhel-7-server-rpms    745 k

	Transaction Summary
	================================================================================
	Install  2 Packages (+3 Dependent packages)


	# vi /etc/keepalived/keepalived.conf

	global_defs {
   		notification_email {
     			janfrode@tanso.net
   		}
   		notification_email_from janfrode@tanso.net
   		smtp_server smtp.altibox.no
   		smtp_connect_timeout 30
	}

	vrrp_instance DNS_VIP {
    		state BACKUP
    		interface enp3s0
    		virtual_router_id 144
    		priority 50
    		advert_int 1
    		smtp_alert
    		authentication {
        		auth_type PASS
        		auth_pass exfdjkfjkfc9
    		}
    		virtual_ipaddress {
        		213.167.104.144
    		}
	}

	virtual_server 213.167.104.144 53 {
   		delay_loop 10
   		lb_algo wrr
   		lb_kind DR
   		protocol UDP
   		real_server 213.167.104.142 5354 {
       			weight 1
       			MISC_CHECK {
           			misc_path "/usr/bin/dig unbound.net  +time=1 +tries=5 +fail +noall +short -p 5354 @213.167.104.142 > /dev/null"
           			misc_timeout 6
       			}
   		}
   		real_server 213.167.104.142 5355 {
       			weight 1
       			MISC_CHECK {
           			misc_path "/usr/bin/dig unbound.net  +time=1 +tries=5 +fail +noall +short -p 5355 @213.167.104.142 > /dev/null"
           			misc_timeout 6
       			}
   		}
	}


	# systemctl start keepalived

	# ipvsadm -S -n
	-A -u 213.167.104.144:53 -s wrr
	-a -u 213.167.104.144:53 -r 213.167.104.142:5354 -g -w 1
	-a -u 213.167.104.144:53 -r 213.167.104.142:5355 -g -w 1


### Configure virtual_server address on loopbacks

Enter each of the containers using nsenter:

	[root@tcpip ~]# nsenter -m -u -i -n -p -t 7428 /bin/sh
	sh-4.2# ip address add 213.167.104.144/32 dev lo
	sh-4.2# exit
	[root@tcpip ~]# nsenter -m -u -i -n -p -t 7507 /bin/sh
	sh-4.2# ip address add 213.167.104.144/32 dev lo
	sh-4.2# exit


### Testing

And it doesn't work... hmmm.... Tcpdump tells me packets are routed to the containers, and unbound replies:

	15:27:45.738352 IP (tos 0x0, ttl 64, id 59733, offset 0, flags [none], proto UDP (17), length 68)
    142.213-167-104.customer.lyse.net.56340 > 172.17.0.141.domain: [bad udp cksum 0xeb15 -> 0x52fc!] 5525+ [1au] A? unbound.net. ar: . OPT UDPsize=4096 (40)
	15:27:45.738360 IP (tos 0x0, ttl 64, id 59733, offset 0, flags [none], proto UDP (17), length 68)
    142.213-167-104.customer.lyse.net.56340 > 172.17.0.141.domain: [bad udp cksum 0xeb15 -> 0x52fc!] 5525+ [1au] A? unbound.net. ar: . OPT UDPsize=4096 (40)
	15:27:45.738489 IP (tos 0x0, ttl 64, id 59521, offset 0, flags [none], proto UDP (17), length 84)
    172.17.0.141.domain > 142.213-167-104.customer.lyse.net.56340: [bad udp cksum 0xeb25 -> 0x9bce!] 5525$ q: A? unbound.net. 1/0/1 unbound.net. [1h36m7s] A 213.154.224.1 ar: . OPT UDPsize=4096 (56)
	15:27:45.738489 IP (tos 0x0, ttl 64, id 59521, offset 0, flags [none], proto UDP (17), length 84)
    172.17.0.141.domain > 142.213-167-104.customer.lyse.net.56340: [bad udp cksum 0xeb25 -> 0x9bce!] 5525$ q: A? unbound.net. 1/0/1 unbound.net. [1h36m7s] A 213.154.224.1 ar: . OPT UDPsize=4096 (56)


But I see no replies on the client...

