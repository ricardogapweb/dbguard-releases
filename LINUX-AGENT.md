# DBGuard Agent — instalação em Linux (MySQL)

O `DBGuard Agent` é a versão headless do DBGuard pra servidores Linux: faz backup de **um servidor MySQL**, sem interface gráfica. 
Diferente da versão Windows (Desktop + Service), ele não fica residente em memória nem tem agendador próprio — roda uma vez, faz o backup, 
envia e-mail se configurado, e encerra. Quem decide a periodicidade é o `cron`/`systemd timer` do próprio Linux.

Baixe `DBGuard.Agent` e `appsettings.json` (exemplo) dos assets desta release.

## 1. Pré-requisitos

- `mysqldump` e `mysql` no `PATH` do sistema (pacote `mysql-client` ou `mariadb-client`).
- Nenhum requisito de runtime .NET — o binário é self-contained (.NET 8 embutido).

## 2. Instalar

\```bash
sudo mkdir -p /opt/dbguard-agent
sudo cp DBGuard.Agent appsettings.json /opt/dbguard-agent/
sudo chmod +x /opt/dbguard-agent/DBGuard.Agent

sudo useradd -r -s /usr/sbin/nologin dbguard || true
sudo chown -R dbguard:dbguard /opt/dbguard-agent
\```

## 3. Configurar (`appsettings.json`)

Edite `/opt/dbguard-agent/appsettings.json` com os dados **não sensíveis** (nunca coloque senha
aqui — não existem campos de senha neste arquivo, de propósito):

\```json
{
  "Mysql": { "Host": "seu-host", "Port": 3306, "Database": "seu_banco", "User": "backup_user" },
  "Job": { "JobName": "Backup-Producao", "DestinationPath": "/var/backups/dbguard", "RetentionCount": 7, "CompressBackup": true },
  "Smtp": { "Server": "smtp.seudominio.com", "Port": 587, "FromAddress": "backup@seudominio.com", "ToAddresses": "voce@seudominio.com", "NotifyOnFailure": true }
}
\```

`Job.DestinationPath` precisa ser gravável pelo usuário `dbguard`:
\```bash
sudo mkdir -p /var/backups/dbguard
sudo chown dbguard:dbguard /var/backups/dbguard
\```

Se não quiser notificação por e-mail, apague ou deixe `Smtp.Server` vazio/nulo.

## 4. Segredos (senha do MySQL e do SMTP)

As senhas nunca ficam no `appsettings.json` — só em variável de ambiente, injetada pelo `systemd`
a partir de um arquivo com permissão restrita (`600`, dono `root:dbguard`):

\```bash
sudo install -o root -g dbguard -m 600 /dev/null /etc/dbguard-agent.env
sudo nano /etc/dbguard-agent.env
\```
Conteúdo (dois underscores entre seção e chave):
\```
Mysql__Password=senha-real-do-banco
Smtp__Password=senha-real-do-smtp
\```

## 5. Unidades systemd

`/etc/systemd/system/dbguard-agent.service`:
\```ini
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
\```

`/etc/systemd/system/dbguard-agent.timer`:
\```ini
[Unit]
Description=Agenda a execução diária do dbguard-agent.service

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
\```

Ative:
\```bash
sudo systemctl daemon-reload
sudo systemctl start dbguard-agent.service   # dispara uma execução imediata pra validar
sudo journalctl -u dbguard-agent.service -n 50 --no-pager

# se o log acima mostrar sucesso, agenda pra valer:
sudo systemctl enable --now dbguard-agent.timer
\```

## 6. Privilégios MySQL mínimos

\```sql
CREATE USER IF NOT EXISTS 'backup_user'@'%' IDENTIFIED BY 'senha-real-do-banco';
GRANT SELECT, LOCK TABLES, SHOW VIEW, TRIGGER ON seu_banco.* TO 'backup_user'@'%';
FLUSH PRIVILEGES;
\```
Troque `'%'` pelo host/faixa real de onde o agente se conecta, mais restrito em produção.

> `PROCESS` é um privilégio só **global** — `GRANT ... PROCESS ON seu_banco.* ...` falha com
> `Error 1221`, e de qualquer forma não é necessário pro `mysqldump` funcionar.

## 7. Erros comuns

| Sintoma | Causa | Solução |
|---|---|---|
| `Access to the path '...' is denied` | Destino não é gravável pelo usuário `dbguard` | `chown dbguard:dbguard` no diretório de destino |
| `Got error: 1130: Host '...' is not allowed to connect` | Usuário MySQL sem grant pro host do agente | `CREATE USER 'user'@'<host>' ...` (seção 6) |
| `Got error: 1045: Access denied ... (using password: YES)` | Senha errada ou `Mysql__Password` não chegou ao processo | Confira `/etc/dbguard-agent.env` e o `EnvironmentFile=` no `.service` |
| Sem e-mail em backup bem-sucedido | `NotifyOnSuccess` é `false` por padrão | Ajuste em `appsettings.json` |

## Escopo atual

Só MySQL. SQL Server/PostgreSQL, multi-banco por conexão e empacotamento `.deb`/`.rpm` ainda não
estão disponíveis no Agent.
