# Guia de Deploy para Aplicação Django
Este é um passo a passo completo para realizar o deploy de uma aplicação Django em um servidor Linux (Ubuntu).

## 1. Acesso ao Servidor
Conecte-se ao seu servidor remoto via SSH:
```
ssh SEU_USUARIO@SEU_ENDERECO_DO_SERVIDOR
```

## 2. Atualização do Sistema e Instalação de Dependências
Atualize o sistema e instale os pacotes necessários:
```
sudo apt update -y && sudo apt upgrade -y
sudo apt autoremove -y

# Compiladores e ferramentas de desenvolvimento
sudo apt install build-essential -y

# Python 3.9 e ferramentas relacionadas
sudo apt install python3.9 python3.9-venv python3.9-dev -y

# Web server e HTTPS
sudo apt install nginx -y
sudo apt install certbot python3-certbot-nginx -y

# Banco de dados PostgreSQL
sudo apt install postgresql postgresql-contrib libpq-dev -y

# Git
sudo apt install git -y
```

## 3. Configuração do Banco de Dados PostgreSQL
Acesse o shell do PostgreSQL e configure o banco:
```
sudo -u postgres psql
```
Dentro do prompt psql, execute:
```
-- Crie um usuário administrador para o PostgreSQL
CREATE ROLE nome_do_usuario WITH LOGIN SUPERUSER CREATEDB CREATEROLE PASSWORD senha_forte;

-- Crie o banco de dados
CREATE DATABASE nome_do_banco WITH OWNER nome_do_usuario;

-- Conceda todas as permissões ao usuário
GRANT ALL PRIVILEGES ON DATABASE nome_do_banco TO nome_do_usuario;

\q
```
Reinicie o serviço do PostgreSQL:
```
sudo systemctl restart postgresql
```

## 4. Configuração do Git
Configure suas informações de usuário Git:
```
git config --global user.name "Seu Nome"
git config --global user.email "seu_email@exemplo.com"
```

## 5. Clonando e Preparando o Projeto Django
```
cd /home
git clone URL_DO_REPOSITORIO
cd nome_da_pasta_do_projeto
```
Crie e ative o ambiente virtual:
```
python3.9 -m venv venv
source venv/bin/activate
```
Instale as dependências do projeto:
```
pip install -r requirements.txt
pip install psycopg2 gunicorn
```

## 6. Configuração de Variáveis de Ambiente (local_settings.py)
Crie o arquivo local_settings.py dentro da pasta do projeto Django (onde está o settings.py):
```
nano caminho/do/seu_projeto/settings/local_settings.py
```
Conteúdo:
```
SECRET_KEY = 'chave_secreta_gerada'

DEBUG = False

ALLOWED_HOSTS = ['seu_dominio.com', 'seu.ip.publico']

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'nome_do_banco',
        'USER': 'nome_do_usuario',
        'PASSWORD': 'senha_forte',
        'HOST': 'localhost',
        'PORT': '5432',
    }
}
```
Para gerar uma chave secreta segura:
```
python -c "import string, secrets; print(''.join(secrets.choice(string.ascii_letters + string.digits + string.punctuation) for _ in range(64)))"
```
No arquivo settings.py, inclua no final:
```
try:
    from .local_settings import *
except ImportError:
    pass
```

## 7. Migração e Configuração do Django
```
python manage.py migrate
python manage.py collectstatic --noinput
python manage.py createsuperuser
```

## 8. Configuração do Gunicorn
Crie o socket:
```
sudo nano /etc/systemd/system/NOME_PARA_SEU_SOCKET.socket
```
Conteúdo:
```
[Unit]
Description=gunicorn blog socket

[Socket]
ListenStream=/run/NOME_PARA_SEU_SOCKET.socket

[Install]
WantedBy=sockets.target
```
Crie o serviço:
```
sudo nano /etc/systemd/system/NOME_PARA_SEU_SERVICO.service
```
Conteúdo:
```
[Unit]
Description=Gunicorn daemon
Requires=NOME_DO_SEU_SOCKET.socket
After=network.target

[Service]
User=SEU_USUARIO_DO_SERVIDOR
Group=www-data
Restart=on-failure
WorkingDirectory=/home/SUA_PASTA_RAIZ_DO_PROJETO
ExecStart=/home/SUA_PASTA_RAIZ_DO_PROJETO/venv/bin/gunicorn \
          --error-logfile /home/SUA_PASTA_RAIZ_DO_PROJETO/gunicorn-error-log \
          --enable-stdio-inheritance \
          --log-level "debug" \
          --capture-output \
          --access-logfile - \
          --workers 6 \
          --bind unix:/run/NOME_DO_SEU_SOCKET.socket \
          PASTA_COM_O_ARQUIVO_SETTINGS.wsgi:application

[Install]
WantedBy=multi-user.target
```
Ative o socket e o service:
```
sudo systemctl start NOME_DO_SEU_SOCKET.socket
sudo systemctl start NOME_DO_SEU_SERVICO.service

sudo systemctl enable NOME_DO_SEU_SOCKET.socket
sudo systemctl enable NOME_DO_SEU_SERVICO.service
```
Comandos úteis:
```
sudo systemctl status NOME_DO_SEU_SOCKET.socket
sudo systemctl status NOME_DO_SEU_SERVICO.service

sudo systemctl restart NOME_DO_SEU_SOCKET.socket
sudo systemctl restart NOME_DO_SEU_SERVICO.service

sudo journalctl -u NOME_DO_SEU_SOCKET.socket
sudo journalctl -u NOME_DO_SEU_SERVICO.service

-- Rode isso caso altere algo no socket ou service
sudo systemctl daemon-reload
```

## 9.1. Configuração do Nginx para IP
Atualize a timezone do servidor (Opcional):
```
sudo timedatectl set-timezone America/Sao_Paulo
```
Crie o arquvio de configuração:
```
sudo nano /etc/nginx/sites-available/seu_projeto
```
Conteúdo:
```
server {
  listen 80;
  listen [::]:80;
  server_name SEU_IP;

  # Add index.php to the list if you are using PHP
  index index.html index.htm index.nginx-debian.html index.php;
  
  # ATTENTION: pasta/para/os/arquivos/estáticos
  location /static {
    autoindex on;
    alias pasta/para/os/arquivos/estáticos;
  }

  # ATTENTION: pasta/para/os/arquivos/de/midia 
  location /media {
    autoindex on;
    alias pasta/para/os/arquivos/de/midia;
  }

  # ATTENTION: NOME_DO_SEU_SOCKET
  location / {
    proxy_pass http://unix:/run/NOME_DO_SEU_SOCKET.socket;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
    proxy_cache_bypass $http_upgrade;
  }

  location ~ /\.ht {
    deny all;
  }

  location ~ /\. {
    access_log off;
    log_not_found off;
    deny all;
  }

  gzip on;
  gzip_disable "msie6";

  gzip_comp_level 6;
  gzip_min_length 1100;
  gzip_buffers 4 32k;
  gzip_proxied any;
  gzip_types
    text/plain
    text/css
    text/js
    text/xml
    text/javascript
    application/javascript
    application/x-javascript
    application/json
    application/xml
    application/rss+xml
    image/svg+xml;

  access_log off;
  #access_log  /var/log/nginx/SEU_IP-access.log;
  error_log   /var/log/nginx/SEU_IP-error.log;
}
```
Adicione a configuração para os sites ativos:
```
sudo ln -s /etc/nginx/sites-available/seu_projeto /etc/nginx/sites-enabled/seu_projeto
```
Verifique se as configurações estão corretas e reinicie o nginx:
```
sudo nginx -t
sudo systemctl restart nginx
```

## 9.2. Configuração do Nginx para Domínio
