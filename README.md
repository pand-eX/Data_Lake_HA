# Alta Disponibilidade no Data Lake 

[![NPM](https://img.shields.io/npm/l/react)](https://github.com/pand-eX/Data_Lake_HA/blob/main/LICENSE) 

# About the project

O Projeto tem duas partes a primeira irei configurar e mostrar os conceito High Availability no Data Lake e na segunda parte do projeto irei implementar segurança com Kerberos e fazer alguns teste de invasão e penetração no Data Lake. 



## Porque? 

Porque o Conceito de Data Lake foi desenvolvido com alta disponibilidade apenas no DataNodes é com esse projeto irei implementar Alta Disponibilidade também nos NameNodes e mostrar os conceito para ajudar na criação de arquitetura de High Availability no Data Lake.

-Este projeto faz parte do meu portfólio pessoal, então ficarei feliz se você puder me fornecer qualquer feedback sobre o projeto, código, estrutura ou qualquer coisa que você possa relatar que possa me fazer um melhor engenheiro de dados!

Email-me: henricao_7@yahoo.com.br

Connect with me at [LinkedIn](https://www.linkedin.com/in/henrique-castro-484269203//).

## Iniciando o Projeto

Essa é aparte 1 do projeto a parte 2 será configurar segurança com Kerberos nesse cluster HA e fazer alguns testes de invasão vou separar em 2 projeto para não ficar maçante o conteúdo.

Todos os script estão em anexo !!!

Para ter a arquitetura de Alta Disponibilidade no Data Lake algumas questões precisam ser respondidas.
1) O Data Lake vai alimentar aplicações de dados, analytics de maneira continua?
2) O Data Lake vai receber dados de maneira continua?
Essa é uma questão de negócio não uma decisão técnica.
Com isso em mente iremos entender a dinâmica do cluster e como ele funciona com alta disponibilidade.

![1](https://github.com/pand-eX/Data_Lake_HA/blob/main/Alta%20Disponibilidade%20Data%20Lake/assets/1.png)

Se pararmos um Slave o cluster continua por que ele foi desenvolvido exatamente para essa situação, mas no caso do Namenode parar não existe mais cluster então a alta disponibilidade atende nesse quesito

![2](https://github.com/pand-eX/Data_Lake_HA/blob/main/Alta%20Disponibilidade%20Data%20Lake/assets/2.png)


Nosso objetivo é implementar uma arquitetura aonde eu tenha alta disponibilidade no sistema de arquivo distribuído no HDFS que nós utilizamos para implementar um Data Lake.
Sem falar em Alta disponibilidade no Hadoop nós temos na arquitetura 1 NameNode Ativo e os DataNode esse é o padrão.
Então para alta disponibilidade a equipe do Hadoop pensou em criar uma réplica do NameNode e aí surgiu o StandBy NameNode, ou seja, ao invés de ter apenas 1 NameNode eu tenho 2. Mas tome cuidado esse não é o Secondary NameNode ele é uma cópia do NameNode aqui é diferente eu tenho 2 NameNode sendo que 1 vai estar ativo é o outro em StandBy o DataNode reporta para os 2 NameNodes, ou seja, com isso eu elimino o problema do ponto único de falha, só que como faço para tornar esse processo automático? No caso de um NameNode cair eu automaticamente ativar outro NameNode. Para que isso acontece eu preciso ter os Metadados armazenados em um ponto único porque se eu deixar os metadados no NameNode Ativo e no StandBy NameNode eles podem em algum momento ficar fora de sincronia, portanto os desenvolvedores do Hadoop pensaram vamos criar uma Localidade Central para guardar esses Metadados(que basicamente são o mapa para chegar até os dados) que é o que chamamos de Journal Nodes(SHARED) Basicamente teremos um Diretorio central para que os dois NameNode fique olhando para o Metadados e para fazer isso de forma automatica usaremos o Zookeeper com o Failover Controller.

![3](https://github.com/pand-eX/Data_Lake_HA/blob/main/Alta%20Disponibilidade%20Data%20Lake/assets/3.png)

Essa é a arquitetura de Alta disponibilidade do Hadoop. Observação: os 2 NameNodes não vão estar ativos ao mesmo tempo porque se isso acontecesse corria o risco de você ter os Metadados corromptidos porque eu teria 2 serviços acessando o mesmo conjunto de dados ao mesmo tempo, embora, seja possivel implementar esse tipo de solução em um Cluster de banco de dados Oracle por exemplo utilizando Rack vocês tem 2 áres de memória acessando o mesmo arquivo físico isso lá no Oracle. Mas aqui a equipe do hadoop implementou de maneiras diferente nós temos 1 NameNode Ativo esse está controlando o Cluster só que as informações de Metadados estão sendo gravas também naquele repositório Central que é o Journa Nodes e o nosso StandBy também está olhando para esse repositório .
O Segredo para Alta Disponibilidade no Hadoop é na verdade você criar esse local compartilhado para que os Metadados possam ser gravados. 
Para criar essa área compartilhada que é o Journal nodes nós temos 2 estratégias com Hadoop
O Quorum Journal Nodes e Shared Storage. 

Utilizando o método Quorum Journal Nodes para implementar Alta Disponibilidade nós inserimos o conceito de mais um tipo de nó que é o Journal Node ele pode ser uma outra máquina no Cluster ou ele pode ser uma máquina rodando um Deamon de Journal Node.

![4](https://github.com/pand-eX/Data_Lake_HA/blob/main/Alta%20Disponibilidade%20Data%20Lake/assets/4.png)

O edit logs e onde o NameNode guarda os Metadados que são informações sobre os blocos, sobre os DataNodes, cluster e etc... Esse Edit log é Gerenciado pelo Journal Nodes pode ser um Deamon em qualquer uma das máquinas e só o Active NameNode e que pode gravar no Edit Log. Do outro lado o StandBy pode apenas fazer essa leitura e atualiza sua base de banco de dados. 
E os DataNodes(slaves) são configurados para enviar o Heartbeat para ambos os NameNodes. Que nada mais é que um sinal DataNode envia para o NameNode. Com essa arquitetura os DataNode esá ciente que tem 2 NameNodes e ele então envia seu heartbeat para os 2 NameNode e isso é importante porque no caso do Active NameNode cair o StandBy NameNode já sabe de tudo que está aconteçendo ele está lá só ouvindo mas quando active NameNode cair ele assume porque ele já sabe onde está os blocos, quem são os DataNodes já tem informações do metadados porque ele faz a leitura do Edit Logs.

1) O Active NameNode e o StandBy NameNode mantêm-se em sincronia entre si por meio de um grupo separado de nós ou daemons chamados JournalNodes. Os JournalNodes seguem a topologia em anel onde os nós são conectados uns aos outros para formar um anel.
2) O JournalNode atende a solicitação que chega a ele e copia as informações para outros nós no anel. Isso fornece tolerância a falhas em caso de falha no JournalNode.
3) O Active NameNode é responsável por atualizar os EditLogs (informações de metadados) presentes nos JournalNodes.
4) O Standby NameNode lê as alterações feitas no EditLogs no JournalNode e as aplica ao seu próprio namespace de maneira constante.
5) Durante o failover, o Standby NameNode garante que atualizou suas informações de metadados dos JournalNodes antes de se tornar o novo Active NameNode. Isso torna o estado do namespace atual sincronizado com o estado antes do failover
6) Os endereços IP de ambos os NAmeNodes estão disponíveis para todos os DataNodes que enviam seus heartbeats e informações de localização de blocos para ambos os NameNodes. Isso fornece um failover rápido (menos tempo de inatividade), pois o StandbyNode possui uma informação atualizada sobre o local do bloco no cluster.
O Outro método é o Shared Storage a ideia é basicamente a mesma, mas ao invés de ter os Journal Nodes nós temos um Shared Storage, ou seja, uma única área de armazenamento que é compartilhada entre os 2 NameNodes o Active e o StandBy.

![5](https://github.com/pand-eX/Data_Lake_HA/blob/main/Alta%20Disponibilidade%20Data%20Lake/assets/5.png)

Todos os Namespaces, ou seja, a base de dados dos 2 NameNodes compartilham Edit logs que está no Shared Storage no armazenamento compartilhado os comportamentos dos DataNodes são exatamente iguais.

1) O Standby NameNode e o Active NameNode mantêm-se em sincronia entre si usando um dispositivo de armazenamento compartilhado (Shared Storage). O Active NameNode registra qualquer modificação feita em seu namespace em um Editlog presente neste armazenamento compartilhado. O Standby NameNode lê as alterações feitas nos EditLogs neste armazenamento compartilhado e as aplica ao seu próprio namespace.

2) Agora, no caso de failover, o Standby NameNode atualiza suas informações de metadados usando o EditLogs no armazenamento compartilhado em primeiro lugar. Em seguida, sincronizado com o estado antes do failover.

3) O administrador deve configurar pelo menos um método de proteção para evitar um cenário de “Split-Brain”. O sistema pode empregar uma variedade de mecanismo. Isso pode incluir a eliminação do processo do NameNode e a revogação de seu acesso ao diretório de armazenamento compartilhado.

4) Como último recurso, podemos cercar o NameNode anteriormente ativo com uma técnica conhecida como STONITH, ou “desligue a outra máquina”. O STONITH usa uma unidade especializada de distribuição de energia para forçar a desativação da máquina NameNode.

## HDFS High Availability Architecture

O Failover é um procedimento pelo qual um sistema transfere automaticamente o controle para o sistema secundário quando detecta uma falha. Existem dois tipos de failover: 

- O Graceful Failover é quando você faz uma transferência no Node Ativo para o StandBy de maneira programada de maneira planejada por exemplo quando você precisa fazer algum tipo de manutenção. 

- E o Automatic Failover que é feito configuração para o caso de alguma falha no Ative NameNode automaticamente o StandBy NameNode Assume a posição de ativo.

Como uma arquitetura como um todo vai saber quando um NameNode Cair??? 
O Responsável por isso é o Apache Zookeeper junto com o Failover Controller vai ficar monitorando os NameNodes

![6](https://github.com/pand-eX/Data_Lake_HA/blob/main/Alta%20Disponibilidade%20Data%20Lake/assets/6.png)

## Para implementar um Cluster de Alta Disponibilidade, você deve ficar atento aos seguintes requisitos de Hardware:

Máquinas NameNode > As Máquinas nas quais você executa os NameNodes Ativo e de Standby devem ter hardware equivalente entre si e hardware equivalente ao que seria usado em um cluster não HA.
Armazenamento Compartilhado > Você precisará ter um diretório compartilhado ao qual as máquinas NameNode tenha acesso de leitura / gravação. Normalmente, esse é um serivodr remoto que suporta NFS e é montado em cada uma das máquinas NameNode. Atualmente, apenas em único diretório de edições compartilhadas é suportado.

## Configurando e testando Arquitetura HA

Primeiro é iniciar o Journal em pelo menos 3 máquinas que é o recomendado pela documentação
Hdfs --daemon start journalnode
Depois e iniciar o zookeeper em todas as máquinas do cluster 
zkServer.sh start
Agora vamos iniciar o NameNode porém agora temos 2 NameNode no primeiro Ok você pode inicializar normalmente porque ele é o Active mas e o segundo ele é o StandBy se ligar ambos não vai funcionar... 
hdfs --daemon start namenode no NameNode Active….
No Standby precisamos copiar os fsimagen(que são os metadados para ele saber o que está acontecendo quando o Active para de funcionar) comando > hdfs namenode -bootstrapStandby (ele diz> esse aqui é o meu NameNode só que ele sera o NameNode StandBy vá até o ativo copie os Metadados e podemos inicializar aqui em StandBy e agora inicializamos o StandBy > hdfs --daemon start namenode  
Agora é só inicializar os Datanode hdfs --daemon start datanode 
Falta agora só habilitar o Failover primeiro precisamos formatar o failover e ele que conecta no NameNode Ativo ele mata qualquer processo ele faz isso conectando via ssh sem senha e então ele habilita o NameNode StandBy um comportamento totalmente automático é isso que nós queremos e tem que ser feito nos 2 NameNode >hdfs zkfc -formatZK e depois podemos inicializar o NameNode hdfs --daemon start zkfc
