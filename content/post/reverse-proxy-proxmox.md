---
title: "Apache HTTP Server como proxy reverso para o Proxmox VE"
description: "Configurando um proxy reverso no apache para acesso ao dashboard do Proxmox VE"
date: 2020-02-11T16:47:41-03:00
draft: false
author: "Ewerton Silva"
tags: ["proxmox","apache","dicas"]
categories: ["proxmox"]
---

Quem me conhece sabe que gosto demais do Proxmox Virtual Environment (PVE para os íntimos), porque ele proporciona uma plataforma completa de virtualização de máquinas virtuais de forma simples, robusta e OpenSource, garantindo inclusive suporte para uma arquitetura de Virtualização Hiperconvergente. [Clique aqui](https://www.proxmox.com/en/proxmox-ve "Link para o site oficial do Proxmox VE") para saber mais sobre PVE. 

Atualmente não o utilizo em produção, apenas em ambiente de estudos. Quem me dera, hein, brincar com esse cara em produção.

Conforme o título desse artigo, o meu objetivo é mostrar como configurar um proxy reverso afim de evitar ficar acessando o dashboard do Proxmox via **IP + Porta** ou **Hostname + Porta**. Chato demais acessar qualquer serviço dessa forma.

De momento irei utilizar o Apache HTTP Server, porque sim, porque eu gosto desse carinha. Num futuro próximo posso demonstrar utilizando o Nginx (pronuncia-se "enginex").

Para este artigo utilizei uma VM bem simples com as seguintes especificações:

- CentOS Linux release 7.7.1908 (Core)
- 1 GB de memória RAM
- 1 vCPU
- 10 GB de espaço em disco

#### Pré-requesitos ####

Logado via SSH como **root** numa VM zerada, com o SO recém instalado, executo alguns pré-requesitos. Como:

- Alterar o estado do [SELinux](https://www.redhat.com/pt-br/topics/linux/what-is-selinux "Link para saber mais sobre SELinux"), porque não quero trabalhar com esse cara habilitado. Nesse caso mudo de ```enforcing``` para ```permissive```.

```bash
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=permissive/g' /etc/selinux/config
```

- Criar regra no **firewalld** liberando as portas **80/HTTP** e **443/HTTPS**.

```bash
firewall-cmd --permanent --add-service={http,https}
firewall-cmd --reload
```

- Adicionar um hostname adequado à função da VM. Pensei aqui em ```proxy-server```. Combina.

```bash
hostnamectl set-hostname proxy-server && bash
```

- Ajustar o arquivo ```/etc/hosts``` com o IP e hostname correto da VM. No meu caso o IP é ```10.10.10.10``` e domínio ```meudominio.local```. O conteúdo do arquivo ficou da seguinte forma:

```bash
127.0.0.1	localhost
::1         localhost
10.10.10.10	proxy-server.meudominio.local proxy-server
```

- Atualizar para a última versão os pacotes instalados.

```bash
yum update -y
```

#### Instalação e configuração do serviço httpd ####

Agora que executei os pré-requesitos acima, posso finalmente instalar os pacotes necessários.

```bash
yum install httpd mod_ssl -y
```

Inicializo o serviço **httpd**.

```bash
systemctl enable --now httpd
```

Vou criar o diretório ```/etc/httpd/ssl``` para armazenar os arquivos do certificado SSL. Isso porque irei utilizar um certificado auto-assinado. No seu caso, se for [Let's Encrypt](https://letsencrypt.org/pt-br/ "Link para acesso ao site da Let's Encrypt") já não precisa, pois o diretório por padrão será outro.

```bash
mkdir /etc/httpd/ssl
```

#### Criação do certificado auto-assinado ####

Execute o comando abaixo para gerar um certificado auto-assinado.

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/httpd/ssl/pve.key -out /etc/httpd/ssl/pve.crt
```

Preencha os campos solicitados, mas **atenção** na opção **Common Name**, pois ela precisa ser igual ao endereço que você pretende informar no browser. No meu caso, ```pve.meudominio.local```.

> [Clique aqui](https://www.digitalocean.com/community/tutorials/how-to-create-a-self-signed-ssl-certificate-for-apache-in-ubuntu-18-04-pt "Link de acesso a um tutorial no site da Digital Ocean") caso você tenha dúvidas sobre o processo acima de gerar o certificado auto-assinado.

O resultado foi a criação de dois arquivos:

```bash
[root@proxy-server ~]# ls -l /etc/httpd/ssl/
total 8
-rw-r--r--. 1 root root 1505 Feb 11 23:57 pve.crt
-rw-r--r--. 1 root root 1704 Feb 11 23:57 pve.key
```

Sensacional. Agora temos tudo para criar o arquivo de Virtual Host.

#### Criando o arquivo de Virtual Host ####

Crie o arquivo ```/etc/httpd/conf.d/pve.conf``` com o conteúdo abaixo.

```html
<VirtualHost *:80>
    ServerName pve.meudominio.local
    ServerAlias pve
    ServerAdmin sysadmin@meudominio.local
    
    Redirect Permanent / https://pve.meudominio.local/
</VirtualHost>

<VirtualHost *:443>
    Servername pve.meudominio.local
    ServerAlias pve
    ServerAdmin sysadmin@meudominio.local
    
    ErrorLog logs/pve.meudominio.local_error.log
    CustomLog logs/pve.meudominio.local_access.log combined

    SSLProxyEngine On
    SSLProxyVerify None
    SSLProxyCheckPeerCN Off
    SSLProxyCheckPeerName Off
    RequestHeader unset Accept-Encoding

    ProxyPreserveHost On
    ProxyRequests Off
    ProxyTimeout 600
    Timeout 600

    <Location />
           ProxyPass https://10.10.1.122:8006/
           ProxyPassReverse https://10.10.1.122:8006/
    </Location>

    <LocationMatch ^/(api2/json/nodes/[^\/]+/[^\/]+/[^\/]+/vncwebsocket.*)$>
           ProxyPass wss://10.10.1.122:8006/$1 retry=0
    </LocationMatch>

    <Location /websockify>
           ProxyPass ws://10.10.1.122:8006
           ProxyPassReverse ws://10.10.1.122:8006
    </Location>

    SSLEngine On
    SSLCipherSuite EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH
    SSLProtocol All -SSLv2 -SSLv3 -TLSv1 -TLSv1.1
    SSLHonorCipherOrder On
    
    SSLCertificateKeyFile /etc/httpd/ssl/pve.key
    SSLCertificateFile /etc/httpd/ssl/pve.crt
</VirtualHost>
```

**Primeira Observação:** No arquivo acima o IP ```10.10.1.122``` é o meu servidor Proxmox. Você precisa então substituir pelo IP do seu Proxmox.

**Segunda Observação:** O nome ```pve.meudominio.local``` está configurado no arquivo ```/etc/hosts``` do meu notebook. No seu caso o nome escolhido precisa estar em conformidade com o seu DNS público e/ou local.

Feito todos os passos acima, já posso abrir o meu browser e digitar ```http://pve.meudominio.local``` que terei acesso ao dashboard do Proxmox com tudo funcional, inclusive **spice** e **websockets**.

**Terceira Observação:** Por se tratar de um certificado auto-assinado, no primeiro acesso teremos aquela puxada de orelha do browser informando que o certificado do site não é confiável. Normal.

Olha só como ficou no browser. Bem melhor que digitar ```https://10.10.1.122:8006/```, hein.

![pve.meudominio.local](https://raw.githubusercontent.com/ewerton-silva00/blog-sysadmin-linux/master/static/images/screenshot-dashboard-pve.png "Screenshot do dashboard do Proxmox VE")

Aqui te mostrei o básico para criar um proxy reverso utilizando o Apache HTTP Server num servidor CentOS 7.x.

Se preciso for, o apache também pode ser instalado e configurado no próprio servidor do Proxmox, mas nesse caso mudam alguns dos passos acima, pois será trabalhado o pacote ```apache2``` num sistema operacional ```Debian```. Assunto para um próximo artigo.

Espero que aproveite todo esse conteúdo. Abraço!