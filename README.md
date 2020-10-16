# SoftEtherVPN
automated convenience script (This script is outdated )

VPN server
##How to setup a VPN server using Soft-Ether VPN on CentOS 7


Windows, Linux, Mac, Android, iPhone, iPad and Windows Mobile are supported. SSL-VPN (HTTPS) and 6 major VPN protocols (OpenVPN, IPsec, L2TP, MS-SSTP, L2TPv3 and EtherIP) are all supported as VPN tunneling underlay protocols.



##OS Requirements
To install SoftEther VPN, you need a maintained version of CentOS 7. Archived versions aren’t supported or tested.

###Install SoftEther VPN 
It is very handy to use automated convenience script when installing vpn server on a new host machine. 

As a user with root privileges, write the following lines into a file and save it with the file name [softether-install.sh] and Run this command on the command line on the machine to install SoftEther VPN.

sh softether-install.sh




    #!/bin/sh
    set -e
    
    if [ $EUID -ne 0 ] ; then
       echo "This script must be run as root" 
       exit 1
    fi
    
    if [ `firewall-cmd --state` = running ] > /dev/null 2>&1 ; then
        systemctl stop firewalld && systemctl disable firewalld && systemctl mask --now firewalld
    fi
    
    yum -y update
    
    yum -y install \
    @'Development Tools' ncurses-devel openssl-devel readline-devel epel-release  curl wget tar
    
    yum -y install certbot
    echo "0 0,12 * * * root python -c 'import random; import time; time.sleep(random.random() * 3600)' && certbot renew" | sudo tee -a /etc/crontab > /dev/null
    
    curl -s https://api.github.com/repos/SoftEtherVPN/SoftEtherVPN_Stable/releases/latest \
    | grep "browser_download_url.*vpnserver-[^extended].*-linux-x64-64bit\.tar\.gz" \
    | cut -d : -f 2,3 \
    | tr -d \" \
    | wget -i - -O - \
    | tar -xvz -C /usr/local \
    && printf '1\n1\n1\n' | make -i --directory=/usr/local/vpnserver 
    
    ( cd /usr/local/vpnserver
    chmod 600 * && chmod 700 vpncmd vpnserver )
    
    echo '[Unit]
    Description=SoftEther VPN Server
    After=network.target auditd.service
    
    [Service]
    Type=forking
    EnvironmentFile=-/usr/local/vpnserver
    ExecStart=/usr/local/vpnserver/vpnserver start
    ExecStop=/usr/local/vpnserver/vpnserver stop
    KillMode=process
    Restart=on-failure
    
    [Install]
    WantedBy=multi-user.target' | sudo tee /usr/lib/systemd/system/vpnserver.service > /dev/null
    
    chmod 644 /usr/lib/systemd/system/vpnserver.service
    
    /usr/local/vpnserver/vpnserver start


###Configure SoftEther VPN to start on boot
Most current Linux distributions (RHEL, CentOS, Fedora, Ubuntu 16.04 and higher) use systemd to manage which services start when the system boots. [ this should be add to the script instead of being seprated ]

    # systemctl start vpnserver
    # systemctl enable vpnserver




* [1] This script is carefully written to follow all instructions provided in the SoftEther VPN man page.
* [2] you need to manually read License Agreement in the /usr/local/vpnserver directory.





###Initial Configurations
After VPN Server is installed, there are several settings that first must be configured. This section describes how to configure these settings with examples of the settings when using vpncmd.

Write the following lines into a file and save it with the file name [sampleConfig.txt]

    ServerPasswordSet admin1
    ListenerCreate 1197
    HubCreate eth /PASSWORD:none
    VpnOverIcmpDnsEnable /ICMP:no /DNS:no
    IPsecEnable /L2TP:yes /L2TPRAW:yes /ETHERIP:no /PSK:vpn /DEFAULTHUB:eth
    ServerCertSet /LOADCERT:/etc/letsencrypt/live/SA/fullchain.pem /LOADKEY:/etc/letsencrypt/live/SA/privkey.pem
    SstpEnable yes 
    OpenVpnEnable yes /PORTS:"1194, 1197"
    OpenVpnMakeConfig client-conf.zip
    Hub eth
    SecureNatEnable
    DhcpSet /START:192.168.30.10 /END:192.168.30.255 /MASK:255.255.255.0 /EXPIRE:7200 /GW:192.168.30.1 /DNS:8.8.8.8 /DNS2:8.8.4.4 /DOMAIN:none /LOG:yes
    UserCreate kian /GROUP:none /REALNAME:none /NOTE:none
    UserPasswordSet kian /PASSWORD:8h2ysd


* [1] to learn more about these commands please visit Softether VPN man page.
* [2] client-conf.zip can be found under /root directory

Update configuration file by using vpncmd with following command line arguments.

    # /usr/local/vpnserver/vpncmd /server localhost /in:sampleConfig.txt



You also need to setup https if you dont want to manually port ssl certificate to every client for ms sstp.

(Skip this step if you are using a domain name ) You can register a hostname on the DNS record of ".softether.net".  free of charge (plz visit here)

 /usr/local/vpnserver/vpncmd /server localhost /CMD DynamicDnsSetHostname hostname

Use certbot tool to Install Let’s Encrypt certificate on this VPN Server.

    # certbot certonly --agree-tos --standalone -d hostname.softether.net--register-unsafely-without-email --cert-name SA

* [1] hostname.softether.net have to be use as destination on sstp client.
