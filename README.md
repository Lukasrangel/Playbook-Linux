# PLAYBOOK — INFRAESTRUTURA LINUX

Comandos e procedimentos rápidos para administração de aplicações em VPS/Linux.

---

#  DOCKER

## Redes

### Criar uma rede Docker externa

Útil quando diferentes projetos Docker Compose precisam compartilhar uma rede específica.

```bash
docker network create database-admin
```

No `docker-compose.yml`:

```yaml
networks:
  database-admin:
    external: true
```

### Listar redes

```bash
docker network ls
```

### Inspecionar uma rede

```bash
docker network inspect database-admin
```

### Desconectar um container de uma rede

```bash
docker network disconnect <rede> <container>
```

Exemplo:

```bash
docker network disconnect luksdev_luksdev-net phpmyadmin
```

### Ver quais containers estão conectados a uma rede

```bash
docker network inspect <rede> \
  --format '{{range .Containers}}{{.Name}}{{"\n"}}{{end}}'
```

---

## Containers

### Remover container preso

```bash
docker rm -f <container>
```

> Remove o container, mas não remove dados armazenados em volumes ou diretórios bind-mounted.

### Recriar uma stack

```bash
docker compose down
docker compose up -d --force-recreate
```

> Não utilizar `docker compose down -v` se a intenção for preservar volumes.

Para reconstruir também as imagens:

```bash
docker compose up -d --build --force-recreate
```

---

### Verificar configuraões de um container (para recriar docker-compose por exemplo)

```bash
docker inspect uptime-kuma \
    --format 'Image: {{.Config.Image}}
Ports: {{json .HostConfig.PortBindings}}
Mounts: {{json .Mounts}}
Restart: {{.HostConfig.RestartPolicy.Name}}'
```

# MYSQL / MARIADB

## Restaurar SQL diretamente para um container

### Método rápido

```bash
docker exec -i <container-db> \
  mariadb -u <usuario> -p <nome-do-db> \
  < /caminho/arquivo.sql
```

Exemplo:

```bash
docker exec -i luksdev-db \
  mariadb -u luksdev -p luksdev \
  < /home/debina/setembro-2026-luksdev.sql
```

O `-p` sem senha faz o MariaDB solicitar a senha.

> Evitar colocar a senha diretamente no comando (`-pSENHA`), pois ela pode ficar registrada no histórico do shell ou aparecer em processos.

---

## Fazer o dump de database do container

```bash
docker exec mariadb mariadb-dump -u root -p \
  --databases luksdev sitenovo \
  --routines \
  --triggers \
  --events \
  > backup-apps-$(date +%F).sql
```



## Criar usuario e banco de dados 

```bash
docker exec -it mariadb mariadb -u root -p
```


```bash
CREATE DATABASE novosite
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

CREATE USER 'novosite'@'%'
    IDENTIFIED BY 'uma-senha-bem-forte';

GRANT ALL PRIVILEGES
    ON novosite.*
    TO 'novosite'@'%';

FLUSH PRIVILEGES;
```

---

## Resetar completamente um banco

Apaga todas as tabelas e recria o banco.

```bash
docker exec -it <container-db> \
  mariadb -u root -p \
  -e "DROP DATABASE <nome-do-db>; CREATE DATABASE <nome-do-db> CHARACTER SET utf8 COLLATE utf8_general_ci;"
```

⚠️ **DESTRUTIVO:** todos os dados do banco serão perdidos.

---

## Converter charset/collation de um dump SQL

Antes de modificar, fazer backup:

```bash
cp arquivo.sql arquivo.sql.bak
```

Converter `utf8mb4_0900_ai_ci`:

```bash
sed -i \
  -e 's/utf8mb4_0900_ai_ci/utf8_general_ci/g' \
  arquivo.sql
```

Converter `utf8mb4`:

```bash
sed -i \
  -e 's/utf8mb4_0900_ai_ci/utf8_general_ci/g' \
  -e 's/utf8mb4/utf8/g' \
  arquivo.sql
```

> A ordem importa: a collation específica precisa ser substituída antes de substituir `utf8mb4` genericamente.

---

# 🔐 SSH

Enviar arquivo de máquina local ara vps via ssh

```bash
scp arquivo.txt usuario@servidor:/caminho/destino/

```


---

## Criar túnel SSH

Exemplo para acessar um serviço que está disponível apenas em `127.0.0.1:8080` no VPS:

```bash
ssh -L 8080:127.0.0.1:8080 usuario@seu-vps
```

Depois acessar localmente:

```text
http://localhost:8080
```

### Fluxo

```text
Seu computador
      │
      │ SSH tunnel
      ▼
VPS :8080
      │
      ▼
Serviço local
```

Útil para ferramentas administrativas como phpMyAdmin que não precisam ficar expostas à Internet.

---

#  NGINX

## Habilitar um Virtual Host

Criar/configurar o arquivo:

```text
/etc/nginx/sites-available/domain-name.com
```

Criar o link simbólico:

```bash
ln -s /etc/nginx/sites-available/domain-name.com \
      /etc/nginx/sites-enabled/
```

### Testar configuração

```bash
nginx -t
```

### Recarregar Nginx

```bash
systemctl reload nginx
```

> `reload` aplica alterações sem derrubar as conexões existentes.

### Reiniciar Nginx

```bash
systemctl restart nginx
```

Usar quando realmente for necessário reiniciar o processo.

---

## Debug de logs conexão ip que chegam no nginx

No nginx.conf dentro de http{}

```bash
log_format realip_debug
    'remote=$remote_addr '
    'original=$realip_remote_addr '
    'cf=$http_cf_connecting_ip '
    'xff=$http_x_forwarded_for '
    'host=$host '
    'request="$request"';
```

No arquivo do site em server{}

```bash
access_log /var/log/nginx/neotube/realip-debug.log realip_debug;
``

---
#  LINUX

## Adicionar usuário ao grupo sudo

```bash
sudo usermod -aG sudo <usuario>
```

Exemplo:

```bash
sudo usermod -aG sudo kemper
```

### Aplicar o novo grupo na sessão atual

```bash
newgrp sudo
```

Alternativamente, sair da sessão e entrar novamente.

### Ver grupos do usuário

```bash
groups <usuario>
```

---

# 🌿 GIT

## Branches

### Listar branches locais

```bash
git branch
```

### Criar e mudar para uma nova branch

```bash
git switch -c <nova_branch>
```

Exemplo:

```bash
git switch -c dockerizada
```

### Mudar para uma branch existente

```bash
git switch <branch>
```

Exemplo:

```bash
git switch main
```

### Excluir uma branch

```bash
git branch -d <branch>
```

Forçar exclusão:

```bash
git branch -D <branch>
```

### Ver branch atual

```bash
git branch --show-current
```

---

## Staging

### Adicionar arquivos

```bash
git add <arquivo>
```

Todos:

```bash
git add .
```

### Desfazer `git add`

Manter as alterações, mas retirar do staging:

```bash
git restore --staged <arquivo>
```

Todos:

```bash
git restore --staged .
```

---

## Commit

### Criar commit

```bash
git commit -m "mensagem do commit"
```

### Ver estado do repositório

```bash
git status
```

### Ver histórico

```bash
git log --oneline
```

---

## Remote

### Ver repositórios remotos

```bash
git remote -v
```

### Adicionar remote

```bash
git remote add origin <URL>
```

### Enviar branch para o remote

```bash
git push -u origin <branch>
```

Exemplo:

```bash
git push -u origin dockerizada
```

Depois disso, normalmente basta:

```bash
git push
```




---
#  UFW — FIREWALL

## Configuração básica

Política padrão:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

Permitir SSH:

```bash
sudo ufw allow 22/tcp
```

Permitir HTTP:

```bash
sudo ufw allow 80/tcp
```

Permitir HTTPS:

```bash
sudo ufw allow 443/tcp
```

Ativar:

```bash
sudo ufw enable
```

Ver status:

```bash
sudo ufw status verbose
```

### Regras básicas

```text
Entrada  → bloqueada por padrão
Saída    → permitida por padrão

22/tcp   → SSH
80/tcp   → HTTP
443/tcp  → HTTPS
```

 Em VPS remoto, **liberar SSH antes de ativar o UFW**. Idealmente manter a sessão SSH atual aberta e testar uma segunda conexão antes de encerrar a primeira.

---

## UFW como serviço

O UFW possui integração com `systemd`.

Verificar:

```bash
sudo systemctl status ufw
```

Habilitar no boot:

```bash
sudo systemctl enable ufw
```

> Normalmente o `ufw enable` já deixa o firewall configurado para iniciar automaticamente.

---

#  COMANDOS DE DIAGNÓSTICO RÁPIDO

## Docker

```bash
docker ps
docker ps -a
docker network ls
docker network inspect <rede>
docker logs <container>
docker inspect <container>
```

## Nginx

```bash
nginx -t
systemctl status nginx
journalctl -u nginx
```

## Serviços

```bash
systemctl status <serviço>
systemctl restart <serviço>
systemctl reload <serviço>
```

## Portas abertas

```bash
ss -tulpn
```

Somente TCP:

```bash
ss -tlpn
```

Somente UDP:

```bash
ss -ulpn
```

---

#  REGRA GERAL

Antes de executar qualquer comando destrutivo:

```text
docker rm -f
docker compose down -v
DROP DATABASE
rm -rf
ufw enable
```

confirmar:

1. O que será apagado/alterado?
2. Existe backup?
3. Estou conectado remotamente?
4. Posso perder acesso ao servidor?
5. Existe algum container/serviço dependendo desse recurso?
