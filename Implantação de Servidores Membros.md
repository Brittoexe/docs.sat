# Implantação de Servidores Membros na Contabo

> Procedimento operacional padrão para implantação, configuração e integração de servidores membros da infraestrutura da SAT.

---

## Objetivo

Esta documentação descreve o processo padrão para implantação de novos servidores membros hospedados na Contabo e sua integração à infraestrutura da SAT.

O objetivo é padronizar todas as implantações, garantindo que os servidores sejam configurados de forma consistente, segura e alinhada aos padrões da empresa.

Ao final do procedimento, o servidor deverá:

- Estar conectado à VPN corporativa.
- Receber as Políticas do Active Directory
- Fazer parte do domínio **SAT.COM**. 
- Receber as Políticas de Grupo (GPO).
- Possuir acesso aos recursos corporativos.
- Estar apto para receber aplicações corporativas.

---

## Escopo

Esta documentação aplica-se a todos os servidores membros hospedados na Contabo.


Esta documentação **não** se aplica aos Controladores de Domínio.

---

# Infraestrutura

A infraestrutura da SAT é composta por dois Controladores de Domínio e diversos Servidores Membros.

```text
                               INTERNET
                                    │
                           IP Público Contabo
                                    │
                           OpenVPN (UDP 1194)
                                    │
                          Rede VPN 10.8.0.0/24
                                    │
            ┌───────────────────────┴───────────────────────┐
            │                                               │
┌───────────────────────────────┐             ┌───────────────────────────────┐
│          SERVERSAT            │◄──────────►│         ADDACONTABO            │
├───────────────────────────────┤ Replicação ├───────────────────────────────┤
│ Controlador de Domínio        │     AD     │ Controlador de Domínio         │
│ DNS Principal                 │            │ DNS Secundário                 │
│ OpenVPN Server                │            │ Replicação do Active Directory │
│ EasyRSA                       │            │                                │
└───────────────┬───────────────┘            └───────────────┬────────────────┘
                │                                            │
                └──────────────────────┬─────────────────────┘
                                       │
                              Servidores Membros
                                       │
          ─────────────────────┬───────────────────────────
                                       │
                               Demais Servidores na nuvem
```

---

## Componentes da Infraestrutura

### SERVERSAT

Servidor responsável por:

- Active Directory Domain Services
- DNS Principal
- OpenVPN Server
- EasyRSA
- Compartilhamentos corporativos
- Políticas de Grupo (GPO)

---

### ADDACONTABO

Servidor responsável por:

- Controlador de Domínio Adicional
- Replicação do Active Directory
- DNS Secundário
- Redundância da autenticação

---

### Servidores Membros

Os servidores membros executam aplicações corporativas e utilizam os serviços disponibilizados pelos Controladores de Domínio.

Eles não executam serviços de Active Directory.

---

# Fluxo de Implantação

Todo servidor membro deverá seguir obrigatoriamente a sequência abaixo.

```text
Novo Servidor

        │

        ▼

Preparação Inicial

        │

        ▼

Configuração da Rede

        │

        ▼

Criação do Certificado VPN

        │

        ▼

Instalação do OpenVPN

        │

        ▼

Conexão com a VPN

        │

        ▼

Ingresso no Domínio SAT.COM

        │

        ▼

Aplicação das GPOs

        │

        ▼

Instalação das Aplicações

        │

        ▼

Validação Final
```

---

# Organização da Documentação

Esta documentação está organizada em três grandes áreas.

## Infraestrutura

Documentação da infraestrutura corporativa.

Inclui:

- Active Directory
- DNS
- OpenVPN
- GPO
- Compartilhamentos
- Padrões da empresa

---

## Implantação

Procedimentos para implantação de novos servidores membros.

Inclui:

- Preparação do servidor
- Certificados
- Cliente OpenVPN
- Ingresso no domínio
- Área de Trabalho Remota
- Instalação do SAT

---

## Operação

Procedimentos de manutenção.

Inclui:

- Troubleshooting
- Boas práticas
- Manutenção
- Checklist
- Procedimentos operacionais

---

# Convenções

Ao longo desta documentação serão utilizados os seguintes padrões.

## PowerShell

```powershell
Get-Service
```

## Prompt de Comando

```cmd
ipconfig /all
```

## OpenVPN

```ovpn
client
proto udp
```

## Registro do Windows

```reg
Windows Registry Editor Version 5.00
```

---

# Boas Práticas

- Execute todos os procedimentos utilizando uma conta com privilégios administrativos.
- Valide cada etapa antes de prosseguir para a próxima.
- Não altere configurações fora do padrão definido nesta documentação.
- Toda alteração realizada em produção deve ser previamente validada em ambiente de homologação.

---

# Controle de Revisões

| Versão | Data | Descrição |
|---------|------|-----------|
| 1.0 | Em desenvolvimento | Criação inicial da documentação |