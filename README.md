# Ansible-docker

Obiettivo : 

Realizzare un playbook Ansible che permetta di svolgere le seguenti attivita':

1. Provisioning di due VM CentOS. Le VM possono essere locali oppure su un
Cloud provider a scelta.

2. Configurare le VM:
a. Assicurarsi che la partizione utilizzata da Docker abbia almeno 40GB di
spazio disponibile, effettuando un opportuno resize.

3. Setup di Docker sulle VM

4. Configurare Docker:
a. Esporre le API REST del Docker Daemon in modo sicuro
b. Assicurarsi che il Docker Daemon sia configurato come un servizio che
parta automaticamente all'avvio del sistema

5. Configurare un Docker Swarm sulle VM, che sia accessibile in modo sicuro.
Assicurarsi di riuscire ad interagire e deployare servizi sullo Swarm dalla
macchina locale.

Opzionalmente:
6. Testare un task a scelta dei precedenti utilizzando Molecule

Svolgimento : 
La crezione delle vm verrà eseguito su un cluster proxmox all'interno della rete lan.
Per la realizzazione del cluster proxmox si può seguire tranquillamente la guida ufficiale di proxmox ricordandosi che il numero minimo di macchine da utilizzare per la sua creazione è pari a 3.
Per la parte di storage si è optato per l'integrazione all'interno di proxmox di un cluster ceph in modo da simulare un comportamento simile alla vsan di vmware.

Per realizzare il punto 1) il modo più pratico per effettuare un provisioning di una macchina CentOS è tramite l'utilizzo di kickstart,
che ci permetterà di creare macchine idempotenti fin dalla loro nascita.

Ansible lo useremo per dire ad un nodo di proxmox di creare la struttura delle vm e lanciare un'iso "cucinata" a dovere che faccia automaticamente partire un'installazione unattended andandosi a prelevare il file di kickstart da un server web oppure presente all'interno dell'iso stessa.

All'interno dell'iso bisogna editare il seguente file : ./EFI/BOOT/grub.cfg andando ad aggiungere una nuova menuentry

menuentry 'Install CentOS 7 - kickstart' --class fedora --class gnu-linux --class gnu --class os {
	linuxefi /images/pxeboot/vmlinuz inst.stage2=hd:LABEL=CentOS\x207\x20x86_64 quiet
	initrdefi /images/pxeboot/initrd.img inst.ks=http://<ipserverweb>/ks.cfg
}

Per creare il file di kickstart senza dover necessariamente scriverlo interamente da zero,
è possibile effettuare un'installazione manuale di CentOS quanto più simile possibile a ciò che vogliamo realizzare in termini di configurazione dischi (partizioni, dimensioni, file system), utenze di base (root e user), pacchetti in quanto al termine della stessa l'installatore Anaconda ci creerà dentro la cartella di root un file anaconda-ks.cfg che è appunto un riassunto di come rigenerare l'installazione della distro stessa identica e che sarà il nostro punto di partenza per ulteriori modifiche.

Con un'installazione minimale di CentOS verrà generato il seguente file di kickstart : 

auth --enableshadow --passalgo=sha512
cdrom
graphical
firstboot --enable
ignoredisk --only-use=sda
keyboard --vckeymap=it --xlayouts='it'
lang it_IT.UTF-8

network  --bootproto=static --device=eth0 --gateway=aaa.bbb.ccc.ddd --ip=aaa.bbb.ccc.ddd --nameserver=aaa.bbb.ccc.ddd --netmask=aaa.bbb.ccc.ddd--ipv6=auto --activate
network  --hostname=localhost.localdomain

rootpw --iscrypted $6$11UpsqSMu7tmUxYb$I9wtgMop1i2fExnYCIKxyL30YyHkZ202EqKKEnJmIBK07q8c2iW1QxF8gWvPvKwDjtrri.KJg2GMLqf4T8ubn.

services --enabled="chronyd"

timezone Europe/Rome --isUtc


bootloader --append=" crashkernel=auto" --location=mbr --boot-drive=sda
autopart --type=lvm

clearpart --none --initlabel

%packages
@^web-server-environment
@base
@core
@web-server
chrony
kexec-tools

%end

%addon com_redhat_kdump --enable --reserve-mb='auto'

%end

%anaconda
pwpolicy root --minlen=6 --minquality=1 --notstrict --nochanges --notempty
pwpolicy user --minlen=6 --minquality=1 --notstrict --nochanges --emptyok
pwpolicy luks --minlen=6 --minquality=1 --notstrict --nochanges --notempty
%end

Le primissime aggiunte da effettuare sono le seguenti : 

reboot

eula --agreed


%post

cat >/etc/yum.repos.d/docker-ce.repo <<EOF
[docker-ce-stable]
name=Docker CE Stable - $basearch
baseurl=https://download.docker.com/linux/centos/7/$basearch/stable
enabled=1
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-stable-debuginfo]
name=Docker CE Stable - Debuginfo $basearch
baseurl=https://download.docker.com/linux/centos/7/debug-$basearch/stable
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-stable-source]
name=Docker CE Stable - Sources
baseurl=https://download.docker.com/linux/centos/7/source/stable
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-edge]
name=Docker CE Edge - $basearch
baseurl=https://download.docker.com/linux/centos/7/$basearch/edge
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-edge-debuginfo]
name=Docker CE Edge - Debuginfo $basearch
baseurl=https://download.docker.com/linux/centos/7/debug-$basearch/edge
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-edge-source]
name=Docker CE Edge - Sources
baseurl=https://download.docker.com/linux/centos/7/source/edge
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-test]
name=Docker CE Test - $basearch
baseurl=https://download.docker.com/linux/centos/7/$basearch/test
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-test-debuginfo]
name=Docker CE Test - Debuginfo $basearch
baseurl=https://download.docker.com/linux/centos/7/debug-$basearch/test
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-test-source]
name=Docker CE Test - Sources
baseurl=https://download.docker.com/linux/centos/7/source/test
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-nightly]
name=Docker CE Nightly - $basearch
baseurl=https://download.docker.com/linux/centos/7/$basearch/nightly
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-nightly-debuginfo]
name=Docker CE Nightly - Debuginfo $basearch
baseurl=https://download.docker.com/linux/centos/7/debug-$basearch/nightly
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-nightly-source]
name=Docker CE Nightly - Sources
baseurl=https://download.docker.com/linux/centos/7/source/nightly
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

EOF

yum -y update
%end
