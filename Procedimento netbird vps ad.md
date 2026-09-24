# Guia Rápido — Conectar Nova VPS ao AD via NetBird

## 1. Instalar o NetBird (download manual)

Baixe manualmente pelo navegador (não usar script/`Invoke-WebRequest`):
`https://pkgs.netbird.io/windows/x64`

Instale o `.msi` baixado.

## 2. Conectar ao NetBird

```powershell
netbird up --setup-key B960F2DA-A64E-4EA3-9CC6-D65BB7C97FB6
```

## 3. Deixar o NetBird resiliente (evita cair de novo)

```powershell
Set-Service netbird -StartupType Automatic
sc.exe failure netbird reset= 86400 actions= restart/60000/restart/60000/restart/60000
```

## 4. Configurar DNS duplo

Primeiro descubra o nome real da interface (varia por VPS):
```powershell
Get-NetAdapter
```
Ignore `wt0` (túnel do NetBird). Use o nome da placa física real.

```powershell
Set-DnsClientServerAddress -InterfaceAlias "NOME_DA_INTERFACE" -ServerAddresses "100.78.25.61","1.1.1.1"
ipconfig /flushdns
nslookup serversat.rede.com
```

> Nunca deixar só um DNS interno — sem o `1.1.1.1` como fallback, se o NetBird cair a VPS não consegue nem reconectar sozinha (`context deadline exceeded`).

## 5. Ingressar no domínio

```powershell
Add-Computer -DomainName "REDE.COM" -Credential (Get-Credential) -Restart
```

Depois do reboot, validar:
```powershell
Get-ComputerInfo | Select-Object CsDomain, CsDomainRole
Test-ComputerSecureChannel -Verbose
```

**Se der "already in that domain" com secure channel `False`:**
```powershell
Remove-Computer -UnjoinDomainCredential (Get-Credential) -WorkgroupName "WORKGROUP" -Force -PassThru -Restart
# depois do reboot:
Add-Computer -DomainName "REDE.COM" -Credential (Get-Credential) -Restart
```

## 6. Mover para a OU certa (resolve o loop de RDP)

No `serversat` → `dsa.msc` → **Computers** → localizar a VPS → botão direito → **Mover...** → **SERVIDORES_CONTABO**.

Na VPS:
```powershell
gpupdate /force
gpresult /r
```
Confirmar que `RDP - Servidores Contabo` aparece do lado **Computer**.

## 7. Confirmar grupo de acesso

```powershell
net localgroup "Remote Desktop Users"
```
Deve conter `REDE\SERVIDORES_CONTABO`. Se não tiver, adicione:
```powershell
net localgroup "Remote Desktop Users" "REDE\SERVIDORES_CONTABO" /add
```

E confirmar no AD que o **usuário** que vai acessar está de fato dentro do grupo `SERVIDORES_CONTABO` (`dsa.msc` → usuário → aba *Member Of*).

## 8. Testar

RDP com uma conta de **domínio** (não local). Se entrar sem loop, está pronto.

---

## Checklist

- [ ] NetBird baixado manualmente e instalado
- [ ] `netbird up` conectado
- [ ] Auto-start + auto-restart do serviço configurados
- [ ] DNS duplo (`100.78.25.61` + `1.1.1.1`) na interface física correta
- [ ] Ingressado no domínio (`CsDomain: REDE.COM`, secure channel `True`)
- [ ] Movido para a OU `SERVIDORES_CONTABO`
- [ ] `RDP - Servidores Contabo` aplicada (gpresult)
- [ ] Usuário/grupo com permissão de RDP confirmado no AD e no `Remote Desktop Users` local
- [ ] Login de domínio testado sem loop
