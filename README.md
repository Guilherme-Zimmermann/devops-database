# Documentação: Deploy de PostgreSQL em VPS com Docker (Rocky Linux)

1. Introdução
Este documento descreve o processo completo para provisionamento e configuração de um servidor de banco de dados PostgreSQL em um container Docker. A infraestrutura utilizada é uma VPS Linux com Rocky Linux, 4GB de RAM, 50GB de armazenamento e 1 núcleo de CPU. O objetivo é criar um ambiente seguro, persistente e com capacidade de testes de integridade, segurança e restauração de dados.

***ATENÇÃO***
Existem placeholders nesse documento, textos que estiverem dessa forma <EXEMPLO>, entenda como placeholder e o substitua pelo valor correto da sua vps

3. Requisitos
- VPS rodando *Rocky Linux*
- Acesso root ou privilégios sudo

Firewall configurado corretamente
Acessando a VPS
```Abra o seu bash e insira:  ssh root@<seu ip>```
Após isso a senha da sua vps

3. Instalação do Docker e Docker Compose
```
sudo dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io
sudo systemctl enable --now docker
```

Verifique a instalação:
```docker --version```
```
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

```docker-compose version```

4. Configuração do PostgreSQL no Docker
Criar diretório do projeto
```mkdir -p ~/postgres && cd ~/postgres```

Criar arquivo .env para armazenar credenciais
```nano .env```

Adicione as variáveis de ambiente:
Entenda todos os 3 valores como exemplos e substitua-os pelos valores verdadeiros

```
POSTGRES_USER=<SUBSTITUA>
POSTGRES_PASSWORD=<SUBSTITUA>
POSTGRES_DB=<SUBSTITUA>
```

Salve e feche (Ctrl+X, Y, Enter).

para verificar use
```cat .env```

Criar arquivo docker-compose.yml
```nano docker-compose.yml```

Adicione o seguinte conteúdo:
```
services:
  postgres:
    image: postgres:latest
    container_name: postgres_container
    restart: always
    ports:
      - "5432:5432"
    env_file:
      - .env
    volumes:
      - ./pgdata:/var/lib/postgresql/data
```

Salve e feche (Ctrl+X, Y, Enter).
para verificar use
```cat docker-compose.yml```

Subir o container PostgreSQL
```docker compose up -d```

5. Configurar Firewall
Antes de adicionar regras ao firewall, certifique-se de que o serviço está instalado e rodando:

```
sudo dnf install -y firewalld
sudo systemctl enable --now firewalld
sudo systemctl status firewalld
```

Se o serviço estiver ativo, adicione a regra para liberar a porta 5432:

```
sudo firewall-cmd --add-port=5432/tcp --permanent
sudo firewall-cmd --reload
```

6. Testando a Conexão
A equipe pode acessar o PostgreSQL remotamente usando:
```psql -h <IP_DA_VPS> -U <USER_DO_BANCO> -d <DATABASE>```

Ou usando clientes como DBeaver ou PgAdmin, apontando para <IP_DA_VPS> na porta 5432.

7. Manutenção e Gerenciamento
Verificar logs do PostgreSQL
```docker logs postgres_container```

Reiniciar o container
```docker restart postgres_container```

Parar e remover o container
```docker compose down```

8. Configurar SSL simples no PostgreSQL
- Essa configuração de SSL é apenas a nível de servidor, o cliente não precisa de um certificado para conectar ao banco. 
   
1.  Gerar certificado SSL (autoassinado)
No servidor (sua VPS):
```mkdir -p ~/postgres-certs && cd ~/postgres-certs```

```
openssl req -new -x509 -days 365 \
  -nodes \
  -out server.crt \
  -keyout server.key \
  -subj "/CN=postgres-server"

chmod 600 server.key
chmod 644 server.crt

chown 999:999 </CAMINHO_COMPLETO>/server.key
chmod 600 </CAMINHO_COMPLETO>/server.key
```

2.  Atualizar seu docker-compose.yml
Atualize o docker-compose.yml do PostgreSQL para montar os certificados no container e ativar o SSL:

```
services:
  postgres:
    image: postgres:latest
    container_name: postgres_container
    restart: always
    ports:
      - "5432:5432"
    env_file:
      - .env
    volumes:
      - ./pgdata:/var/lib/postgresql/data
      - ./postgres-certs/server.crt:/var/lib/postgresql/server.crt:ro
      - ./postgres-certs/server.key:/var/lib/postgresql/server.key:ro
    command: >
      postgres -c ssl=on
               -c ssl_cert_file=/var/lib/postgresql/server.crt
               -c ssl_key_file=/var/lib/postgresql/server.key
```

3.  Subir o container com SSL
Na pasta onde está o docker-compose.yml:
```
docker compose down
docker compose up -d
```

Criação do usuário de Leitura
Acesse o PostgreSQL como superusuário ou com um usuário com permissão para criar roles:

```docker exec -it postgres_container psql -U <USUARIO_DO_BANCO> -d <DATABASE>```

Dentro do prompt do PostgreSQL:

```
-- Cria o usuário somente leitura
CREATE USER <USUARIO_DO_BANCO> WITH PASSOWORD '<PASSWORD>;
```

```
-- Concede acesso de leitura na tabela clientes
GRANT CONNECT ON DATABASE <DATABASE> TO <USUARIO_DO_BANCO>;
GRANT USAGE ON SCHEMA public TO <USUARIO_DO_BANCO>;
GRANT SELECT ON <TABLE> TO <USUARIO_DO_BANCO>;
```

9. Backup
```
mkdir -p /root/backups/tabela-fipe
docker exec -t postgres_container pg_dump -U <USUARIO_DO_BANCO> -d <DATABASE> -F c -f /tmp/tabela-fipe.dump
docker cp postgres_container:/tmp/tabela-fipe.dump /root/backups/tabela-fipe/
```

10. Restaurar backup
Copiar o arquivo .dump da VPS para dentro do container

```
docker cp /root/backups/tabela-fipe/tabela-fipe.dump postgres_container:/tmp/tabela-fipe.dump
```

Restaurar o backup dentro do container

```
docker exec -t postgres_container pg_restore -U <USUARIO_DO_BANCO> -d tabela-fipe /tmp/tabela-fipe.dump
```

11. Monitoramento
Verificar uso de CPU, memória e espaço em disco
Para monitorar o consumo de recursos do sistema:

```
top          # Exibe processos ativos e uso da CPU/memória
free -h      # Exibe uso da memória
vmstat 1 10  # Mostra estatísticas do sistema a cada 1 segundo, 10 vezes
df -h        # Exibe espaço em disco
```

Para monitorar containers do Docker:

```
docker stats  # Exibe uso de CPU/memória por container
docker ps -a  # Lista containers em execução e parados
```

12. Considerações Finais
Esse procedimento garante um PostgreSQL funcional e seguro rodando em um ambiente Docker no Rocky Linux. Caso precise de persistência dos dados, o volume pgdata garantirá que os dados permaneçam após reinicializações do container, caso necessário, com o backup realizado, você consegue fazer a recuperação se dados forem perdidos.

Alunos:
Guilherme Will Zimmermann
Luiz Gustavo Kolben
Pablo Pereira
João Vitor Alencar
Joenerson
