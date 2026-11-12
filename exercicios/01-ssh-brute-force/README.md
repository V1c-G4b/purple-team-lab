# 01: Força bruta em SSH

| | |
|---|---|
| Data | 2026-11-11 |
| Técnica (MITRE ATT&CK) | [T1110.001 Brute Force: Password Guessing](https://attack.mitre.org/techniques/T1110/001/) |
| Alvo | Ubuntu Server 26.04 (`10.10.10.3`) |
| Atacante | Kali Linux 2026.3 (`10.10.10.4`) |
| Resultado | Conta `suporte` comprometida em 4 segundos, contida e investigada |

> Exercício feito no meu lab isolado, contra máquinas minhas. Arquitetura em [`lab/arquitetura.md`](../../lab/arquitetura.md).

## Cenário

Uma conta de suporte criada "só para um teste" e esquecida no servidor, com uma senha fraca. É um caso comum em empresa: ninguém lembra que a conta existe, e ela fica acessível por SSH como qualquer outra.

| Item | Configuração |
|---|---|
| Rede | Rede NAT do VirtualBox `labnet` (`10.10.10.0/24`), isolada da rede de casa |
| Conta vítima | `suporte`, senha `princess` (fraca de propósito) |
| Serviço exposto | SSH na porta 22, aceitando login por senha, sem limite de tentativas |

## Ataque (lado vermelho)

**Ideia:** um ataque de dicionário. Em vez de testar todas as combinações possíveis, o atacante testa as senhas que mais aparecem em vazamentos reais, começando pelas mais populares.

**1. Wordlist.** Usei as 50 primeiras senhas da `rockyou.txt`, uma lista de 14 milhões de senhas reais vazadas em 2009, ordenada por popularidade. A `princess` é a sexta.

```bash
sudo gunzip /usr/share/wordlists/rockyou.txt.gz
head -n 50 /usr/share/wordlists/rockyou.txt > top50.txt
```

**2. Ataque com o Hydra.**

```bash
hydra -l suporte -P top50.txt -t 4 ssh://10.10.10.3
```

| Parâmetro | Função |
|---|---|
| `-l suporte` | Ataca uma conta fixa |
| `-P top50.txt` | Testa as senhas da lista |
| `-t 4` | 4 tentativas em paralelo (acima disso o SSH começa a recusar conexões) |

**Resultado:** senha encontrada em poucos segundos.

## Detecção (lado azul)

**Ao vivo**, durante o ataque, numa segunda sessão no Ubuntu:

```bash
sudo tail -f /var/log/auth.log
```

O login com sucesso aparece escondido no meio das falhas. Com poucas linhas dá pra ver a olho; com milhares, não. Por isso a investigação usa filtros:

```bash
# logins que deram certo
sudo grep "Accepted password" /var/log/auth.log

# quantas falhas a conta teve
sudo grep "Failed password for suporte" /var/log/auth.log | wc -l
# 8
```

**Evidência** (trecho do `/var/log/auth.log`, horários em UTC):

```
17:42:38 sshd-session[1578]: Failed password for suporte from 10.10.10.4 port 33976 ssh2
17:42:38 sshd-session[1579]: Failed password for suporte from 10.10.10.4 port 33988 ssh2
17:42:40 sshd-session[1577]: Failed password for suporte from 10.10.10.4 port 33972 ssh2
17:42:40 sshd-session[1581]: Failed password for suporte from 10.10.10.4 port 34002 ssh2
17:42:40 sshd-session[1578]: Accepted password for suporte from 10.10.10.4 port 33976 ssh2
17:42:40 sshd-session[1578]: pam_unix(sshd:session): session opened for user suporte(uid=1001)
17:42:42 sshd-session[1581]: Failed password for suporte from 10.10.10.4 port 34002 ssh2
17:42:42 sshd-session[1577]: Failed password for suporte from 10.10.10.4 port 33972 ssh2
17:42:42 sshd-session[1578]: pam_unix(sshd:session): session closed for user suporte
17:42:43 sshd-session[1579]: Failed password for suporte from 10.10.10.4 port 33988 ssh2
```

**O que o rastro mostra:**

| Sinal | Por que indica ataque |
|---|---|
| 8 falhas em 7 segundos na mesma conta | Ninguém erra a própria senha nessa velocidade |
| Todas do mesmo IP | Origem única tentando repetidamente |
| `Accepted password` no meio das falhas | Não foi só tentativa: a conta foi comprometida |
| Falhas **depois** do sucesso | Tentativas em paralelo que já estavam a caminho (`-t 4`) |

**Por que 8 falhas e não 5?** A `princess` era a sexta senha, então seriam 5 erros. Como o Hydra testa 4 senhas ao mesmo tempo, outras tentativas já estavam em andamento quando a certa acertou, e também falharam. O rastro revela até a configuração da ferramenta.

## Linha do tempo

| UTC | Local (UTC-3) | Evento |
|---|---|---|
| 17:13:20 | 14:13:20 | Conta `suporte` criada |
| 17:42:36 | 14:42:36 | Primeira tentativa vinda de `10.10.10.4` |
| 17:42:40 | 14:42:40 | Senha descoberta, sessão aberta |
| 17:42:42 | 14:42:42 | Sessão encerrada pelo atacante (2 segundos dentro) |
| 17:42:44 | 14:42:44 | Últimas conexões do ataque fechadas |
| 17:51:57 | 14:51:57 | Conta travada |

**Tempo de resposta:** 9 minutos entre a invasão e a contenção.

## Resposta ao incidente

**1. Preservar a evidência** antes de alterar qualquer coisa:

```bash
sudo cp /var/log/auth.log ~/evidencia-auth.log
```

**2. Conter.**

```bash
loginctl list-sessions        # o invasor ainda está logado? (não estava)
sudo passwd -l suporte        # trava a senha
sudo usermod -e 1 suporte     # expira a conta
```

`passwd -l` sozinho não basta: ele trava a senha, mas o SSH ainda aceitaria uma chave plantada pelo atacante. Expirar a conta bloqueia qualquer forma de login.

**3. Investigar.**

| Pergunta | Resposta | Como |
|---|---|---|
| Quanto tempo o invasor ficou? | 2 segundos | `session opened` e `session closed` no log |
| Ele deixou persistência? | Não | `/home/suporte/.ssh` não existe, nenhuma chave plantada |
| Ele ainda está dentro? | Não | `loginctl list-sessions` |

O Hydra só confirma a senha e sai. Um atacante real voltaria depois com a senha descoberta, num login limpo, sem nenhuma falha antes, que uma regra de força bruta não pegaria.

## Causas e correções

Qualquer uma destas correções, sozinha, teria impedido o ataque:

| Causa | Correção |
|---|---|
| Senha fraca (6ª da lista mais usada do mundo) | Política de senha forte |
| Qualquer conta pode entrar por SSH | Liberar o SSH só para quem precisa |
| Tentativas ilimitadas | Bloquear o IP depois de algumas falhas |
| SSH aceita login por senha | Exigir chave SSH |
| Conta esquecida no servidor | Remover contas sem uso |

## Achados extras

- **O IP sozinho não separa atacante de administrador.** Meu acesso legítimo passa pelo Kali como bastion, então também aparece no log como `10.10.10.4`. O que diferencia é o padrão: eu entro de primeira, com a minha conta; o atacante errou 8 vezes numa conta de suporte.
- **Ferramentas somem entre versões.** No Ubuntu 26.04, `who` não retornou nada e `last` não vem instalado. A alternativa foi `loginctl` e o próprio `auth.log`.
- **Meu detector ainda não lê esse formato.** O [log-scanner](https://github.com/v1c-g4b/log-scanner) espera uma linha por evento no formato `chave=valor`. O `auth.log` real usa texto livre e várias linhas por evento.

## O que aprendi

<!--
Esta seção é sua. Responda com suas palavras, sem consultar o resto do texto:
- O que mais te surpreendeu no exercício?
- O que você faria diferente na resposta ao incidente?
- Qual conceito da semana 1 apareceu na prática aqui?
-->

## Próximos passos

- [ ] Aplicar as correções, uma de cada vez, e repetir o ataque para ver onde ele é barrado
- [ ] Ensinar o log-scanner a ler o `auth.log` do sshd
- [ ] Criar uma regra para o login limpo que vem depois de uma força bruta bem-sucedida
