# DBGuard Agent — instalação em Linux (MySQL)

Este guia é para quem vai **instalar e usar** o DBGuard Agent num servidor Linux — não é preciso
conhecimento de programação, só acesso `root`/`sudo` ao servidor.

O DBGuard Agent é a versão do DBGuard pra servidores Linux: faz backup de **um servidor MySQL**,
sem interface gráfica. Ele não fica residente em memória — roda uma vez, faz o backup, envia
e-mail de aviso (se você configurar), e encerra. Quem decide a periodicidade (todo dia às 2h da
manhã, por exemplo) é o `systemd` do próprio Linux, configurado no Passo 6.

Siga os passos na ordem. Cada passo tem um jeito de conferir se deu certo antes de ir pro próximo.

## Antes de começar: o que você precisa

- Um servidor Linux com acesso `root` (ou `sudo`).
- O cliente MySQL instalado nesse servidor. Confira com:
  ```bash
  which mysqldump mysql
  ```
  Se não aparecer nada, instale primeiro (`apt install mariadb-client` ou `yum install mysql` /
  `mariadb` — depende da distro).
- Os dois arquivos da release do DBGuard Agent: `DBGuard.Agent` (o programa) e `appsettings.json`
  (arquivo de configuração de exemplo). Baixe em
  https://github.com/ricardogapweb/dbguard-releases/releases (pegue os dois arquivos anexados na
  versão mais recente).
- Os dados de conexão do MySQL que será copiado (host, porta, nome do banco, usuário e senha de
  backup — ver Passo 7 se ainda não tem um usuário dedicado pra isso).

## Passo 1 — Copiar os arquivos pro servidor

Com os dois arquivos baixados (`DBGuard.Agent` e `appsettings.json`) já na máquina/pasta certa:

```bash
sudo mkdir -p /opt/dbguard-agent
sudo cp DBGuard.Agent appsettings.json /opt/dbguard-agent/
sudo chmod +x /opt/dbguard-agent/DBGuard.Agent
```

Crie um usuário próprio do sistema pra rodar o agente (mais seguro que rodar como `root`):
```bash
sudo useradd -r -s /usr/sbin/nologin dbguard || true
sudo chown -R dbguard:dbguard /opt/dbguard-agent
```

## Passo 2 — Configurar `appsettings.json`

Abra o arquivo pra editar:
```bash
sudo nano /opt/dbguard-agent/appsettings.json
```

Preencha com os dados do seu ambiente (não coloque senha aqui — isso é no Passo 4):

```json
{
  "Mysql": {
    "Host": "seu-host",
    "Port": 3306,
    "Database": "seu_banco",
    "User": "backup_user"
  },
  "Job": {
    "JobName": "Backup-Producao",
    "DestinationPath": "/var/backups/dbguard",
    "RetentionCount": 7,
    "CompressBackup": true
  },
  "Smtp": {
    "Server": "smtp.seudominio.com",
    "Port": 587,
    "FromAddress": "backup@seudominio.com",
    "ToAddresses": "voce@seudominio.com",
    "NotifyOnFailure": true
  }
}
```

**Cuidado com um erro comum:** todo valor de texto precisa estar entre aspas duplas (`"assim"`).
Se você escrever `"Server": smtp.seudominio.com,` sem as aspas, o agente não consegue nem iniciar
e mostra um erro tipo `'s' is an invalid start of a value`. Revise o arquivo se receber esse erro.

Não quer receber e-mail? Apague a seção `"Smtp"` inteira (ou deixe como está, sem preencher — se
`Smtp.Server` ficar vazio o agente simplesmente não tenta enviar).

O diretório de destino do backup precisa poder ser escrito pelo usuário `dbguard`:
```bash
sudo mkdir -p /var/backups/dbguard
sudo chown dbguard:dbguard /var/backups/dbguard
```

## Passo 3 — Criar um usuário MySQL só pra backup (recomendado)

Em vez de usar o usuário `root` do MySQL, crie um usuário com só as permissões necessárias.
Conecte no MySQL como administrador e rode:

```sql
CREATE USER IF NOT EXISTS 'backup_user'@'%' IDENTIFIED BY 'escolha-uma-senha-forte';
GRANT SELECT, LOCK TABLES, SHOW VIEW, TRIGGER ON seu_banco.* TO 'backup_user'@'%';
FLUSH PRIVILEGES;
```

Troque `'%'` pelo endereço real do servidor onde o Agent roda, se souber (mais seguro que liberar
de qualquer lugar). Troque também `seu_banco` pelo nome real do banco, e use esse mesmo usuário no
`appsettings.json` do Passo 2 (campo `Mysql.User`).

## Passo 4 — Guardar as senhas com segurança

As senhas **nunca** vão dentro do `appsettings.json` — ficam num arquivo separado, com permissão
restrita, que só o próprio sistema consegue ler:

```bash
sudo install -o root -g dbguard -m 600 /dev/null /etc/dbguard-agent.env
sudo nano /etc/dbguard-agent.env
```

Digite (repare que é `Mysql` com **dois** underscores antes de `Password`):
```
Mysql__Password=senha-real-do-banco
Smtp__Password=senha-real-do-smtp
```

Salve e feche. Não é preciso mexer na permissão desse arquivo além do que o comando acima já fez.

## Passo 5 — Testar antes de agendar

Antes de deixar isso rodando sozinho todo dia, teste uma vez na mão pra garantir que está tudo
certo:

```bash
cd /opt/dbguard-agent
set -a
source /etc/dbguard-agent.env
set +a
./DBGuard.Agent
```

O que cada resultado quer dizer:

- **Terminou sem erro e apareceu um arquivo novo em `/var/backups/dbguard`** → deu certo, siga pro
  Passo 6.
- **`libstdc++.so.6: version 'GLIBCXX_...' not found`** → seu servidor usa uma versão de Linux
  antiga demais (comum em CentOS/RHEL 7). Vá direto pra seção **"Meu servidor é antigo
  (CentOS/RHEL 7): erro GLIBCXX"** no final deste guia antes de continuar.
- **Erro mencionando `JSON`** → volte ao Passo 2, algum valor de texto está sem aspas.
- **`Access denied ... (using password: NO)`** → o comando `source` acima não rodou, ou o arquivo
  do Passo 4 tem o nome da variável errado (confira os dois underscores em `Mysql__Password`).
- **`Access denied ... (using password: YES)`** → a senha no arquivo do Passo 4 está errada.
- **`Host '...' is not allowed to connect`** → o usuário MySQL do Passo 3 não tem permissão pra
  conectar a partir desse servidor — revise o `'@'%'` (ou o host específico) no `CREATE USER`.

## Passo 6 — Agendar a execução automática

Crie o arquivo `/etc/systemd/system/dbguard-agent.service`:
```bash
sudo nano /etc/systemd/system/dbguard-agent.service
```
com este conteúdo:
```ini
[Unit]
Description=DBGuard Agent - backup MySQL (execução única, disparada pelo dbguard-agent.timer)
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
User=dbguard
Group=dbguard
WorkingDirectory=/opt/dbguard-agent
ExecStart=/opt/dbguard-agent/DBGuard.Agent
EnvironmentFile=/etc/dbguard-agent.env
```

> Se seu servidor precisou da correção da seção "Meu servidor é antigo" (CentOS/RHEL 7), adicione
> mais esta linha logo abaixo de `EnvironmentFile=`:
> ```ini
> Environment="LD_LIBRARY_PATH=/opt/dbguard-agent/libs"
> ```

Crie também `/etc/systemd/system/dbguard-agent.timer`:
```bash
sudo nano /etc/systemd/system/dbguard-agent.timer
```
com este conteúdo (por padrão roda todo dia às 2h da manhã — troque o horário em `OnCalendar` se
quiser outro):
```ini
[Unit]
Description=Agenda a execução diária do dbguard-agent.service

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

Ative tudo:
```bash
sudo systemctl daemon-reload

# dispara uma execução imediata, pra confirmar que funciona também via systemd:
sudo systemctl start dbguard-agent.service
sudo journalctl -u dbguard-agent.service -n 50 --no-pager
```

Se o log mostrar sucesso, habilite o agendamento definitivo:
```bash
sudo systemctl enable --now dbguard-agent.timer
systemctl list-timers dbguard-agent.timer
```
O último comando mostra quando será a próxima execução — pronto, o backup automático está
configurado.

## Erros comuns

| Sintoma | Causa | O que fazer |
|---|---|---|
| `libstdc++.so.6: version 'GLIBCXX_...' not found` | Servidor com versão de Linux antiga (CentOS/RHEL 7) | Ver seção "Meu servidor é antigo" abaixo |
| Erro mencionando `JSON`/`is an invalid start of a value` | Falta aspas em algum valor de texto no `appsettings.json` | Revise o Passo 2 |
| `Access to the path '...' is denied` | Pasta de destino do backup não pertence ao usuário `dbguard` | `sudo chown dbguard:dbguard` na pasta (Passo 2) |
| `Got error: 1130: Host '...' is not allowed to connect` | Usuário MySQL sem permissão pra esse servidor | Revise o `CREATE USER` do Passo 3 |
| `Got error: 1045: Access denied ... (using password: NO)` | Senha não chegou ao programa | Revise o Passo 4/5 |
| `Got error: 1045: Access denied ... (using password: YES)` | Senha errada | Revise a senha no arquivo do Passo 4 |
| Não chega e-mail quando o backup dá certo | Por padrão só envia e-mail quando **falha** | Ajuste `NotifyOnSuccess`/`NotifyOnFailure` no `appsettings.json` |
| `535: Incorrect authentication data` (erro de e-mail) | Senha de e-mail errada no arquivo do Passo 4 | Confira `Smtp__Password` |

## Meu servidor é antigo (CentOS/RHEL 7): erro GLIBCXX

Alguns servidores Linux mais antigos (CentOS 7, RHEL 7 e parecidos) não têm uma peça interna
chamada `libstdc++` atualizada o suficiente pro DBGuard Agent funcionar. Isso não é um defeito do
DBGuard — é uma limitação conhecida dessas versões de Linux, que já não recebem mais atualizações
do fabricante (CentOS 7 chegou ao fim do suporte em 30/06/2024).

**Importante:** nunca baixe um arquivo "`.so`" de um site desconhecido pra resolver isso — seria
instalar um componente sem procedência conhecida dentro de todo o sistema, o que é um risco de
segurança real (esse componente passaria a rodar dentro de qualquer outro programa do servidor
que use a mesma peça, incluindo o próprio banco de dados). A receita abaixo usa uma fonte confiável
(Anaconda, empresa conhecida de ferramentas de dados) e instala a peça **isolada**, só pra uso do
DBGuard Agent, sem alterar nada mais no servidor.

```bash
cd /tmp
curl -LO https://conda.anaconda.org/anaconda/linux-64/libstdcxx-ng-9.3.0-hd4cf53a_17.tar.bz2
mkdir conda_extract && cd conda_extract
tar -xjf ../libstdcxx-ng-9.3.0-hd4cf53a_17.tar.bz2

# confirma que essa versão funciona no seu servidor (não deve aparecer nenhum "not found"):
ldd ./lib/libstdc++.so.6.0.28
```

Se o comando `ldd` não mostrar nenhum "not found", prossiga:

```bash
sudo mkdir -p /opt/dbguard-agent/libs
sudo cp ./lib/libstdc++.so.6.0.28 /opt/dbguard-agent/libs/
sudo ln -s libstdc++.so.6.0.28 /opt/dbguard-agent/libs/libstdc++.so.6
sudo chown -R dbguard:dbguard /opt/dbguard-agent/libs

# testa de novo (Passo 5), agora indicando onde está a peça nova:
cd /opt/dbguard-agent
LD_LIBRARY_PATH=/opt/dbguard-agent/libs ./DBGuard.Agent
```

Se funcionar, volte pro Passo 6 e não esqueça de adicionar a linha `Environment=` indicada lá
dentro do arquivo `.service`, senão o agendamento automático volta a dar o mesmo erro.

Se mesmo assim aparecer um erro parecido (agora mencionando `GLIBC` em vez de `GLIBCXX`), esse
servidor é antigo demais até pra essa solução — nesse caso, fale com o suporte do DBGuard pra
avaliar rodar o Agent dentro de um container (Docker), que resolve isso de forma definitiva.

## Escopo atual

O DBGuard Agent hoje cobre só **MySQL**. Suporte a SQL Server/PostgreSQL, backup de múltiplos
bancos numa única conexão, e pacotes prontos `.deb`/`.rpm` ainda não estão disponíveis nesta
versão.
