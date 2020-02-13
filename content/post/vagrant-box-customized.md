---
title: "Criando sua própria Vagrant Box para o VirtualBox"
description: "Passo à passo para criar sua Vagrant Box personalizada e executar no VirtualBox"
author: "Ewerton Silva"
date: 2020-02-12T12:41:18-03:00
draft: true
tags: ["vagrant"]
categories: ["vagrant"]
---

[Vagrant](https://www.vagrantup.com/ "Link de acesso ao site oficial do Vagrant") e [VirtualBox](https://www.virtualbox.org/ "Link de acesso ao site oficial do VirtualBox"), nunca vi tecnologias ser tão amadas, mas também tão odiadas ao mesmo tempo. Falo isso, porque sou "rato" de blog técnico, gosto demais de ficar lendo os artigos que a turma publica e isso me fez perceber que o pessoal reclama muito do Vagrant e do VirtualBox, o primeiro por ser uma tecnologia de provisionamento de máquinas virtuais, sendo que o hype é containers, infraestrutura imutável e afins e o segundo, porque apresenta muitos problemas até em ambientes de estudos.

Confesso que gosto das duas tecnologias, porque me ajudaram demais nos meus LABs. Coleciono muitas histórias felizes com ambos.

Agora, porque personalizar uma Vagrant Box se posso simplesmente utilizar as boxes prontas disponíveis no [Vagrant Cloud](https://app.vagrantup.com/boxes/search "Link de acesso ao repositório oficial de imagens do Vagrant")? Minha resposta é **controle**.

Quando crio minha própria Box consigo ter controle sobre quase tudo. Isso me proporciona uma maior segurança.

Para este artigo utilizei o seguinte:

- VirtualBox 6.0.16
- Vagrant 2.2.6
- CentOS 7.7.1908 (CentOS-7-x86_64-Minimal-1908.iso)

Vou iniciar passando logo um "pulo do gato" que é criar uma interface virtual no VirtualBox para termos comunicação direta com as VMs do Vagrant. Isso ajuda demais, porque não gosto de ficar mapeando portas quando posso ter acesso direto. Think about it!

![Imagem 00](/archives/Blog/blog/static/images/00-screenshot-vagrant-box-customized.png "Interface vboxnet0 criada")

Nesse caso, criei uma interface virtual chamada **vboxnet0** com IP **10.10.10.1/24**, ou seja, quando eu provisionar qualquer VM no meu notebook dentro dessa faixa de rede terei acesso direto. Bom demais!

#### Criando uma máquina virtual no VirtualBox ####

Aqui, criaremos uma máquina virtual mais crua possível para no final converter numa Vagrant Box.

**Passo 00:** Atribuo um nome para a minha VM.

![Imagem 01](/archives/Blog/blog/static/images/01-screenshot-vagrant-box-customized.png "Atribuindo um nome para a VM")

**Passo 01:** Deixo a memória RAM em 512 MB.

![Imagem 02](/archives/Blog/blog/static/images/02-screenshot-vagrant-box-customized.png "Memória RAM em 512 MB")

**Passo 02:** Seleciono a opção **Create a virtual hard disk now**.

![Imagem 03](/archives/Blog/blog/static/images/03-screenshot-vagrant-box-customized.png "Opção Create a virtual hard disk now selecionada")

**Passo 03:** Marco a opção **VDI (VirtualBox Disk Image)**.

![Imagem 04](/archives/Blog/blog/static/images/04-screenshot-vagrant-box-customized.png "Opçao VDI (VirtualBox Disk Image selecionada)")

**Passo 04**: Seleciono a opção **Dynamically allocated**.

![Imagem 05](/archives/Blog/blog/static/images/05-screenshot-vagrant-box-customized.png "Opção Dynamically allocated selecionada")

**Passo 05:** Aumento o tamanho do disco, de **8,00 GB** para **10,00 GB**.

![Imagem 06](/archives/Blog/blog/static/images/06-screenshot-vagrant-box-customized.png "Aumentei o disco de 8,00 GB para 10,00 GB")

Após clicar em **Create** a VM estará pronta para "bootar" a imagem do sistema operacional, mas antes disso preciso de mais alguns passos.

**Passo 06:** Desabilito a interface USB desmarcando a opção **Enable USB Controller**.

![Imagem 07](/archives/Blog/blog/static/images/07-screenshot-vagrant-box-customized.png "Desabilitar a interface USB desmarcando a opçao Enable USB Controller")

**Passo 07:** Desabilito o áudio desmarcando a opção **Enable Audio**.

![imagem 08](/archives/Blog/blog/static/images/08-screenshot-vagrant-box-customized.png "Desabilitar o áudio desmarcando a opçao Enable Audio")

**Passo 08:** Seleciono a ISO do CentOS que irei utilizar.

![Imagem 09](/archives/Blog/blog/static/images/09-screenshot-vagrant-box-customized.png "ISO do CentOS selecionada para boot")



#### Instalação do Sistema Operacional ####

Após inicializar a VM, siga os passos convencionais de instalação, mas atenção para o particionamento do disco.

![Imagem 10](/archives/Blog/blog/static/images/10-screenshot-vagrant-box-customized.png "Disco devidamente particionado")

Particionei da seguinte forma:

- 256MB para a partição **/boot**, com filesystem XFS;

- 256MB para a **swap**, com filesystem XFS + LVM;

- O que sobrou aloquei na partição **/ [raiz]**, com filesystem XFS + LVM; 

Utilizar LVM na **swap** e **/ [raiz]** nos permite futuramente aumentar o disco através do Vagrantfile. Mostro isso num próximo artigo.

Concluo essa parte da seguinte forma:

![Imagem 11](/archives/Blog/blog/static/images/11-screenshot-vagrant-box-customized.png "Status do Installation Summary")

À seguir, chegamos nos passos de criar uma senha para o usuário **root** e criar um usuário à parte, na opção **User Creation**.

Outro "pulo do gato" aqui é criar um usuário chamado **vagrant** com senha **vagrant** e permissão de **Administrator**. Ficando assim:

![Imagem 12](/archives/Blog/blog/static/images/12-screenshot-vagrant-box-customized.png "Usuário vagrant criado")

Feito isso é só aguardar a conclusão da instalação do sistema operacional.

#### Preparando a VM para converter numa Vagrant Box ####

Certo. A VM concluiu a instalação do sistema operacional, reiniciou e já está online, pronta para darmos continuidade.

Os passos à seguir podem ser feitos via interface gráfica do VirtualBox, mas irei fazer via command line mesmo, pois evito ter que ficar tirando screenshots. É bem massante.

No terminal do meu notebook ao executar o comando ```VBoxManage list runningvms``` tenho como retorno a VM recém provisionada com estado **running**, ou seja, em execução.

```bash
"CentOS-7-x86_64-Minimal-1908" {81b71bb8-6163-4389-86ab-4957df85790a}
```

Paro a VM com o comando abaixo.

```bash
VBoxManage controlvm CentOS-7-x86_64-Minimal-1908 poweroff
```

Crio uma regra de NAT Port Forward para a interface NAT da VM permitindo o acesso via SSH.

```bash
VBoxManage modifyvm "CentOS-7-x86_64-Minimal-1908" --natpf1 "guestssh,tcp,,2222,,22"
```

Inicio a VM novamente com o comando abaixo.

```bash
VBoxManage startvm "CentOS-7-x86_64-Minimal-1908" --type headless
```

Agora estamos jogando em casa. SSH funcional.

```bash
ssh -l root localhost -p 2222
```

Já conectado, atualizo os pacotes instalados. Claro.

```bash
yum update -y
```

Faço a instalação do repositório EPEL (Extra Packages for Enterprise Linux), pois vou precisar desse cara para instalar o **dkms**.

```bash
yum install epel-release -y && yum clean metadata
```

Preciso instalar o **VBoxGuestAdditions** na VM. Lembrando que precisa ser na mesma versão do VirtualBox. No meu caso, **6.0.16**.

Como dependência, instalo os pacotes abaixo.

```bash
yum install wget gcc kernel-devel kernel-headers dkms make bzip2 perl -y
```

E agora sim baixo a ISO do **VBoxGuestAdditions**.

```bash
wget https://download.virtualbox.org/virtualbox/6.0.16/VBoxGuestAdditions_6.0.16.iso
```

Crio um diretório temporário pra montar a ISO.

```bash
mkdir /media/VBoxGuestAdditions
```

Monto a ISO.

```bash
mount -o loop,ro VBoxGuestAdditions_6.0.16.iso /media/VBoxGuestAdditions
```

Exporto as duas variáveis de ambiente abaixo.

```bash
export KERN_DIR=/usr/src/kernels/$(uname -r)
export MAKE="/usr/bin/gmake -i"
```

E agora sim posso instalar o VBoxGuestAdditions com o comando abaixo.

```bash
sh /media/VBoxGuestAdditions/VBoxLinuxAdditions.run --nox11
```

O comando acima precisa de alguns minutos para concluir e o resultado da execução é a seguinte:

```bash
[root@localhost ~]# sh /media/VBoxGuestAdditions/VBoxLinuxAdditions.run --nox11
Verifying archive integrity... All good.
Uncompressing VirtualBox 6.0.16 Guest Additions for Linux........
VirtualBox Guest Additions installer
Copying additional installer modules ...
Installing additional modules ...
VirtualBox Guest Additions: Starting.
VirtualBox Guest Additions: Building the VirtualBox Guest Additions kernel 
modules.  This may take a while.
VirtualBox Guest Additions: To build modules for other installed kernels, run
VirtualBox Guest Additions:   /sbin/rcvboxadd quicksetup <version>
VirtualBox Guest Additions: or
VirtualBox Guest Additions:   /sbin/rcvboxadd quicksetup all
VirtualBox Guest Additions: Building the modules for kernel 
3.10.0-1062.12.1.el7.x86_64.
```

Show. Desmonto a ISO e removo o diretório temporário.

```bash
umount /media/VBoxGuestAdditions && rmdir /media/VBoxGuestAdditions
```

Já posso remover também a ISO baixada.

```bash
rm -f VBoxGuestAdditions_6.0.16.iso
```

Agora é chegada a hora de fazer uns ajustes finais e extremamente importante para que tudo funcione corretamente. Que são:

Criar o arquivo ```/etc/sudoers.d/vagrant``` com o conteúdo abaixo.

```bash
Defaults:vagrant !requiretty
vagrant ALL=(ALL) NOPASSWD: ALL
```

AJusto as permissões do arquivo.

```bash
chmod 0440 /etc/sudoers.d/vagrant
```

Dou aquela conferida pra checar se ficou tudo OK mesmo com o arquivo.

```bash
visudo -c -f /etc/sudoers.d/vagrant
```

O resultado do comando acima precisa ser ```parsed OK```.

Agora crio o diretório que ficará a chave pública do usuário **vagrant**.

```bash
mkdir /home/vagrant/.ssh
```

Insiro a chave pública.

```bash
curl -k https://raw.githubusercontent.com/mitchellh/vagrant/master/keys/vagrant.pub > /home/vagrant/.ssh/authorized_keys
```

E ajusto as permissões.

```bash
chown vagrant:vagrant /home/vagrant/.ssh -R
chmod 0700 /home/vagrant/.ssh
chmod 0600 /home/vagrant/.ssh/authorized_keys
```

Faço uma rápida conferência no arquivo ```/etc/ssh/sshd_config``` para garantir que as linhas abaixo estejam presentes.

```bash
AuthorizedKeysFile      .ssh/authorized_keys
UseDNS no
```

Agora executo os comandos abaixo para limpar a VM antes de empacotar.

```bash
yum clean all
dd status=progress if=/dev/zero of=/EMPTY bs=1M
rm -f /EMPTY
echo > /root/.bash_history && history -c && exit
```

#### Gerando a Box ####

Desligo a VM com o comando abaixo.

```bash
VBoxManage controlvm CentOS-7-x86_64-Minimal-1908 poweroff
```

Lembrando que ```CentOS-7-x86_64-Minimal-1908``` é o nome da minha VM.

Agora sim executo o famigerado comando para empacotar a VM.

```bash
vagrant package --output CentOS-7-x86_64-Minimal-1908.box --base CentOS-7-x86_64-Minimal-1908
```

Acima, ```CentOS-7-x86_64-Minimal-1908.box``` é o nome que dei para minha Box, na qual gerei tendo como base a VM de nome ```CentOS-7-x86_64-Minimal-1908```. Bem simples.

O comando acima irá demandar alguns minutos e terá como resultado a seguinte saída.

```bash
==> CentOS-7-x86_64-Minimal-1908: Clearing any previously set forwarded ports...
==> CentOS-7-x86_64-Minimal-1908: Exporting VM...
==> CentOS-7-x86_64-Minimal-1908: Compressing package to: /home/ewerton/CentOS-7-x86_64-Minimal-1908.box
```

Vagrant Box gerado, já posso importar via ```vagrant box add ...```, mas eu particularmente não gosto. Prefiro criar um arquivo JSON com as informações de importação da Box, porque assim tenho mais controle e oportunidade para versionar. Isso mesmo, trabalhar com versionamento.

Pra isso, gero uma hash sha256sum do arquivo.

```bash
sha256sum CentOS-7-x86_64-Minimal-1908.box
```

Resultado do comando acima.

```bash
5f3ae4fda102fe8da68b60b683316e5dc6d7d37c9c9291e98524379719c0464a  CentOS-7-x86_64-Minimal-1908.box
```

Agora "confecciono" um arquivo JSON chamado ```CentOS-7-x86_64-Minimal-1908.json``` - gosto de sempre utilizar o nome da Box - cujo conteudo é:

```json
{
    "name": "CentOS-7-x86_64-Minimal-1908",
    "description": "This box contains CentOS 7.7.1908 (Core) 64-bit.",
    "versions": [
      {
        "version": "1.0.0",
        "providers": [
          {
            "name": "virtualbox",
            "url": "file:///home/ewerton/CentOS-7-x86_64-Minimal-1908.box",
            "checksum_type": "sha256",
            "checksum": "5f3ae4fda102fe8da68b60b683316e5dc6d7d37c9c9291e98524379719c0464a"
          }
        ]
      }
    ]
}
```

Só de bater o olho já percebemos do que se trata. Bem autoexplicativo.

Agora é só criar um Vagrantifile e rodar a VM.

Como? Assim, gigante.

```yaml
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|


  config.vm.define "centos-tutorial" do |centos|
    centos.vm.box = "CentOS-7-x86_64-Minimal-1908"
    centos.vm.box_version = "1.0.0"
    centos.vm.box_url = "file:///home/ewerton/CentOS-7-x86_64-Minimal-1908.json"
    centos.vm.box_check_update = false
    centos.vm.hostname = "centos-tutorial"
    centos.vm.network "private_network", ip: "10.10.10.10"
    centos.vm.post_up_message = "CentOS 7.7.1908"

    centos.vm.provider "virtualbox" do |vb|
      vb.name = "vagrant-centos-tutorial"
      vb.gui = false
      vb.cpus = 2
      vb.memory = 1024
    end

    centos.vm.provision "shell", inline: <<-SHELL
      getenforce
      firewall-cmd --state
    SHELL
  end
end
```

Confiro se a sintaxe está correta.

```bash
vagrant validate
```

O resultado do comando acima precisa ser ```Vagrantfile validated successfully.```

Já posso brincar com a Box executando o comando ```vagrant up```.

![Imagem 13](/archives/Blog/blog/static/images/13-screenshot-vagrant-box-customized.png "Resultado do comando vagrant up")

O resultado só poderia ser sucesso. Muito bom.

Seguindo os passos acima e adaptando alguns pontos você conseguirá criar qualquer Box personalizada para utilizar em ambientes de estudo.

Lembrando que aqui mostrei a importação de uma Box local, no meu notebook. Num próximo artigo demonstro como criar um repositório privado e centralizado utilizando um servidor web. Dessa forma você não precisará utilizar o [Vagrant Cloud](https://app.vagrantup.com/boxes/search "Link de acesso ao site do Vagrant Cloud").

Até a próxima e sucesso. Abraço!