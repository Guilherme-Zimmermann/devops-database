# Documentação: Deploy de PostgreSQL em VPS com Docker (Rocky Linux)
1. Introdução
## Este documento fornece um guia detalhado para configurar um servidor PostgreSQL em uma VPS rodando Rocky Linux utilizando Docker e Docker Compose.
2. Requisitos
VPS rodando Rocky Linux


Acesso root ou privilégios sudo

Firewall configurado corretamente

Acessando a VPS
Abra o seu bash e insira:  ```ssh root@<seu ip>```

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
POSTGRES_USER={substitua}
POSTGRES_PASSWORD={substitua}
POSTGRES_DB={substitua}
```

Salve e feche (Ctrl+X, Y, Enter).
para verificar use
```cat .env```
Criar arquivo docker-compose.yml
```nano docker-compose.yml```

Adicione o seguinte conteúdo:
```
version: '3.8'

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
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
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

8. Considerações Finais
Esse procedimento garante um PostgreSQL funcional e seguro rodando em um ambiente Docker no Rocky Linux. Caso precise de persistência dos dados, o volume pgdata garantirá que os dados permaneçam após reinicializações do container.

