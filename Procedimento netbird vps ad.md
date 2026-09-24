# Procedimento: Conectar VPS ao Active Directory via NetBird Cloud

**Última validação:** 23/09/2026 — testado com sucesso na VPS `vmi1484408` (Windows Server 2025)

---

## Visão geral da arquitetura

```
┌───────────────────────┐          ┌────────────────────┐
│   serversat            │ ◄───────►│   NetBird Cloud     │
│   (AD principal)       │  túnel   │   (gerenciamento)   │
│   192.168.1.100         │  P2P     │                    │
│   IP NetBird: 100.78.25.61        └────────────────────┘
└──────────┬────────────┘                    ▲
           │ Rota de rede                    │ túnel P2P
           │ 192.168.1.0/24                  │
           ▼                                 │
┌─────────────────────┐         ┌──────────────────────┐
│   ADCONTABO            │◄───────►│   VPS clientes (1-30)  │
│   IP NetBird: 100.78.61.229     │   (Windows Server)     │
└──────────────────────┘         └──────────────────────┘
```

- O **NetBird Cloud** (plano gratuito, até 100 peers) cria uma rede privada virtual entre todas as máquinas, sem precisar abrir portas no roteador.
- Uma **rota de rede** publicada pelo `serversat` permite que qualquer VPS conectada enxergue a rede local `192.168.1.0/24`.
- Um **nameserver customizado** no NetBird aponta consultas do domínio `REDE.COM` para o DNS do AD.
- O **DNS do serversat** tem *forwarders* configurados (`8.8.8.8`, `1.1.1.1`) para resolver a internet também — essencial, sem isso o NetBird perde conexão quando o DNS da VPS aponta só para o AD.

---

## Pré-requisitos (feitos uma única vez, já concluídos)

- [x] Conta criada em [app.netbird.io](https://app.netbird.io) (plano Free)
- [x] `serversat` conectado ao NetBird (IP: `100.78.25.61`)
- [x] `ADCONTABO` conectado ao NetBird (IP: `100.78.61.229`)
- [x] Rota de rede criada: `192.168.1.0/24` via peer `serversat`
- [x] Nameserver customizado criado: domínio `REDE.COM` → DNS do AD, grupo `All`
- [x] Forwarders configurados no DNS do `serversat` (via `dnsmgmt.msc` → Propriedades → Encaminhadores → `8.8.8.8` e `1.1.1.1`)
- [x] Setup Key **reutilizável** criada, sem limite de uso

> ⚠️ Guarde a Setup Key com cuidado — qualquer máquina com ela consegue entrar na rede NetBird.

---

## Passo a passo para cada VPS nova (~5 minutos cada)

Execute tudo no **PowerShell como Administrador**, salvo onde indicado.

### 1. Baixar o instalador do NetBird — **manualmente**

> ⚠️ **Não use `Invoke-WebRequest` nem qualquer script para baixar o instalador automaticamente.** Em VPS Contabo esse download automatizado já falhou silenciosamente ou trouxe um arquivo incompleto sem erro aparente — o instalador precisa ser baixado manualmente pelo link, e não por script.

1. Na própria VPS (via RDP), abra o navegador e acesse:
   `https://pkgs.netbird.io/windows/x64`
   O download do `.msi` já inicia automaticamente ao abrir o link — só é preciso fazer isso **pelo navegador**, não por comando.
2. Confirme que o arquivo baixado tem tamanho razoável (não é `0 KB` nem truncado) antes de seguir.
3. Execute o instalador manualmente, com duplo clique no `.msi` baixado (ou, se preferir, `Start-Process` apontando para o caminho exato do arquivo já baixado — nunca para uma URL).

### 2. Conectar à rede NetBird com a Setup Key definitiva

```powershell
netbird up --setup-key SUA-SETUP-KEY
```

Se der erro de timeout/`DeadlineExceeded`, force a Management URL explicitamente:

```powershell
netbird up --management-url https://api.netbird.io:443 --setup-key SUA-SETUP-KEY
```

Confirme no painel do NetBird (app.netbird.io → Peers) que a VPS aparece conectada.

### 3. Deixar o NetBird conectando sozinho após reboot ou queda

```powershell
Set-Service netbird -StartupType Automatic
sc.exe failure netbird reset= 86400 actions= restart/60000/restart/60000/restart/60000
```

O primeiro comando garante que o serviço suba junto com o Windows. O segundo garante que, se o serviço cair por qualquer motivo (não só reboot), o Windows tenta reiniciá-lo automaticamente até 3 vezes, esperando 60s entre tentativas — sem isso, uma queda do NetBird exige alguém entrar manualmente para reconectar.

### 4. Apontar o DNS da VPS para o AD — **sempre com DNS duplo**

> ⚠️ **Antes de rodar o comando abaixo, confirme o nome real da interface de rede.** O nome `"Ethernet"` é só um exemplo — ele varia por VPS/provedor (já vimos `Ethernet`, `Ethernet 2`, etc.). Rodar o comando com o nome errado dá o erro:
> ```
> Set-DnsClientServerAddress : Nenhum objeto MSFT_DNSClientServerAddress encontrado com a propriedade
> 'InterfaceAlias' igual a 'Ethernet'.
> ```
> Para descobrir o nome certo:
> ```powershell
> Get-DnsClientServerAddress
> ```
> ou, para ver junto o tipo/fabricante de cada placa:
> ```powershell
> Get-NetAdapter
> ```
> Ignore a interface `wt0` (ou similar) — essa é o túnel virtual do próprio NetBird (`WireGuard Tunnel`), não a placa de rede física. Pegue o nome exato (coluna `InterfaceAlias`/`Name`) da placa de rede real da VPS — normalmente a que já aparece com IP/DNS preenchido no `Get-DnsClientServerAddress`, ou com descrição de fabricante (ex.: `Red Hat VirtIO Ethernet Adapter`) no `Get-NetAdapter`. Use esse nome exato no lugar de `"Ethernet"` no comando abaixo.

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses "100.78.25.61","1.1.1.1"
ipconfig /flushdns
```

> ⚠️ **Sempre configure DNS duplo desde o primeiro dia** — `100.78.25.61` (serversat, via NetBird) como primário e `1.1.1.1` como secundário. Se o NetBird cair por qualquer motivo, o Windows recorre automaticamente ao segundo DNS, evitando o catch-22 onde a VPS não consegue resolver nem o endereço do management do NetBird para reconectar sozinha (`context deadline exceeded` no `netbird up`). Configurar só um DNS interno é a causa raiz mais comum desse erro.
>
> **Use o IP do `serversat` na malha NetBird (`100.78.25.61`), não o IP local do AD (`192.168.1.100`).** Em teoria, apontar para `192.168.1.100` deveria funcionar via a rota de rede `192.168.1.0/24` publicada pelo `serversat` — mas, na prática, isso só funciona se essa rota estiver habilitada e no grupo certo para aquela VPS específica. Na `vmi1484408` a rota não estava alcançando a VPS e `192.168.1.100` deu timeout total; `100.78.25.61` resolveu de primeira. Se quiser usar `192.168.1.100` no futuro, confirme antes no painel NetBird (Network Routes) que a rota está habilitada e inclui o grupo dessa VPS — enquanto isso não for validado, use o IP NetBird do `serversat` diretamente.

### 5. Validar antes de prosseguir (não pule esta etapa)

```powershell
nslookup REDE.COM
nslookup app.netbird.io
ping 100.78.25.61
netbird status
```

Espera-se:
- `nslookup REDE.COM` e `nslookup app.netbird.io` resolvendo sem timeout
- `ping` respondendo
- `netbird status` mostrando `Management: Connected` e `Signal: Connected`

> **Se `netbird status` aparecer `Disconnected` com `context deadline exceeded`**: é o catch-22 DNS ↔ NetBird — o DNS da VPS já está apontando só para o AD e o NetBird não consegue mais falar com o management para reconectar. Corrija assim:
> ```powershell
> Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses "1.1.1.1","8.8.8.8"
> ipconfig /flushdns
> netbird up
> netbird status
> ```
> Assim que `Management: Connected`, volte o DNS para o par duplo `100.78.25.61`,`1.1.1.1` (passo 4) e valide de novo. Se essa VPS já estava com DNS duplo configurado desde o início, esse problema não deveria ocorrer — revise o passo 4 se ele se repetir.

### 6. Ingressar a VPS no domínio

```powershell
Add-Computer -DomainName "REDE.COM" -Credential (Get-Credential) -Restart
```

Use uma conta de administrador do domínio (ex.: `REDE\Administrador`) quando solicitado. A VPS reinicia automaticamente.

> Se o `Add-Computer` retornar **"já está nesse domínio"**, é porque a VPS já foi ingressada antes (ex.: antes de uma queda do NetBird). Não é erro — vá direto para o passo 7 e confirme o estado do secure channel.

### 7. Confirmar após o reboot e validar o secure channel

```powershell
Get-ComputerInfo | Select-Object CsDomain, CsDomainRole
nltest /dsgetdc:REDE.COM
Test-ComputerSecureChannel -Verbose
```

Deve retornar `CsDomain: REDE.COM`, o `nltest` deve encontrar `serversat.REDE.COM` sem erro, e o secure channel deve estar `True`.

**Se o secure channel vier `False`:**

```powershell
Test-ComputerSecureChannel -Repair -Credential (Get-Credential)
```

**Se o `-Repair` falhar** (secure channel muito dessincronizado — sintoma comum quando a VPS já tinha sido ingressada e o NetBird caiu no meio do processo), é preciso sair e reingressar à força:

```powershell
# 1. Sair do domínio à força (Remove-Computer normal falha se o secure channel já estiver quebrado)
Remove-Computer -UnjoinDomainCredential (Get-Credential) -WorkgroupName "WORKGROUP" -Force -PassThru -Verbose -Restart

# 2. Depois do restart, confirmar que saiu do domínio
Get-ComputerInfo | Select-Object CsDomain, CsDomainRole   # deve mostrar WORKGROUP

# 3. Ingressar novamente (o NetBird precisa estar conectado — repita passos 2/3/5 antes)
Add-Computer -DomainName "REDE.COM" -Credential (Get-Credential) -Restart

# 4. Validar
Get-ComputerInfo | Select-Object CsDomain, CsDomainRole   # deve mostrar REDE.COM
Test-ComputerSecureChannel -Verbose                        # deve retornar True
```

> `Add-Computer` sozinho **recusa** reingressar (`"already in that domain"`) quando a máquina já pertence ao domínio, mesmo com o secure channel quebrado — por isso o `Remove-Computer -Force` é obrigatório nesse cenário, não opcional.

### 8. Corrigir o "loop" de login no RDP (mover para a OU correta)

Toda VPS ingressada cai, por padrão, no contêiner **Computers** do AD — que **não recebe GPO nenhuma**. É isso que causa o efeito de **loop no RDP**: o usuário digita a senha, a tela pisca e volta para o login, porque a política que libera o logon via RDP nunca chega até a máquina.

No **serversat**:

```
dsa.msc
```

1. Vá em **Computers** (contêiner padrão) — localize o objeto da VPS (ex.: `VMI1484408`).
2. Botão direito → **Mover...** → selecione a OU **`SERVIDORES_CONTABO`** → OK.
   > Se o objeto já estiver nessa OU, não precisa mover para fora e para dentro de novo — não muda nada.

Na VPS, force a atualização:

```powershell
gpupdate /force
gpresult /r
```

Confirme que **`RDP - Servidores Contabo`** aparece em "Objetos de política de grupo aplicados" do lado **Computer**.

### 9. Controlar quem pode acessar a VPS

Opção rápida (por VPS):
```powershell
lusrmgr.msc
```
Adicione o usuário do domínio desejado ao grupo local **Remote Desktop Users**.

Opção centralizada (recomendada para 30+ VPS): já coberta pelo passo 8 — a GPO `RDP - Servidores Contabo`, vinculada à OU `SERVIDORES_CONTABO`, controla o acesso de todas as VPS daquela OU de uma vez via o grupo `SERVIDORES_CONTABO`.

### 10. Validar o login com conta de domínio (obrigatório)

> Rodar `gpresult /r` como uma conta **local** da própria VPS sempre mostra `N/A` do lado User, mesmo com tudo certo — isso é esperado e não indica bug. Contas locais nunca recebem GPO de usuário do AD.

1. Faça RDP na VPS usando uma conta de **domínio** do grupo `SERVIDORES_CONTABO` (ex.: `REDE\usuario.teste`).
2. Se conseguir logar sem loop, o problema do passo 8 está resolvido.
3. Dentro dessa sessão, rode `gpresult /r` de novo e confirme que aparece, do lado **User**, a GPO de papel de parede/tema (ex.: `GPO_WallpaperDominio`) e que ela aplicou visualmente.

---

## Solução de problemas comuns

| Sintoma | Causa provável | Solução |
|---|---|---|
| `Set-DnsClientServerAddress` falha com "Nenhum objeto... encontrado com a propriedade 'InterfaceAlias'" | Nome da interface (`"Ethernet"`) não existe nessa VPS — varia por provedor | Rodar `Get-DnsClientServerAddress` ou `Get-NetAdapter`, pegar o nome real da placa física (ignorar `wt0`, que é o túnel do NetBird) |
| `netbird up` trava com `DeadlineExceeded` | DNS da VPS não resolve `api.netbird.io` (geralmente porque já aponta só para o AD, sem fallback) | Trocar DNS temporariamente para `8.8.8.8`/`1.1.1.1`, reconectar o NetBird, depois voltar para o DNS do AD |
| Interface `wt0` desaparece / `Management: Disconnected` depois de um reboot | Serviço `netbird` não subiu automático, ou o DNS ficou preso no AD antes do túnel subir | `Set-Service netbird -StartupType Automatic`; se já caiu, usar a correção de DNS temporário acima |
| `Add-Computer` falha com "servidor não pode executar a operação" | DNS não está resolvendo os registros SRV do AD (`_ldap._tcp.dc._msdcs.REDE.COM`) | Confirmar `Set-DnsClientServerAddress` para o IP do AD antes de tentar o join |
| `Add-Computer` diz "already in that domain" mas o secure channel está quebrado | VPS já tinha sido ingressada antes (ex.: NetBird caiu no meio do processo) | `Test-ComputerSecureChannel -Repair`; se falhar, `Remove-Computer -Force` + `Add-Computer` de novo |
| Login via RDP entra em loop (senha aceita, mas volta pra tela de login) | Objeto de computador ainda está no contêiner padrão `Computers`, sem GPO de RDP aplicada | Mover o objeto para a OU `SERVIDORES_CONTABO` no `dsa.msc` + `gpupdate /force` |
| `gpresult /r` mostra `N/A` do lado User mesmo depois de mover a OU | Teste foi feito com conta **local**, não de domínio | Refazer o teste logado com uma conta `REDE\usuario` |
| Setup Key para de funcionar depois de 1 uso | Key criada sem marcar "Reusable" | Sempre usar uma Setup Key marcada como reutilizável e sem limite |
| `netbird up` conecta no IP público da própria VPS em vez do NetBird Cloud | Perfil salvo com Management URL customizada errada | Apagar `C:\ProgramData\Netbird\*.json`, reiniciar o serviço, reconectar com `--management-url https://api.netbird.io:443` explícito |

---

## Checklist rápido por VPS

- [ ] Baixar o instalador do NetBird **manualmente** pelo link (nunca via script/`Invoke-WebRequest`) e confirmar o tamanho do arquivo
- [ ] Instalar o cliente NetBird
- [ ] `netbird up --setup-key SUA-SETUP-KEY`
- [ ] `Set-Service netbird -StartupType Automatic`
- [ ] Confirmar o nome real da interface de rede (`Get-NetAdapter`, ignorando `wt0`) antes de configurar o DNS
- [ ] DNS duplo: `100.78.25.61` (serversat via NetBird) + `1.1.1.1` (fallback público) — nunca só um DNS interno; e não usar `192.168.1.100` salvo rota de rede confirmada
- [ ] Configurar auto-restart do serviço `netbird` (`sc.exe failure netbird ...`)
- [ ] Validar `nslookup` + `ping` + `netbird status`
- [ ] `Add-Computer -DomainName "REDE.COM"`
- [ ] Confirmar `CsDomain` após reboot + `Test-ComputerSecureChannel` (`-Repair` ou reingresso forçado se necessário)
- [ ] Mover/confirmar o objeto de computador na OU `SERVIDORES_CONTABO`
- [ ] `gpupdate /force` + `gpresult /r` confirmando a GPO de RDP (lado Computer)
- [ ] Adicionar usuários autorizados ao grupo `SERVIDORES_CONTABO` (ou diretamente em Remote Desktop Users)
- [ ] Testar RDP com conta de **domínio** e validar GPO de usuário/papel de parede no `gpresult /r`
