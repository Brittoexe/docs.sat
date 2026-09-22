# Procedimento: Conectar VPS ao Active Directory via NetBird Cloud

**Última validação:** 22/09/2026 — testado com sucesso na VPS ADCONTABO (Windows Server 2025)

---

## Visão geral da arquitetura

```
┌─────────────────────┐         ┌──────────────────────┐
│   serversat          │◄───────►│   NetBird Cloud        │
│   (AD principal)      │  túnel  │   (gerenciamento)      │
│   192.168.1.100       │  P2P    │                        │
│   IP NetBird: 100.78.25.61      └──────────────────────┘
└──────────┬───────────┘                    ▲
           │ Rota de rede                    │ túnel P2P
           │ 192.168.1.0/24                  │
           ▼                                 │
┌─────────────────────┐         ┌──────────────────────┐
│   ADCONTABO           │◄───────►│   VPS clientes (1-30)  │
│   86.48.21.106         │         │   (Windows Server)     │
│   IP NetBird: 100.78.61.229     │                        │
└──────────────────────┘         └──────────────────────┘
```

- O **NetBird Cloud** (plano gratuito, até 100 peers) cria uma rede privada virtual entre todas as máquinas, sem precisar abrir portas no roteador.
- Uma **rota de rede** publicada pelo `serversat` permite que qualquer VPS conectada enxergue a rede local `192.168.1.0/24`.
- Um **nameserver customizado** no NetBird aponta consultas do domínio `REDE.COM` para o DNS do AD (`192.168.1.100`).
- O **DNS do serversat** tem *forwarders* configurados (`8.8.8.8`, `1.1.1.1`) para resolver a internet também — isso é essencial, sem isso o NetBird perde conexão quando o DNS da VPS aponta só para o AD.

---

## Pré-requisitos (feitos uma única vez, já concluídos)

- [x] Conta criada em [app.netbird.io](https://app.netbird.io) (plano Free)
- [x] `serversat` conectado ao NetBird (IP: `100.78.25.61`)
- [x] `ADCONTABO` conectado ao NetBird (IP: `100.78.61.229`)
- [x] Rota de rede criada: `192.168.1.0/24` via peer `serversat`
- [x] Nameserver customizado criado: domínio `REDE.COM` → `192.168.1.100`, grupo `All`
- [x] Forwarders configurados no DNS do `serversat` (via `dnsmgmt.msc` → Propriedades → Encaminhadores → `8.8.8.8` e `1.1.1.1`)
- [x] Setup Key **reutilizável** criada, sem limite de uso:

```
netbird up --setup-key B960F2DA-A64E-4EA3-9CC6-D65BB7C97FB6
```

> ⚠️ Guarde essa key com cuidado — qualquer máquina com ela consegue entrar na sua rede NetBird.

---

## Passo a passo para cada VPS nova (~2-3 minutos cada)

Execute tudo no **PowerShell como Administrador**.

### 1. Instalar o cliente NetBird

```powershell
Invoke-WebRequest -Uri "https://pkgs.netbird.io/windows/x64" -OutFile "$env:TEMP\netbird-installer.msi"
Start-Process msiexec.exe -ArgumentList "/i `"$env:TEMP\netbird-installer.msi`" /quiet" -Wait
```

### 2. Conectar à rede NetBird com a Setup Key definitiva

```powershell
netbird up --setup-key B960F2DA-A64E-4EA3-9CC6-D65BB7C97FB6
```

Se der erro de timeout/`DeadlineExceeded`, force a Management URL explicitamente:

```powershell
netbird up --management-url https://api.netbird.io:443 --setup-key B960F2DA-A64E-4EA3-9CC6-D65BB7C97FB6
```

### 3. Apontar o DNS da VPS para o AD

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses "192.168.1.100"
ipconfig /flushdns
```

### 4. Validar antes de prosseguir (não pule esta etapa)

```powershell
nslookup REDE.COM
nslookup app.netbird.io
ping 192.168.1.100
netbird status
```

Espera-se:
- `nslookup REDE.COM` e `nslookup app.netbird.io` resolvendo sem timeout
- `ping` respondendo
- `netbird status` mostrando `Management: Connected` e `Signal: Connected`

### 5. Ingressar a VPS no domínio

```powershell
Add-Computer -DomainName "REDE.COM" -Credential (Get-Credential) -Restart
```

Use uma conta de administrador do domínio (ex: `REDE\Administrador`) quando solicitado. A VPS reinicia automaticamente.

### 6. Confirmar após o reboot

```powershell
Get-ComputerInfo | Select-Object CsDomain, CsDomainRole
nltest /dsgetdc:REDE.COM
```

Deve retornar `CsDomain: REDE.COM` e o `nltest` deve encontrar `serversat.REDE.COM` sem erro.

### 7. Controlar quem pode acessar a VPS

Opção rápida (por VPS):
```powershell
# Abrir gerenciamento de usuários/grupos locais
lusrmgr.msc
```
Adicione o usuário do domínio desejado ao grupo local **Remote Desktop Users**.

Opção centralizada (recomendada para 30+ VPS):
1. Crie uma OU no AD, ex: `VPS-Clientes`.
2. Mova o objeto de computador da VPS para essa OU.
3. Crie uma GPO vinculada à OU usando **Restricted Groups** para gerenciar o grupo `Remote Desktop Users` de todas as VPS da OU de uma vez.

---

## Solução de problemas comuns

| Sintoma | Causa provável | Solução |
|---|---|---|
| `netbird up` trava com `DeadlineExceeded` | DNS da VPS não resolve `api.netbird.io` (geralmente porque está apontando só para o AD sem forwarder) | Trocar DNS temporariamente para `8.8.8.8`/`1.1.1.1`, reconectar o NetBird, depois voltar para `192.168.1.100` |
| Interface `wt0` desaparece / `Management: Disconnected` | Mesma causa acima | Mesma solução acima |
| `Add-Computer` falha com "servidor não pode executar a operação" | DNS não está resolvendo os registros SRV do AD (`_ldap._tcp.dc._msdcs.REDE.COM`) | Confirmar `Set-DnsClientServerAddress` para `192.168.1.100` antes de tentar o join |
| Setup Key para de funcionar depois de 1 uso | Key criada sem marcar "Reusable" | Sempre usar a key `B960F2DA-A64E-4EA3-9CC6-D65BB7C97FB6` (já é reutilizável e sem limite) |
| `netbird up` conecta no IP público da própria VPS (`http://SEU_IP:80`) em vez do NetBird Cloud | Perfil salvo com Management URL customizada errada | Apagar `C:\ProgramData\Netbird\*.json`, reiniciar o serviço, reconectar com `--management-url https://api.netbird.io:443` explícito |

---

## Checklist rápido por VPS

- [ ] Instalar cliente NetBird
- [ ] `netbird up --setup-key B960F2DA-A64E-4EA3-9CC6-D65BB7C97FB6`
- [ ] DNS → `192.168.1.100`
- [ ] Validar `nslookup` + `ping` + `netbird status`
- [ ] `Add-Computer -DomainName "REDE.COM"`
- [ ] Confirmar `CsDomain` após reboot
- [ ] Adicionar usuários autorizados ao grupo Remote Desktop Users (ou via GPO/OU)
