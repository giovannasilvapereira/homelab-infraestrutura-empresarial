# 🖥️ Homelab de Infraestrutura Empresarial Virtualizada

Projeto prático desenvolvido para simular uma infraestrutura corporativa
virtualizada e segmentada, com foco em **Redes, Infraestrutura, Windows
Server, Segurança e Troubleshooting**.

O ambiente foi construído do zero em um laboratório isolado, utilizando
**Proxmox VE, pfSense, Windows Server, Active Directory, DNS, DHCP, GPO,
File Server, Windows 11 e Debian Linux**.

> **Status:** Versão 1.0 concluída e validada.\
> **Próxima fase:** monitoramento com Zabbix e dashboards no Grafana.

------------------------------------------------------------------------

## 🎯 Objetivo

Construir um ambiente empresarial virtualizado para praticar, de forma
integrada:

-   Virtualização de servidores;
-   Segmentação de redes com VLANs;
-   Roteamento e regras de firewall;
-   Active Directory Domain Services;
-   DNS e DHCP centralizados;
-   DHCP Relay;
-   Administração de usuários, grupos e OUs;
-   Ingresso de estações Windows no domínio;
-   Group Policy (GPO);
-   File Server com SMB e permissões NTFS;
-   Administração Linux;
-   Troubleshooting por camadas.

------------------------------------------------------------------------

## 🧱 Ambiente virtualizado

O **Proxmox VE 9.2** foi utilizado como hypervisor do laboratório. Cada
função principal foi separada em uma máquina virtual para facilitar
administração, testes e troubleshooting.

<img width="1270" height="958" alt="image" src="https://github.com/user-attachments/assets/5eff73bc-09b1-470d-9569-0d319a45f3a1" />



### Máquinas virtuais

  VM                Função
  ----------------- ------------------------------------------
  `FW-PFSENSE-01`   Firewall e roteamento
  `SRV-LINUX-01`    Servidor Debian Linux
  `SRV-AD-01`       Active Directory, DNS e DHCP
  `PC-W11-01`       Estação Windows 11 ingressada no domínio
  `SRV-FS-01`       File Server

------------------------------------------------------------------------

## 🌐 Arquitetura de rede

O laboratório foi mantido isolado da rede física. O pfSense atua como
gateway e roteador entre as redes internas, enquanto uma bridge
VLAN-aware no Proxmox transporta as VLANs do ambiente.

``` text
                  Rede externa
                       |
                     WAN
                       |
                 +-----------+
                 |  pfSense  |
                 +-----------+
                       |
                 Bridge interna
                  VLAN-aware
                  /         \
                 /           \
        VLAN10_USERS      VLAN20_SERVERS
        10.10.10.0/24     10.10.20.0/24
              |                  |
         PC-W11-01       SRV-AD-01 / SRV-FS-01
```

  Rede             Finalidade                Gateway
  ---------------- ------------------------- ---------------
  LAN base         Gerenciamento/transição   `192.168.1.1`
  VLAN10_USERS     Estações de usuários      `10.10.10.1`
  VLAN20_SERVERS   Servidores                `10.10.20.1`

------------------------------------------------------------------------

## 🔥 Firewall e comunicação entre VLANs

O tráfego entre as VLANs não foi liberado de forma irrestrita.

Foram criadas regras específicas no pfSense para permitir apenas os
serviços necessários entre usuários e servidores.

<img width="1236" height="584" alt="95c456e9-b593-4a6f-87c5-7b481968b986" src="https://github.com/user-attachments/assets/42a0e4a1-d799-4caa-8f2e-1baa60817a77" />


Entre as liberações implementadas estão:

-   Comunicação necessária com o controlador de domínio;
-   Portas TCP e UDP utilizadas pelos serviços do Active Directory;
-   SMB TCP/445 especificamente para o File Server;
-   ICMP utilizado durante os testes de conectividade.

Essa abordagem mantém a segmentação entre usuários e servidores.

------------------------------------------------------------------------

## 🏢 Active Directory, DNS e DHCP

O servidor `SRV-AD-01` foi configurado como controlador do domínio:

``` text
empresa.lab
```

Foram instalados e configurados:

-   Active Directory Domain Services (AD DS);
-   DNS;
-   DHCP.

<img width="1286" height="805" alt="2d4e53ed-5542-4d8c-818a-79317fda21a3" src="https://github.com/user-attachments/assets/e4e2b621-bb93-4697-9198-325191a6e01a" />


### Organização do Active Directory

A estrutura foi organizada utilizando OUs para separar os diferentes
tipos de objetos do ambiente.

``` text
EMPRESA
├── Computadores
├── Grupos
├── Servidores
└── Usuários
    ├── TI
    └── Administrativo
```

<img width="768" height="524" alt="f5d3b81b-4b1c-4203-b247-fec4f7f03176" src="https://github.com/user-attachments/assets/678aee3d-1988-40b8-8a7a-5e5a1617cf44" />


Também foi criado o grupo de segurança:

``` text
GG-TI
```

Esse grupo foi posteriormente utilizado para controlar o acesso ao
compartilhamento do departamento de TI.

------------------------------------------------------------------------

## 📡 DHCP centralizado

O DHCP está hospedado no `SRV-AD-01`, localizado na VLAN de servidores.

Foi criado um escopo para a VLAN10:

``` text
Rede:       10.10.10.0/24
Pool DHCP:  10.10.10.50 - 10.10.10.200
Gateway:    10.10.10.1
DNS:        10.10.20.10
Domínio:    empresa.lab
```

<img width="989" height="401" alt="c976aad8-0cb5-4498-998c-179c48ee0eaa" src="https://github.com/user-attachments/assets/935ed6d1-ec85-40b3-8ec9-558d5dd7b53c" />


Como clientes e servidor DHCP estão em redes diferentes, o **DHCP Relay
do pfSense** foi utilizado para encaminhar as solicitações da VLAN10 até
o servidor DHCP.

O cliente `PC-W11-01` recebeu corretamente suas configurações através
desse processo.

------------------------------------------------------------------------

## 💻 Windows 11 no domínio

A estação `PC-W11-01` foi ingressada no domínio `empresa.lab`.

O login de domínio foi validado com uma conta de usuário criada no
Active Directory, assim como a comunicação com o controlador de domínio.

------------------------------------------------------------------------

## 📜 Group Policy (GPO)

Foram implementadas políticas em dois escopos.

### Política de computador

Foi criada uma GPO para bloquear o Windows Installer na estação
gerenciada.

A aplicação foi validada utilizando:

``` powershell
gpupdate /force
gpresult /r /scope computer
```

### Política de usuário

Também foi criada uma política para impedir o acesso ao Painel de
Controle e às Configurações para usuários da OU de TI.

A aplicação da política foi validada diretamente na estação cliente e
através do `gpresult`.

------------------------------------------------------------------------

## 📁 File Server

O `SRV-FS-01` foi configurado como servidor membro do domínio e recebeu
armazenamento dedicado aos dados.

Foi criado o compartilhamento:

``` text
\\SRV-FS-01\TI
```

O acesso é controlado por permissões NTFS através do grupo:

``` text
EMPRESA\GG-TI
```

O grupo recebeu permissão **Modify**, permitindo leitura, criação,
edição e exclusão de arquivos.

------------------------------------------------------------------------

## 🔐 Validação do File Server

Antes de alterar permissões, a conectividade SMB foi testada a partir da
estação cliente:

``` powershell
Test-NetConnection 10.10.20.20 -Port 445
```

Resultado após a configuração da regra no pfSense:

``` text
TcpTestSucceeded : True
```

<img width="817" height="431" alt="image" src="https://github.com/user-attachments/assets/3e60d53a-3b36-4a0b-b121-75cb18033686" />


### Usuário autorizado

Um usuário pertencente ao `GG-TI` conseguiu:

-   Acessar o compartilhamento;
-   Criar arquivo;
-   Editar e salvar;
-   Excluir arquivo.

<img width="889" height="678" alt="image" src="https://github.com/user-attachments/assets/a86a9354-4d98-4f0d-9f2e-b965ab1075b4" />


### Usuário não autorizado

Também foi criado um usuário fora do grupo `GG-TI`.

Ao tentar acessar o mesmo compartilhamento, o servidor recusou a
solicitação.

<img width="862" height="566" alt="image" src="https://github.com/user-attachments/assets/6c2db31c-aa07-4c99-af62-b2e2f00ae2b7" />


Isso confirmou que o acesso não dependia apenas da conectividade SMB: as
permissões NTFS baseadas em grupos do Active Directory também estavam
sendo aplicadas corretamente.

------------------------------------------------------------------------

## 🛠️ Troubleshooting realizado

Um dos objetivos do projeto foi praticar diagnóstico por camadas,
evitando alterar configurações sem antes identificar a origem do
problema.

### VLAN10 sem comunicação com o gateway

Durante a configuração inicial, a estação na VLAN10 não conseguia
responder ao gateway.

A resolução ARP indicava que o tagging VLAN estava funcionando em camada
2. A análise seguinte mostrou que a nova interface do pfSense não
possuía regra permitindo o tráfego.

Foi criada uma regra de ICMP para validação e a comunicação passou a
funcionar.

### File Server inacessível

O compartilhamento inicialmente não podia ser acessado pela estação
cliente.

O teste:

``` powershell
Test-NetConnection 10.10.20.20 -Port 445
```

retornou:

``` text
TcpTestSucceeded : False
```

Em vez de alterar as permissões NTFS, as regras do firewall foram
verificadas primeiro.

Foi identificada a ausência de uma regra permitindo SMB entre a VLAN10 e
o File Server.

Após criar uma regra específica:

``` text
VLAN10_USERS → SRV-FS-01 → TCP/445
```

o teste passou a retornar:

``` text
TcpTestSucceeded : True
```

Somente depois da conectividade validada foram realizados os testes de
autorização dos usuários.

### DHCP entre VLANs

Como broadcasts DHCP não atravessam roteadores normalmente, o servidor
DHCP localizado na VLAN20 não receberia diretamente as solicitações dos
clientes da VLAN10.

A solução foi configurar **DHCP Relay no pfSense**, mantendo o DHCP
centralizado no Windows Server.

------------------------------------------------------------------------

## ✅ Resultado

Ao final da primeira fase foram validados:

-   Ambiente virtualizado no Proxmox;
-   Firewall pfSense;
-   Segmentação com VLANs;
-   Comunicação controlada entre usuários e servidores;
-   Active Directory;
-   DNS;
-   DHCP centralizado;
-   DHCP Relay;
-   Estrutura de OUs, usuários e grupos;
-   Windows 11 ingressado no domínio;
-   GPOs de computador e usuário;
-   File Server SMB;
-   Permissões NTFS baseadas em grupo;
-   Testes de acesso autorizado e não autorizado;
-   Troubleshooting de rede e serviços.

------------------------------------------------------------------------

## 🚀 Próximas implementações

### Zabbix

Implementação de monitoramento para acompanhar:

-   Disponibilidade dos servidores;
-   CPU;
-   Memória;
-   Disco;
-   Rede;
-   Serviços importantes.

### Grafana

Criação de dashboards para visualização das métricas coletadas pelo
ambiente de monitoramento.

### Outras evoluções

-   Automação administrativa com PowerShell;
-   Novas políticas de GPO;
-   Expansão do ambiente Linux;
-   Integração futura com serviços de Cloud.

------------------------------------------------------------------------

## 🧠 Competências praticadas

`Proxmox` `pfSense` `VLAN` `Firewall` `Windows Server`
`Active Directory` `DNS` `DHCP` `DHCP Relay` `GPO` `SMB` `NTFS`
`Debian Linux` `Troubleshooting` `Redes` `Infraestrutura`

------------------------------------------------------------------------

## 📄 Documentação

## 📄 Documentação completa

Para uma visão detalhada da implementação, arquitetura, testes e troubleshooting:

[📄 Acessar o portfólio completo em PDF](Portfolio_Homelab_Infraestrutura_Empresarial_Giovanna.pdf)

------------------------------------------------------------------------

> Projeto desenvolvido exclusivamente para fins de estudo e portfólio em
> ambiente isolado de laboratório. Os endereços, nomes e estruturas
> apresentados pertencem ao homelab e não representam a infraestrutura
> de uma organização real.
