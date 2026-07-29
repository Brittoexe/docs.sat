# Preparação do Servidor

> Este procedimento descreve as configurações iniciais necessárias antes da instalação do cliente OpenVPN.

## Objetivo

Preparar o servidor para iniciar o processo de integração à infraestrutura da SAT.

## Escopo

Aplica-se a todos os servidores membros hospedados na Contabo.

## Pré-requisitos

- Acesso administrativo ao servidor.
- Conexão com a Internet.

---

## Etapa 1 - Renomear o servidor

Antes de iniciar a implantação, altere o nome do computador conforme o nome da empresa.

### Procedimento

1. Abra o **Server Manager**.
2. Clique em **Local Server**.
3. Clique sobre o nome do computador.
4. Clique em **Change...**.
5. Informe o novo nome do servidor.
6. Reinicie o servidor.


### Validação

Após a reinicialização, execute:

```powershell
hostname
```

### Resultado esperado

O comando deve retornar o nome configurado para o servidor.

Exemplo:

```text
PEGAPEGA
```

---

## Etapa 2 - Verificar acesso à Internet

Confirme que o servidor possui conectividade antes da instalação da VPN.

### Teste de conectividade

```cmd
ping 8.8.8.8
```

### Teste de resolução de nomes

```cmd
ping google.com
```

### Resultado esperado

Os dois comandos devem responder sem perda de pacotes.

---

## Etapa 2 - Configurar o fuso horário

Verifique se o servidor utiliza o fuso horário correto.

### Verificando o fuso

```powershell
Get-TimeZone
```

Caso necessário, mude o fuso horário:

```powershell
Set-TimeZone "E. South America Standard Time"
```

---

## Validação

```powershell
Get-TimeZone
```

---

## Etapa 3 - Verificar data e hora

Confirme que a data, hora e o fuso horário do servidor estão corretos.

### Verificação

```powershell
Get-Date
```

Caso seja necessário, ajuste manualmente utilizando as configurações do Windows.

!!! warning

    Data e hora incorretas podem impedir o estabelecimento da conexão VPN e o ingresso no domínio.

---

## Próxima etapa

Após concluir estas verificações, prossiga para o documento:

**Cliente OpenVPN**
