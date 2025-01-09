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

O processo de migração deverá passar por alguns processos até termos todo o conteúdo original replicado dentro da nuvem AWS e totalmente funcional, onde poderemos aposentar o servidor original _on-premise_:
* Replicação da aplicação dentro da nuvem AWS usando o **AWS MGN**.
* Replicação do Banco de Dados usando o **AWS DMS**.
* Testagem total dos componentes da aplicação, para averiguarmos total funcionamento independente do servidor original.
* Criação das instâncias de transição 100% funcionais.
* Criação de um Load Balancer e rotas com o serviço **Route 53** para migração completa do serviço, o que seria a parte final deste processo, já que esta etapa significa que o usuário do aplicativo do cliente tem acesso total a ele e o servidor original pode ser aposentado.
Cada passo descrito acima será melhor detalhado abaixo:

##### 1. Migração do App do Cliente para a Nuvem AWS

A etapa de migração da aplicação precisa passar por alguns estágios até estar completa, como a imagem abaixo explica em uma forma compreensível:
![MGN](mgn.png)
Detalharemos melhor nas etapas a seguir.

###### 1a. Preparação do Ambiente de Migração

Dentro do ambiente AWS devemos criar uma sub-rede que será usada como a sub-rede de preparação, onde os dados replicados do servidor de origem serão mandados pelo agente replicador. 
Nesta etapa devemos configurar algumas portas:
* **TCP 443** - Esta porta será usada dentro da sub-rede de preparação para enviar dados do servidor de replicação para a API da **MGN** 
* **TCP 1500** - Esta porta deverá ser configurada no servidor de origem para mandar dados replicados de forma comprimida e criptografada (ponta-a-ponta via TLS 1.2) para o servidor de replicação, onde ele será descomprimido e descriptografado dentro desta sub-rede e depois escritas nos volumes respectivos.

###### 1b. Instalação e Configuração do Agente de Replicação

Primeiro passo da migração será instalar o agente de replicação do **AWS MGN** no servidor _on-premise_ para conectarmos os dados do servidor aos serviços da AWS e fazermos as migrações do servidor.
Devemos configurar no servidor original do cliente para liberar a porta **TCP 443** (HTTPS) para se comunicar com o serviço **MGN** da AWS, esta operação tem diversos propósitos: 
* Baixar o software necessário para se comunicar com a Nuvem AWS (O agente replicador).
* Atualizar agentes já instalados.
* Conectar os servidores conectados ao console da **MGN** e disponibilizar ao mesmo o estado de replicação. 
* Monitorar o servidor de origem para solução de problemas e métricas de consumo de recursos (uso de CPU, RAM, etc) 
* Reportar eventos relacionados ao servidor de origem (como remoção de algum disco ou formatação de disco).
* Transmitir informações relacionadas com o servidor de origem para o serviço de migração (incluindo informação de hardware, serviços rodando, pacotes e aplicações instaladas).
* Preparar o servidor de origem para teste e transição (_cutover_).

Depois de instalado o agente de replicação, devemos esperar pela sincronização completa entre o servidor de origem e o servidor de replicação. Feito isto, podemos seguir para as próximas etapas de testes e transição.

###### 1c. Etapas de Testes

Nesta etapa iremos criar instâncias em modo de teste, onde poderemos averiguar se os servidores replicados estão funcionando corretamente.
Antes desta etapa deveremos configurar os **Launch Templates** de cada servidor que replicamos e que desejamos testar, onde deveremos configurar o tamanho da máquina que queremos disponibilizar para cada servidor, qual sistema operacional usar, qual sub-rede esta **EC2** será lançada, quais **Security Groups**, quais tipos de volumes **EBS** e até mesmo algum arquivo extra de configuração inicial (**user_data.sh**).
Nesta parte deveremos criar dois **Launch Templates** distintos, um para a máquina que terá o React e outra que terá as Regras de Negócio.
Próxima etapa será o de fazer a migração do Banco de Dados para testarmos se a aplicação tem conexão total com o **MySQL**.

##### 2. Migração do Banco de Dados

Na parte do Banco de Dados precisaremos usar um serviço diferente, o **AWS DMS**, que fará toda replicação dos Bancos para o serviço de Banco de Dados da AWS, nesse caso o cliente necessita de uma replicação contínua de dados para manter a sincronia de origem e destino indefinidamente, mantendo seu destino sincronizado com uma origem transacionamente ativa, garantindo que os dados estejam em concomitância com o ambiente modernizado. 
Este serviço da AWS criará uma réplica usando o **AWS RDS** como o banco de destino final.
Esta parte é um pouco menos complicada que o processo anterior de migração da aplicação, temos que preparar uma sub-rede apenas para o Banco de Dados, que tenha como configuração de entrada e saída pela porta **TCP 3306** e acesso contínuo ao banco de dados original, como o Banco de Dados de origem também é um servidor **MySQL** não teremos nenhum problema de conversão na hora da replicação (o serviço de conversão da AWS é o **AWS SCT**, que não precisaremos usar neste projeto), já que o servidor de replicação também usará o **MySQL**.

Abaixo segue a imagem detalhada de como funciona processo de replicação de um banco de dados usando a **AWS DMS**:
![DMS](image.png)

##### 3. Final das Testagens e Criação de Instâncias de Integração

Agora que possuímos um banco de dados completamente independente do banco original, é hora de testarmos sua integração com as aplicações replicadas na nuvem. Ainda na parte de testes podemos conectar nossa instância de back-end ao **RDS** e testarmos conectividade, também podemos testar se a aplicação funciona de forma independente.
Após passado esse processo, podemos marcar as instâncias para _cutover_, onde iniciaremos o processo de integração.
O processo de integração é mais simples, somente precisaremos configurar as rotas de entrada da aplicação, pois no fim deste processo a migração estará completa e o **MGN** parará de replicar os dados do servidor original, o que significa que poderemos aposentar o servidor _on-premise_.

##### 4. Configuração das Rotas de Entrada e Load Balancer

Como o cliente já usa o **Route 53**, não precisaremos fazer muitas configurações, pois no final do processo de transição só precisaremos do Load Balancer configurado para se comunicar com o **Route 53**.
Nesta etapa do Load Balancer, criaremos um do tipo **Application Load Balancer**, que é a solução mais moderna da AWS para balanceamento de cargas de uma aplicação.
Para configurarmos o **ALB** precisaremos especificar um **Target Group**, registrar os objetos alvos, configurar o Load Balancer e um Listener e, por fim, testar o Load Balancer.
Feito isto, temos toda a arquitetura de integração e já podemos fazer os testes finais e aposentar o servidor de origem.

---

#### 2. Quais as ferramentas vão ser utilizadas?

* **MGN** - Serviço que facilita a migração de servidores e aplicações locais ou em outras nuvens para a infraestrutura da AWS, projeto para simplificar o processo de migração ao usar replicação contínua e reduzir o tempo de inatividade.

* **DMS** - Serviço para facilitar a migraçãode bancos de dados para a nuvem da AWS ou entre bancos de dados diferentes, utilizado para modernizar ambientes de dados, transferindo-os para bancos de dados gerenciados, como Amazon RDS.

* **RDS** - Serve para facilitar a configuração, operação e escalabilidade de bancos de dados relacionais na nuvem. Ele elimina grande parte do trabalho operacional envolvido  na manutenção de bancos de dados, como privisionamento de hardware, instalação de software, backup, recuperação e atualizações.

* **EC2** - Fornece servidores virtuais para hospedar aplicações e executar cargas de trabalho de forma escalável e flexível. 

* **EBS** - Serviço utilizado para armazenamento de blocos da AWS projetado para ser usado com instâncias EC2. Ele fornece volumes de armazenamento persistente que podem ser anexados a instâncias EC2, oferecendo desempenho de alto nível para cargas de trabalho como bancos de dados, sistemas de arquivos e aplicativos intensivos de dados.

* **Route 53** - Serviço DNS que oferece recursos de roteamento de tráfego, registro de domínios e verificação de integridade, projetado para alta disponibilidade e escalabilidade. Utilizado para gerenciar nomes de domínio e direcionar usuários finais para aplicações hospedadas na AWS ou fora dela.

* **ALB** - Solução para balanceamento de carga gerenciada pela AWS, projetada para operar na camada de aplicação (Camada 7 do modelo OSI). Ele distribui o tráfego HTTP/HTTPS de forma inteligente com base em regras avançadas, permitindo a criação de aplicações modernas e altamente escaláveis.

* **S3** - Serviço de armazenamento em nuvem altamente escalável, durável e seguro. Permite armazenar e recuperar qualquer quantidade de dados de qualquer lugar na internet.

* **SSM** - Ajuda a gerenciar e operar sua infraestrutura na nuvem e on-premises de forma centralizada. Fornece ferramentas para automação, controle de configuração, gerenciamento de patches, execução remota de comandos e coleta de logs, sendo amplamente utilizado para melhorar a segurança e a eficiência operacional.

---

#### 3. Qual o diagrama da infraestrutura na AWS?

![Diagrama](diagrama.png)

---

#### 4. Como serão garantidos os requisitos de Segurança?

Para garantir a segurança da arquitetura de migração, deve-se utilizar o **AWS Application Migration Service (MGN)** assegurando que as conexões sejam feitas de maneira criptografada (TLS 1.2), transferindo os dados entre o ambiente de origem e a nuvem, protegendo-os contra interceptação. 

As permissões concedidas pelo **AWS MGN** são configuradas pelo **AWS IAM** para assegurar que somente os usuários ou roles autorizadas possuam o acesso. As regras de segurança Inbound e Outbound rules serão de acordo com a infraestrutura da migração, sendo a porta **TCP 443** como acesso direto a esses endpoint da API de serviço pelo protocolo HTTPS, e a porta **TCP 1500** como saída direta do servidor de origem para a sub-rede da área de preparo, esta porta usará o protocolo TLS 1.2, que criptografa ponta-a-ponta e compacta os dados do servidor de origem. A área de preparação possuíra acesso direto a esses endpoints da API de serviço pela porta **TCP 443** com entrada direta pela porta **TCP 1500**, garantindo assim boas práticas de seguranças durante a migração.

---

#### 5. Como será realizado o processo de Backup?

A Staging Area se comunica com o **Amazon S3** utilizando a porta **TCP 443** para proteger os dados em trânsito. Os dados transferidos são armazenados em um bucket **S3** com políticas de segurança, aceitando somente conexões seguras, sendo acessado restritamente pela Staging Area de administradores autorizados.

---

#### 6. Qual o custo da infraestrutura na AWS (AWS Calculator)?

Para a primeira etapa, referente a migração das aplicações e banco de dados, foi realizado uma estimativa dos custos com o **AWS Pricing Calculator** com todos os serviços incluídos no processo. 

Segue abaixo a tabela de custos da estimativa:

|Service           |Price (Monthly)    |Region          |
|------------------|-------------------|-----------------
|MGN               |$ 0,00             |North Virginia  |
|S3                |$ 1,04             |North Virginia  |
|EC2 (MGN)         |$ 42,47            |North Virginia  |
|EC2 (React)       |$ 16,18            |North Virginia  |
|EC2 (Back-end)    |$ 28,45            |North Virginia  |
|SSM               |$ 0,00             |North Virginia  |
|RDS               |$ 293,34           |North Virginia  |
|DMS               |$ 169,92           |North Virginia  |
|ALB               |$ 22,27            |North Virginia  |
|VPC               |$ 104,48           |North Virginia  |
|Route 53          |$ 3,60             |North Virginia  |
|Total             |$ 681,76

Esses valores se referem a um custo mensal, agora podemos calcular o valor mais aproximado do tempo estimado da migração.

...

---

### Escopo Detalhado (Modernização Kubernetes)

---

#### Quais atividades são necessárias para a modernização?

Para esta etapa precisaremos seguir alguns passos para completa modernização do sistema usando o Kubernetes:
* 

---

#### Quais as ferramentas vão ser utilizadas?

---

#### Qual o diagrama da infraestrutura na AWS?

---

#### Como serão garantidos os requisitos de Segurança?

---

#### Como será realizado o processo de Backup?

---

#### Qual o custo da infraestrutura na AWS (AWS Calculator)?

Para a segunda etapa, referente a modernização com o ambiente EKS incluso, também foi realizado a estimativa de custos com o **AWS Pricing Calculator**. Nessa parte foi criado a estimativa tanto para a região North Virginia quanto para a de Ohio, com o objetivo de mostrar diferentes escolhas com uma arquitetura de Multi-Regiões ao cliente.

Segue abaixo a tabela de custos da estimativa da região North Virginia:

|Service           |Price (Monthly)    |Region          |
|------------------|-------------------|----------------|
|RDS               |$ 293,34           |North Virginia  |
|RDS (Réplica)     |$ 293,34           |North Virginia  |
|Route 53          |$ 3,60             |North Virginia  |
|ALB               |$ 22,27            |North Virginia  |
|WAF               |$ 25,00            |North Virginia  |
|S3                |$ 2,91             |North Virginia  |
|CloudWatch        |$ 16,08            |North Virginia  |
|Secrets Manager   |$ 4,00             |North Virginia  |
|KMS               |$ 11,00            |North Virginia  |
|EKS Cluster       |$ 73,00            |North Virginia  |
|VPC               |$ 154,20           |North Virginia  |
|Total             |$ 898,74           

Valor total da modernização Single-Region = $ 898,74

Segue abaixo a tabela de custos da estimativa da região Ohio:

|Service           |Price (Monthly)    |Region          |
|------------------|-------------------|----------------|
|RDS               |$ 293,34           |Ohio            |
|RDS (Réplica)     |$ 293,34           |Ohio            |
|ALB               |$ 22,27            |Ohio            |
|WAF               |$ 25,00            |Ohio            |
|S3                |$ 2,91             |Ohio            |
|CloudWatch        |$ 16,08            |Ohio            |
|Secrets Manager   |$ 4,00             |Ohio            |
|KMS               |$ 11,00            |Ohio            |
|EKS Cluster       |$ 73,00            |Ohio            |
|VPC               |$ 154,20           |Ohio            |
|Total             |$ 895,14

Para o modelo de Multi-Region, somamos o custo mensal das duas regiões para uma infraestrutura com uma alta disponibilidade e resiliência das aplicações, reduzindo a sobrecarga de cada região, otimizando a utilização de recursos.

Valor total da modernização Multi-Region = $ 1.793,88

---

### Referências

[Documentação da AWS MGN](https://docs.aws.amazon.com/mgn/latest/ug/what-is-application-migration-service.html)
[Documentação da AWS DMS](https://docs.aws.amazon.com/dms/latest/userguide/Welcome.html)

---
texto do vito
texto do Gabriel
