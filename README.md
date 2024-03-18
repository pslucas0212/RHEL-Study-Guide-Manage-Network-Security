# RHEL Study Guide: Manage Network Security

[RHEL Study Guide - Table of Contents](https://github.com/pslucas0212/RHEL-Study-Guide) 

In this section we learn about using SELinux with network connectivity and the firewall.

First let's see if we can access the http server on port 80 and port 1001
```
$ curl http://serverb.lab.example.com
curl: (7) Failed to connect to serverb.lab.example.com port 80: No route to host
```
Try port 1001
```
$ curl http://serverb.lab.example.com:1001
curl: (7) Failed to connect to serverb.lab.example.com port 1001: No route to host
```

Let's see why we can connect via either port 80 or port 1001 on serverb.  SSH to serverb and raise your privelage to root via sudo -i

First let's check if httpd is active.
```
# systemctl is-active httpd
inactive
```

Since its not active let's try to enable and start httpd.
```
# systemctl enable --now httpd
Created symlink /etc/systemd/system/multi-user.target.wants/httpd.service → /usr/lib/systemd/system/httpd.service.
Job for httpd.service failed because the control process exited with error code.
See "systemctl status httpd.service" and "journalctl -xeu httpd.service" for details.
```
httpd won't start up we can see the reason a couple of ways.  Let's use systecmctl status first.  Note you may have to make the terminal window as wide as possible to see the full message.
```
# systemctl status httpd
× httpd.service - The Apache HTTP Server
     Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled; vendor preset: disabled)
     Active: failed (Result: exit-code) since Mon 2024-03-18 11:36:27 EDT; 1min 30s ago
       Docs: man:httpd.service(8)
    Process: 1605 ExecStart=/usr/sbin/httpd $OPTIONS -DFOREGROUND (code=exited, status=1/FAILURE)
   Main PID: 1605 (code=exited, status=1/FAILURE)
     Status: "Reading configuration..."
        CPU: 34ms

Mar 18 11:36:27 serverb.lab.example.com systemd[1]: Starting The Apache HTTP Server...
Mar 18 11:36:27 serverb.lab.example.com httpd[1605]: (13)Permission denied: AH00072: make_sock: could not bind to address [::]:1001
Mar 18 11:36:27 serverb.lab.example.com httpd[1605]: (13)Permission denied: AH00072: make_sock: could not bind to address 0.0.0.0:1001
Mar 18 11:36:27 serverb.lab.example.com httpd[1605]: no listening sockets available, shutting down
Mar 18 11:36:27 serverb.lab.example.com httpd[1605]: AH00015: Unable to open logs
Mar 18 11:36:27 serverb.lab.example.com systemd[1]: httpd.service: Main process exited, code=exited, status=1/FAILURE
Mar 18 11:36:27 serverb.lab.example.com systemd[1]: httpd.service: Failed with result 'exit-code'.
Mar 18 11:36:27 serverb.lab.example.com systemd[1]: Failed to start The Apache HTTP Server.
[root@serverb ~]# 
```
You can also use journalctl.  The messages are similar.
```
# journalctl -xeu httpd.service
░░ 
░░ The job identifier is 2167.
Mar 18 11:38:52 serverb.lab.example.com httpd[1662]: (13)Permission denied: AH00072: make_sock: could not bind to address [::]:1001
Mar 18 11:38:52 serverb.lab.example.com httpd[1662]: (13)Permission denied: AH00072: make_sock: could not bind to address 0.0.0.0:1001
Mar 18 11:38:52 serverb.lab.example.com httpd[1662]: no listening sockets available, shutting down
Mar 18 11:38:52 serverb.lab.example.com httpd[1662]: AH00015: Unable to open logs
Mar 18 11:38:52 serverb.lab.example.com systemd[1]: httpd.service: Main process exited, code=exited, status=1/FAILURE
░░ Subject: Unit process exited
░░ Defined-By: systemd
░░ Support: https://access.redhat.com/support
░░ 
░░ An ExecStart= process belonging to unit httpd.service has exited.
░░ 
░░ The process' exit code is 'exited' and its exit status is 1.
Mar 18 11:38:52 serverb.lab.example.com systemd[1]: httpd.service: Failed with result 'exit-code'.
░░ Subject: Unit failed
░░ Defined-By: systemd
░░ Support: https://access.redhat.com/support
░░ 
░░ The unit httpd.service has entered the 'failed' state with result 'exit-code'.
Mar 18 11:38:52 serverb.lab.example.com systemd[1]: Failed to start The Apache HTTP Server.
░░ Subject: A start job for unit httpd.service has failed
░░ Defined-By: systemd
░░ Support: https://access.redhat.com/support
░░ 
░░ A start job for unit httpd.service has finished with a failure.
░░ 
░░ The job identifier is 2167 and the job result is failed.
lines 41-68/68 (END)
```

Why can't httpd bind to port 1001? Maybe its an SELinux setting.  Let's use sealert -a with the audit.log to see what information we can learn.
```
# sealert -a /var/log/audit/audit.log 
100% done
found 1 alerts in /var/log/audit/audit.log
--------------------------------------------------------------------------------

SELinux is preventing /usr/sbin/httpd from name_bind access on the tcp_socket port 1001.

*****  Plugin bind_ports (99.5 confidence) suggests   ************************

If you want to allow /usr/sbin/httpd to bind to network port 1001
Then you need to modify the port type.
Do
# semanage port -a -t PORT_TYPE -p tcp 1001
    where PORT_TYPE is one of the following: http_cache_port_t, http_port_t, jboss_management_port_t, jboss_messaging_port_t, ntop_port_t, puppet_port_t.

*****  Plugin catchall (1.49 confidence) suggests   **************************

If you believe that httpd should be allowed name_bind access on the port 1001 tcp_socket by default.
Then you should report this as a bug.
You can generate a local policy module to allow this access.
Do
allow this access for now by executing:
# ausearch -c 'httpd' --raw | audit2allow -M my-httpd
# semodule -X 300 -i my-httpd.pp


Additional Information:
Source Context                system_u:system_r:httpd_t:s0
Target Context                system_u:object_r:hi_reserved_port_t:s0
Target Objects                port 1001 [ tcp_socket ]
Source                        httpd
Source Path                   /usr/sbin/httpd
Port                          1001
Host                          <Unknown>
Source RPM Packages           httpd-2.4.51-7.el9_0.x86_64
Target RPM Packages           
SELinux Policy RPM            selinux-policy-targeted-34.1.29-1.el9_0.noarch
Local Policy RPM              selinux-policy-targeted-34.1.29-1.el9_0.noarch
Selinux Enabled               True
Policy Type                   targeted
Enforcing Mode                Enforcing
Host Name                     serverb.lab.example.com
Platform                      Linux serverb.lab.example.com
                              5.14.0-70.13.1.el9_0.x86_64 #1 SMP PREEMPT Thu Apr
                              14 12:42:38 EDT 2022 x86_64 x86_64
Alert Count                   10
First Seen                    2024-03-14 17:30:10 EDT
Last Seen                     2024-03-18 11:38:52 EDT
Local ID                      e5bfafac-befc-49b5-9cf4-d48b155344f2

Raw Audit Messages
type=AVC msg=audit(1710776332.948:364): avc:  denied  { name_bind } for  pid=1662 comm="httpd" src=1001 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:hi_reserved_port_t:s0 tclass=tcp_socket permissive=0


type=SYSCALL msg=audit(1710776332.948:364): arch=x86_64 syscall=bind success=no exit=EACCES a0=5 a1=5565e03e5ea8 a2=10 a3=7ffffc2958ec items=0 ppid=1 pid=1662 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm=httpd exe=/usr/sbin/httpd subj=system_u:system_r:httpd_t:s0 key=(null)ARCH=x86_64 SYSCALL=bind AUID=unset UID=root GID=root EUID=root SUID=root FSUID=root EGID=root SGID=root FSGID=root

Hash: httpd,httpd_t,hi_reserved_port_t,tcp_socket,name_bind
```

We see might need to enable SELinux to allow httpd to use port 1001.  We can use semanage port --list | grep http to see what ports SELinux is allowing httpd to access.
```
# semanage port --list | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```

We need to enable port 1001 with SELinux
```
# semanage port -a -t http_port_t -p tcp 1001
#
```

Let's check one more time to see if the port is now "open"
```
# semanage port -l | grep http_port_t
http_port_t                    tcp      1001, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
```

Let's see if we can start httpd
```
# systemctl enable --now httpd
# 
```
Perfect httpd started.  Let's see if can access ports 80 and 1001.
```
$ curl http://serverb.lab.example.com/
SERVER B
```
Port 80 is good to go.  Let's try port 1001
```
$ curl http://serverb.lab.example.com:1001
curl: (7) Failed to connect to serverb.lab.example.com port 1001: No route to host
```

Port 1001 is stil an issue.  Let's check the firewall.  First we will see what the default-zone is set to...
```
# firewall-cmd --get-default-zone
public
```

All good here.  Let's check active ports
```
# firewall-cmd --zone=public --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: eth0
  sources: 
  services: cockpit dhcpv6-client http ssh
  ports: 
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules:
```

Notice 1001 is missing from the ports section.  Let's add port 1001 to the firewall configuration
```
# firewall-cmd --zone=public --permanent --add-port=1001/tcp
success
```
Let's reload the firewall and the check the settings
```
# firewall-cmd --reload
success
# firewall-cmd --zone=public --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: eth0
  sources: 
  services: cockpit dhcpv6-client http ssh
  ports: 1001/tcp
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
```

We should be good to go.  Let's see if we can acces both ports 80 and 1001 now.
```
$ curl http://serverb.lab.example.com/
SERVER B
$ curl http://serverb.lab.example.com:1001
VHOST 1
```
All good.




