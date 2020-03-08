---
title: "Monitoramento do Proxmox Virtual Environment (PVE) com Prometheus"
description: "Provisionando um ambiente de monitoramento com Prometheus para monitoramento do PVE"
author: "Ewerton Silva"
date: 2020-03-07T15:14:32-03:00
draft: false
tags: ["proxmox","prometheus"]
categories: ["proxmox"]
comments: "true"
---

Neste artigo estarei dando início à uma série sobre monitoramento com Prometheus e para iniciar quero mostrar como provisionar uma stack simples para monitorar o ```Proxmox Virtual Environment (PVE)```, pois o utilizo bastante para provisionar minhas VMs de estudo.

O [Prometheus](https://prometheus.io/ "Link de acesso ao site oficial do Prometheus") é um sistema de monitoramento open-source desenvolvido pela [SoundCloud](https://soundcloud.com/ "Link de acesso ao site oficial da SoundCloud") e que hoje é graduado pela [Cloud Native Computing Foundation](https://www.cncf.io/ "Link para acesso ao site oficial da CNCF").

O [Proxmox Virtual Environment](https://www.cncf.io/ "Link de acesso ao site oficial do PVE") é uma plataforma open-source de virtualização. Pra mim, o melhor projeto open-source de virtualização.

Para continuarmos, preciso que você já estude ou trabalhe com as ferramentas abordadas aqui, pois não irei me aprofundar em cada uma delas. Meu objetivo é te proporcionar um norte nessa jornada sobre monitoramento do PVE.

#### Criando um usuário específico para monitoramento no PVE ####

Seguindo a primeira boa prática, criaremos um usuário somente leitura chamado ```prometheus``` na qual terá a role **PVEAuditor**. O objetivo desse usuário é somente ter acesso as metricas do PVE.

Conectado via SSH como **root** no host PVE, utilize o utilitário ''pveum'' para criar o usuário em questão.

Crie um grupo chamado ```Monitoring Group```.

```bash
pveum group add monitoring -comment 'Monitoring Group'
```

Conceda a permissão ```PVEAuditor``` ao grupo criado.

```bash
pveum aclmod / -group monitoring -role PVEAuditor
```

Agora crie o usuário ```prometheus```.

```bash
pveum user add prometheus@pam
```

Adicione o usuário no grupo ```monitoring```. Dessa forma ele herdará a role ```PVEAuditor```.

```bash
pveum usermod prometheus@pam -group monitoring
```

Finalize essa parte criando uma senha para o usuário ```prometheus```.

```bash
pveum passwd prometheus@pam
```



#### Instalando o Exporter ####

Para que o Prometheus colete as métricas do PVE utilizaremos um [exporter](https://prometheus.io/docs/instrumenting/exporters/ "Link de acesso ao site do Prometheus contendo informações sobre exporters") na qual conversa com a API do hypervisor e exporte as métricas num formato na qual o Prometheus conheça.

Utilizaremos um exporter sensacional escrito em Python, desenvolvido pelo usuário [znreol](https://github.com/znerol "Link para acesso ao perfil do desenvolvedor no Github"). O cara mandou muito bem. [Clique aqui](https://github.com/znerol/prometheus-pve-exporter "Link de acesso ao repositório do projeto no Github") para conferir o projeto no Github.

Ainda conectado via SSH no host PVE com o usuário **root** execute os passos abaixo.

Instale o [pip](https://pypi.org/project/pip/ "Link de acesso ao site oficial do PyPip").

```bash
apt install python-pip -y
```

Na sequência, instale o [virtualenv](https://virtualenv.pypa.io/en/latest/ "Link de acesso ao site da documentação do virtualenv").

```bash
pip install virtualenv
```

Crie um ambiente Python isolado para instalarmos o exporter.

```bash
virtualenv /opt/prometheus-pve-exporter
```

Agora sim instale o exporter com o comando abaixo.

```bash
/opt/prometheus-pve-exporter/bin/pip install prometheus-pve-exporter
```

O resultado do comando acima precisa ser semelhante ao resultado abaixo.

```
Successfully installed Werkzeug-1.0.0 certifi-2019.11.28 chardet-3.0.4 idna-2.9 prometheus-client-0.7.1 prometheus-pve-exporter-1.1.2 proxmoxer-1.0.4 pyyaml-5.3 requests-2.23.0 urllib3-1.25.8
```

Crie o arquivo ```/etc/prometheus/pve.yml``` com o seguinte conteúdo:

```yaml
default:
  user: prometheus@pam
  password: 8JJ7te6zepQnO74Jt2urc1VkxXPsTG
  verify_ssl: false
```

* **user**: usuário criado nos passos acima com permissão ```PVEAudit```.
* **password:** senha atribuída ao usuário ```prometheus@pam```.
* **verify_ssl**: parâmetro para ignorar erros devido a certificados não reconhecidos.



Crie o usuário ```exporter```.

```bash
useradd --user-group --no-create-home --shell /sbin/nologin --comment "Prometheus Exporter" exporter
```



Crie o arquivo ```/etc/systemd/system/pve_exporter.service```.

```bash
[Unit]
Description=Prometheus Exporter for Proxmox VE
Documentation=https://github.com/znerol/prometheus-pve-exporter
Wants=network-online.target
After=network-online.target

[Service]
Type=simple

User=exporter
Group=exporter

ExecStart=/opt/prometheus-pve-exporter/bin/pve_exporter /etc/prometheus/pve.yml

Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Recarregue as configurações do systemd.

```bash
systemctl daemon-reload
```

Habilite o serviço para ser inicializado no boot do sistema operacional, mas que também o execute de imediato.

```bash
systemctl enable --now pve_exporter.service
```

Confira se ficou tudo OK com o serviço executando o comando abaixo.

```bash
systemctl status pve_exporter.service
```

Por padrão, o serviço do exporter utiliza a porta ```9122/tcp```.

Lembrando que todo o processo realizado acima pode ser automatizado com o uso do [Ansible](https://docs.ansible.com/ "Link de acesso a documentação oficial do Ansible"),  por exemplo. Principalmente quando você suporta vários nós PVE.

#### Configurando o Prometheus ####

Toda a stack de monitoramento que utilizo roda sob containers [Docker](https://www.docker.com/ "Link para acesso ao site oficial do Docker"), com isso você precisa possuir algum conhecimento sobre.

Faça o clone do meu repositório contendo os arquivos necessários para execução do Prometheus.

```bash
git clone https://github.com/ewerton-silva00/pve-monitoring.git
```

O diretório ```pve-monitoring``` possui a seguinte estrutura.

```
../monitoring-pve
├── alertmanager
│   └── alertmanager.yml
├── docker-compose.yml
├── prometheus
│   ├── alerts.yml
│   └── prometheus.yml
├── pve_monitoring.json
└── README.md

2 directories, 6 files
```



No arquivo ```prometheus.yml``` você definirá os targets, ou seja, os hosts PVE na qual você quer monitorar.

```yaml
  - job_name: 'pve'
    metrics_path: /pve
    params:
      module: [default]
    static_configs:
      - targets: ['pve.meudominio.local:9221']
        labels:
          costumer: 'blog'
          environment: 'desenvolvimento'
          hostname: 'pve.meudominio.local'
```

Perceba que os hosts monitorados são definidos na sessão ```scrape_configs``` e no exemplo acima, o host que estou definindo para ser monitorado se chama ```pve.meudominio.local```.

Veja também que defino três ```labels``` para esse ```job_name```. Isso facilita no momento em que precisamos elaborar as ```queries``` para extração de métricas, pois posso executar sempre as mesmas queries com o detalhe de que os resultados variam de acordo com as labels definidas. Útil também para estabelecer uma distinção entre os ambientes à serem monitorados e prioridade dos alertas.

Para saber mais, indico fortemente a leitura da documentação oficial do Prometheus, pois o que estou mostrando aqui é somente o básico.



#### Inicializando a stack de monitoramento

Por que **stack**?

Aqui serão executados vários containers/serviços, uns conversando com os outros.

* **Traefik**: esse serviço será responsável pelo proxy reverso. Assim evitaremos ficar acessando os serviços via IP + Porta.
* **Prometheus**: como já mencionado, esse serviço é o coração de toda a stack. Ele quem irá coletar e processar todas as métricas.
* **AlertManager**: esse serviço será responsável pelos alertas, ou seja, nele definimos como, quando e para onde as notificações são enviadas.
* **Grafana**: nesse serviço criamos os dashboards para acompanhamento visual do nosso cenário monitorado.

Crie uma network chamada ```public_network```. Será nessa network que o **traefik** conversará com os demais containers que serão acessados via proxy.

```bash
docker network create public_network
```

Agora, adicione no seu ```/etc/hosts``` o seguinte conteúdo:

```
127.0.0.1 traefik.meudominio.local
127.0.0.1 prometheus.meudominio.local
127.0.0.1 alertmanager.meudominio.local
127.0.0.1 grafana.meudominio.local
```

Inicialize a stack com o comando abaixo.

```bash
docker-compose up --detach
```

>O arquivo ```docker-compose.yml``` está adaptado para ser executado apenas como ambiente de estudos, por isso não há volumes criados. 
>
>Caso necessite, basta criar os volumes para persistência dos dados.

Feito! Com os containers em execução, basta chamar no browser o nome dos serviços e começar a brincadeira.

No meu exemplo, posso chamar no browser o endereço ```http://pve.meudominio.local:9221/pve?target=localhost``` para ter uma idéia de como as métricas são modeladas para o Prometheus.

No Prometheus, confira na aba ```Status / Targets ``` se o endpoint do PVE está com o status ```UP```.



No arquivo ```alerts.yml``` definimos as regras para os alertas. Aqui no nosso exemplo criei apenas dois na qual notifica quando um exporter está down e quando uma VM também fica down.

```yaml
- name: PVEAlerts
  rules:
  - alert: PVEGuestDown
    expr: sum by (costumer, environment, hostname, id) (pve_up{environment="desenvolvimento"}  == 0)
    for: 2m
    labels:
      severity: high
    annotations:
      summary: "A VM de id {{ $labels.id }} encontra-se indisponível."
      description: "VM indisponível no PVE {{ $labels.hostname }}, no ambiente de {{ $labels.environment }} do cliente {{ $labels.costumer }}."
```

> No caso acima o alerta só irá funcionar quando o ambiente for de desenvolvimento. Apenas exemplo.



#### Criando um dashboard no Grafana ####

Acesse o endereço ```grafana.meudominio.local```, informe ```grafana``` como usuário e ```grafana``` para senha. Em ```Add data source``` selecione o ```Prometheus``` e no campo ```URL``` informe ```http://prometheus:9090```.

Na sequência importe o dashboard ```pve_monitoring.json``` disponível como arquivo no repositório baixado.



Chegamos então no fim deste artigo. Agora você já possui o norte que faltava para desenvolver uma stack de monitoração.

Essa stack pode ficar ainda mais robusta, adicionando os ```receivers```, que são os alvos das notificações, como por exemplo: Rocketchat, Slack, Telegram, E-mail...

Em ambientes maiores e mais complexos podemos integrar o Prometheus à um serviço de ```Service Discovery```, como por exemplo o **Consul**. As possibilidades são inúmeras.

Obrigado por ter lido este artigo. Sucesso!