---
title: "Provisionando um repositório privado para Vagrant Boxes"
description: "Provisionando um repositório privado para Vagrant Boxes com Apache HTTP Server"
author: "Ewerton Silva"
date: 2020-02-13T23:42:32-03:00
draft: false
tags: ["vagrant","apache"]
categories: ["vagrant"]
---

No [artigo anterior](http://localhost:1313/post/vagrant-box-customized/ "Link de acesso ao artigo anterior") demonstrei passo à passo como criar suas próprias Vagrant Boxes, do zero até a execução. Ficou bem interessante.

Neste artigo quero te mostrar como criar um repositório privado pra você publicar suas Boxes de forma controlada.

Aqui estou utilizando uma VM com as seguintes especificações:

- CentOS Linux release 7.7.1908 (Core)
- 1 GB de memória RAM
- 10 GB de armazenamento
- Usuário SSH com permissão de **sudo**



#### Executando alguns pré-requesitos ####

Antes de executar qualquer instalação e/ou configuração gosto de assegurar que alguns pré-requesitos sejam atendidos. Como:

- Ajustar o nome do servidor de acordo com a função à ele concedida.

```bash
hostnamectl set-hostname vagrant-repo && bash
```

- Editar o arquivo ```/etc/hosts``` para garantir que ele esteja semelhante a configuração abaixo.

```bash
127.0.0.1	localhost
::1         localhost
10.10.10.10	vagrant-repo.meudominio.local vagrant-repo
```

```10.10.10.10``` é o IP do meu servidor, ```vagrant-repo``` é o nome do servidor e ```meudominio.local``` é o domínio que estou utilizando para este artigo. Ajuste de acordo com o seu ambiente.

- Ajustar minhas configurações de localidades.

```bash
localectl set-locale LANG=pt_BR.UTF-8
```

- Garantir que o meu timezone esteja correto.

```bash
timedatectl set-timezeon "America/Recife"
```

- Alterar o estado do **SELinux** para **permissive**.

```bash
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=permissive/g' /etc/selinux/config
```

- Vou adiantar logo a liberação da porta ```80/HTTP``` no firewalld.

```bash
firewall-cmd --permanent --add-service=http
firewall-cmd --reload
```

Tudo certo. Agora posso continuar tranquilo.

#### Criando a estrutura de diretórios necessária ####

Preciso criar uma estrutura de diretórios na qual seja possível versionar posteriormente as Boxes de forma organizada.

```bash
mkdir -p /var/www/html/vagrant/boxes
```

Cada Box terá um diretório principal (raíz) e diversos subdiretórios que corresponderão por suas versões.

Para este artigo, por exemplo, estou utilizando uma Box chamada **centos-base-image**, que está na versão **20200213.0.0**.

> A versão acima nada mais é que ```MAJOR.MINOR.PATCH```.
>
> **MAJOR**: aqui utilizo a data de publicação da Box, semelhante ao que se pede no formato de data estabelecido pela [ISO 8601 Date And Time Format](https://www.iso.org/iso-8601-date-and-time-format.html "Link de acesso ao site oficial do ISO.ORG").
>
> **MINOR**: aqui são adições de funcionalidades, porém mantendo a compatibilidade.
>
> **PATCH**: esse representa correções de falhas mantendo a compatibilidade.
>
> Para mais informações sobre esse formato de versionamento é só consultar o manual [Versionamento Semântico 2.0.0](https://semver.org/lang/pt-BR/ "Link de acesso ao site do manual sobre Versionamento Semântico").

Com as informações acima devidamente explicadas, posso dar continuidade criando o diretório base da Box assim como seu subdiretório.

```bash
mkdir -p /var/www/html/vagrant/boxes/centos-base-image/v20200213.0.0
```

Faço o upload do arquivo ```centos-base-image.box``` para o repositório.

```bash
scp centos-base-image.box root@vagrant-repo.meudominio.local:/var/www/html/vagrant/boxes/centos-base-image/v20200213.0.0
```

No mesmo diretório da Box mantenho um arquivo contendo a hash [SHA256](https://pt.wikipedia.org/wiki/SHA-2 "Link para saber mais sobre SHA256") que será utilizada para validar o arquivo.

Para gerar a hash é simples. Basta executar o comando abaixo.

```bash
sha256sum centos-base-image.box
```

Na sequência basta pegar a saída do comando acima e criar um arquivo de mesmo nome da Box, mas com sufixo **.sha256**. No meu caso o arquivo que criei foi nomeado de ```centos-base-image.box.sha256``` cujo conteúdo é:

```bash
55dbddadfa7f169e569898bcfcef44c5cec3d679bf53fc0634fdcb2d11074700  centos-base-image.box
```

Para finalizar essa parte, preciso apenas criar um arquivo no formato JSON que servirá como um index contendo as informações de todas as Boxes disponíveis no repositório. Vou nomear o arquivo de ```index.json```.

Como só tenho uma Box no repositório até então, o arquivo ficará da seguinte forma

```json
{
    "name": "centos-base-image",
    "description": "This box contains CentOS 7.7.1908 (Core) 64-bit.",
    "versions": [
      {
        "version": "20200213.0.0",
        "providers": [
          {
            "name": "virtualbox",
            "url": "http://repo.meudominio.local/boxes/centos-base-image/v20200213.0.0/centos-base-image.box",
            "checksum_type": "sha256",
            "checksum": "55dbddadfa7f169e569898bcfcef44c5cec3d679bf53fc0634fdcb2d11074700"
          }
        ]
      }
    ]
}
```

> Perceba que acima, no campo **url**, utilizei o DNS ```repo.meudominio.local``` e não ```vagrant-repo.meudominio.local```, que é o nome do servidor. Gosto e acho importante trabalhar assim. Utilizarei esse nome mais na frente, na criação do Virtual Host.
>
> Com isso o nome ```repo.meudominio.local``` é um ```CNAME``` para ```vagrant-repo.meudominio.local```.
>
> Analise se isso faz sentido pra você.

Para finalizar esta parte temos a seguinte estrutura de diretórios.

```bash
/var/www/html/vagrant/
├── boxes
│   └── centos-base-image
│       └── v20200213.0.0
│           ├── centos-base-image.box
│           └── centos-base-image.box.sha256
└── index.json

3 directories, 3 files
```



#### Configuração do Virtual Host ####

Aqui o negócio é mais simples.

Basta eu criar o arquivo ```/etc/httpd/conf.d/vagrant.conf``` com o seguinte conteúdo:

```html
<VirtualHost *:80>
	ServerName repo.meudominio.local
	ServerAlias repo
	ServerAdmin sysadmin@meudominio.local
	DocumentRoot "/var/www/html/vagrant"

	ErrorLog logs/repo.meudominio.local_error.log
	CustomLog logs/repo.meudominio.local_access.log combined

	<IfModule dir_module>
		DirectoryIndex index.json
	</IfModule>
</VirtualHost>
```

Por enquanto vou brincar só na porta 80/HTTP mesmo.

Verifico se ficou tudo OK com a sintaxe do arquivo. Importante.

```bash
apachectl configtest
```

O resultado precisa ser ```Syntax OK```.

Na sequência, recarrego as configurações do Apache com o comando abaixo.

```bash
systemctl reload httpd
```

Agora é só chamar no browser.

![Imagem 00](https://raw.githubusercontent.com/ewerton-silva00/blog-sysadmin-linux/master/static/images/00-screenshot-private-vagrant-repository.png "Imagem com o resultado do funcionamento do repositorio no browser")



#### Baixando e executando a Box a partir do Vagrantfile ####

Para validar se ficou tudo OK mesmo, basta eu criar o Vagrantfile abaixo.

```bash
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  
  config.vm.define "centos-base-image" do |centos|
    centos.vm.box = "centos-base-image"
    centos.vm.box_version = "20200213.0.0"
    centos.vm.box_url = "http://repo.meudominio.local"
    centos.vm.box_download_checksum = "http://repo.meudominio.local/boxes/centos-base-image/v20200213.0.0/centos-base-image.box.sha256"
    centos.vm.box_download_checksum_type = "sha256"
    centos.vm.box_check_update = true
    centos.vm.hostname = "centos-tutorial"
    centos.vm.network "private_network", ip: "10.10.10.5"
    centos.vm.post_up_message = "Vagrant Box v20200213.0.0 - CentOS 7.7.1908 (Core)"

    centos.vm.provider "virtualbox" do |vb|
      vb.name = "vagrant-centos-tutorial-repo"
      vb.gui = false
      vb.cpus = 1
      vb.memory = 512
    end

    centos.vm.provision "shell", inline: <<-SHELL
     getenforce
     firewall-cmd --state
     SHELL
  end
end
```

Executo um teste de validação de sintaxe. Importante.

```bash
vagrant validate
```

O resultado do comando acima precisa ser ```Vagrantfile validated successfully.```

Agora é só executar o comando ```vagrant up```.

Abaixo o resultado da execução do comando acima.

![Imagem 01](https://raw.githubusercontent.com/ewerton-silva00/blog-sysadmin-linux/master/static/images/01-screenshot-private-vagrant-repository.png "Screenshot da execução do comando vagrant up")

Só sucesso. Nas próximas execuções dessa Box não será mais necessário baixar a imagem do repositório, porque a Box já foi importada para o repositório local. Esse fluxo pode ser comparado a trabalhar com imagens Docker.



#### Versionando as Boxes ####

Então, para versionar as Boxes é simples, basta seguir o mesmo fluxo que fiz neste artigo, porém, acrescentando os subdiretórios das versões que irão surgindo.

Digamos que precisei adicionar o serviço ```ntpd``` na Box, para quando alguém baixar e executar a Box o serviço já esteja instalado e devidamente configurado. Sendo assim, seguindo a regra do ```Versionamento Semântico``` a imagem será versionada como ```20200213.1.0```. Lembre-se de que falei dessa nomenclatura mais acima.

A estrutura de diretórios agora está da seguinte forma:

```bash
/var/www/html/vagrant/
├── boxes
│   └── centos-base-image
│       ├── v20200213.0.0
│       │   ├── centos-base-image.box
│       │   └── centos-base-image.box.sha256
│       └── v20200213.1.0
│           ├── centos-base-image.box
│           └── centos-base-image.box.sha256
└── index.json

4 directories, 5 files
```

E o arquivo ```index.json``` ficará da seguinte forma:

```json
{
    "name": "centos-base-image",
    "description": "This box contains CentOS 7.7.1908 (Core) 64-bit.",
    "versions": [
      {
        "version": "20200213.0.0",
        "providers": [
          {
            "name": "virtualbox",
            "url": "http://repo.meudominio.local/boxes/centos-base-image/v20200213.0.0/centos-base-image.box",
            "checksum_type": "sha256",
            "checksum": "55dbddadfa7f169e569898bcfcef44c5cec3d679bf53fc0634fdcb2d11074700"
          }
        ]
    },{
        "version": "20200213.1.0",
        "providers": [
          {
            "name": "virtualbox",
            "url": "http://repo.meudominio.local/boxes/centos-base-image/v20200213.1.0/centos-base-image.box",
            "checksum_type": "sha256",
            "checksum": "55dbddadfa7f169e569898bcfcef44c5cec3d679bf53fc0634fdcb2d11031456"
          }]
    }]
}
```

> É normal que a partir do momento em que vamos incrementando dados no arquivo ```index.json``` os requisitos de abrir e fechar ```}``` e ```]``` fiquem confusos. Para checar a sintaxe, utilizo um serviço web sensacional chamado [JSON formatter & Validator](https://jsonformatter.curiousconcept.com/# "Link de acesso ao site do serviço mencionado") que faz um lint detalhado no conteúdo do arquivo JSON

Quando chamo no browser obtenho o seguinte.

![Imagem 02](https://raw.githubusercontent.com/ewerton-silva00/blog-sysadmin-linux/master/static/images/02-screenshot-private-vagrant-repository.png "Imagem com a visualização do repositório no browser")

O "pulo do gato" aqui é o valor boleano da linha abaixo...

```bash
centos.vm.box_check_update = true
```

... que por estar ```true``` fará com que o Vagrant, no momento da execução do ```vagrant up```, busque no repositório por atualizações e quando houver irá automaticamente executar o download da Box.

Missão cumprida. Analise se essa solução é útil pra você e em caso positivo desejo bom uso.

Abraço!