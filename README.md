## Atividade de Kubernetes na Nuvem AWS
_Projeto de Kubernetes - Compass.UOL_

**Autores: Gabriel Faraco e Vitor Renk**

---

### Introdução

O projeto se baseia em uma arquitetura de um cliente fictício, onde temos que migrar a arquitetura atual para a Nuvem AWS, depois de migrada teremos que criar uma arquitetura nova e moderna baseada em Kubernetes.
O projeto é apenas teórico, onde teremos que detalhar de forma técnica todo o processo de migração e modernização.

---

### Índice

* [Escopo Detalhado (As-Is)](#escopo-detalhado-as-is)
* [Escopo Detalhado (Kubernetes)](#escopo-detalhado-modernização-kubernetes)

---


### Escopo Detalhado (As-Is)

#### Quais atividades são necessárias para a migração?

##### 1. Migração do App do Cliente para a Nuvem AWS

Esta parte do processo é a mais complexa desta etapa, por isso, separamos seus processos em passos separados abaixo:

###### 1a. Preparação do Ambiente de Migração

Dentro do ambiente AWS devemos criar uma sub-rede que será usada como a sub-rede de preparação, onde os dados replicados do servidor de origem serão mandados pelo agente replicador.

###### 1b. Instalação e Configuração do Agente de Replicação
Primeiro passo da migração será instalar o agente de replicação do **AWS MGN** no servidor _on-premise_ para conectarmos os dados do servidor aos serviços da AWS e fazermos as migrações do servidor.
Devemos configurar no servidor original do cliente para liberar a porta TCP 443 (HTTPS) para se comunicar com o serviço **MGN** da AWS, esta operação tem diversos propósitos: 
* Baixar o software necessário para se comunicar com a Nuvem AWS (O agente replicador).
* Atualizar agentes já instalados.
* Conectar os servidores conectados ao console da **MGN** e disponibilizar ao mesmo o estado de replicação. 
* Monitorar o servidor de origem para solução de problemas e métricas de consumo de recursos (uso de CPU, RAM, etc) 
* Reportar eventos relacionados ao servidor de origem (como remoção de algum disco ou formatação de disco).
* Transmitir informações relacionadas com o servidor de origem para o serviço de migração (incluindo informação de hardware, serviços rodando, pacotes e aplicações instaladas).
* Preparar o servidor de origem para teste e transição (_cutover_).


##### 2. Migração do Banco de Dados do Cliente
Na parte do Banco de Dados precisaremos usar um serviço diferente, o **AWS DMS**, que fará toda replicação dos Bancos para o serviço de Banco de Dados da AWS, o **AWS RDS**.

#### Quais as ferramentas vão ser utilizadas?

* **MGN**
* **DMS**
* **RDS**
* **EC2**
* **EBS**
* **Route 53**
* **ELB**
* **S3**
* **SSM**


#### 3. Qual o diagrama da infraestrutura na AWS?

#### 4. Como serão garantidos os requisitos de Segurança?
Para garantir a segurança da arquitetura de migração, deve-se utilizar o **AWS Application Migration Service (MGN)** assegurando que as conexões sejam feitas de maneira criptografada (TLS/SSL), transferindo os dados entre o ambiente de origem e a nuvem, protegendo-os contra interceptação. 

As permissões concedidas pelo **AWS MGN** são configuradas pelo **AWS IAM** para assegurar que somente os usuários ou roles autorizadas possuam o acesso. As regras de segurança Inbound e Outbound rules serão de acordo com a infraestrutura da migração, sendo a **port TCP 443** como acesso direto a esses endpoint da API de serviço pelo protocolo HTTPS, e a **port TCP 1500** como saída direta do servidor de origem para a sub-rede da área de preparo. A área de preparação possuíra acesso direto a esses endpoints da API de serviço pela port TCP 443 com entrada direta pela port TCP 1500, garantindo assim boas práticas de seguranças durante a migração.

#### 5. Como será realizado o processo de Backup?
A Staging Area se comunica com o Amazon S3 utilizando a port TCP 443 para proteger os dados em trânsito. Os dados transferidos são armazenados em um bucket S3 com políticas de segurança, aceitando somente conexões seguras, sendo acessado restritamente pela Staging Area de administradores autorizados.



#### 6. Qual o custo da infraestrutura na AWS (AWS Calculator)?

---

### Escopo Detalhado (Modernização Kubernetes)

#### Quais atividades são necessárias para a migração?

#### Quais as ferramentas vão ser utilizadas?

#### Qual o diagrama da infraestrutura na AWS?

#### Como serão garantidos os requisitos de Segurança?

#### Como será realizado o processo de Backup?

#### Qual o custo da infraestrutura na AWS (AWS Calculator)?

texto do vito
texto do Gabriel
