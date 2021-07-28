# Prática de Ataques em um ambiente Virtualizado - Disciplina de Mestrado em Ciência da Computação

Esse é um projeto para a realização de ataques em um ambiente virtualizado, o objetivo é explorar vulnerabilidades de um nó vulnerável de uma rede interna usando ferramentas disponíveis na distribuição Kali Linux e Metasploit. A Figura 1 ilustra o cenário de rede no qual foram realizados os ataques.

<p>
  <img src="images/setup.png" alt="Cenário proposto" style="width:100%">
  <p align="center">Figura 1 - Cenário proposto</p>
</p>
<br>

# Configuração do cenário
Todos os nós desse experimento usam a imagem do sistema operacional Ubuntu 18.04 (server), todos eles possuem um servidor http, ftp e telnet.

Na instalação base de totdos os nós, foram executados os seguintes comandos:
```bash
sudo apt install apache2
sudo apt install openssh-server
sudo apt install lynx
sudo apt install ftp
sudo apt install iptables
sudo apt install bind9
sudo apt install proftpd
sudo apt install telnetd
```

Nas máquinas da rede local (host1a e host1b), o nome associada a placa de rede é intnet1, nas máquinas da DMZ (host2a, host2b) o nome será intnet2, na placa de rede interna da WAN (host3a) estará conectada a uma Rede NAT, por fim, na VM do Firewall tem-se a primeira placa de rede interna na intnet1, a segunda na intnet2 e a terceira com a mesma Rede NAT do host3a. A tabela 1 apresenta as redes internas das VM's. 

| VM        | Redes Internas | 
| ------------- |:-------------:| 
| host1a (LAN)        | intnet1 |
| host1b (LAN)     | intnet1     |  
| host2a (DMZ) | intnet2      |
| host2b (vulnerable) | intnet2      |
| host3a (WAN) | Rede NAT |
| Firewall | intnet1, intnet2, Rede NAT |    
 <p align="center">Tabela 1 - Placas de rede </p>

 A seguir são apresentadas as configurações de rede de todos os nós do experimeto.
 ## Arquivo /etc/netplan/00-installer-config.yaml do host1a
```bash
network:
  ethernets:
    enp0s9:
      dhcp4: no
      addresses: [172.16.1.1/24]
      gateway4: 172.16.1.254
      nameservers:
        addresses: [8.8.8.8]  
  version: 2
```
## Arquivo /etc/netplan/00-installer-config.yaml do host1b
```bash
  network:
    ethernets:
      enp0s9:
        dhcp4: no
        addresses: [172.16.1.2/24]
        gateway4: 172.16.1.254
        nameservers:
          addresses: [8.8.8.8]
    version: 2
```
## Arquivo /etc/netplan/00-installer-config.yaml do host2a
```bash
network:
  ethernets:
    enp0s9:
      dhcp4: no
      addresses: [172.16.2.3/24]
      gateway4: 172.16.2.254
      nameservers:
        addresses: [8.8.8.8]
  version: 2
```

## Arquivo /etc/netplan/00-installer-config.yaml do host2b(vulneravel)
```bash
network:
  ethernets:
    enp0s9:
      dhcp4: no
      addresses: [172.16.2.2/24]
      gateway4: 172.16.2.254
      nameservers:
        addresses: [8.8.8.8]
  version: 2
```

## Arquivo /etc/netplan/00-installer-config.yaml do host3a
```bash
network:
  ethernets:
    enp0s3:
      dhcp4: true    
  version: 2
```
## Arquivo /etc/netplan/00-installer-config.yaml do Firewall
```bash
network:
  ethernets:
    enp0s8:// WAN/INTERNET
      dhcp4: true
    enp0s9: //LAN
      dhcp4: no
      addresses: [172.16.1.254/24]
    enp0s10://DMZ
      dhcp4: no
      addresses: [172.16.2.254/24]
    version: 2
```

## Ligando roteamento  no firewall
Agora ligaremos ligaremos o modo de roteamento para o firewall, nessa configuração o Firewall é quem possui uma interface com acesso à Internet, e ele que de fato irá ceder aos nós da LAN e DMZ. Para isso, no firewall e no host3a abra o arquivo /etc/sysctl.conf, e acrescente a linha abaixo:
```bash
net.ipv4.ip_forward=1
```

# Configuração do Firewall
Nessa pratica o objetivo é atacar uma rede vulnerável, por esse motivo o Firewall terá políticas brandas de segurança, ele só pode ser acessado pelo host da lan 172.16.1.1

```bash
echo "limpando as tabelas iptables"
iptables -t nat -F
iptables -F

echo "ligando mascaramento para tudo que sair para internet"
iptables -t nat -A POSTROUTING -o enp0s8 -j MASQUERADE

echo "Permitindo acesso ssh pelo host1a"
iptables -A INPUT -p tcp --dport 22 -i enp0s9 -s 172.16.1.1 -j ACCEPT
iptables -A INPUT -j DROP

echo "Permitindo a passagem de trafego encaminhado pelo firewall"
iptables -A FORWARD -j ACCEPT

echo "Permitindo a passagem de trafego que sai do Firewall pelo firewall"
iptables -A OUTPUT -j ACCEPT
```

# Configurações específicas do nó vulnerável (host2a)
