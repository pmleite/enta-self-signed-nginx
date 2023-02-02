# Self Sign Local Host - howto


A) Criar as seguintes pastas no diretório do projeto:

 - cfg (onde vai ser mapeado o volume com os ficheiros de configuração do nginx)

 - src (Onde vai ser mapeado o volume com os ficheiros da página que iremos criar)

 - crt (Onde vai ser mapeado o volume com os certificados de segurança do portal)

 - ssl (Onde vai ser mapeado o volume com a chave da plataforma)

B) Dentro da pasta src criar um ficheiro index.html com uma página básica qualquer (exemplo)

```
<!DOCTYPE html>
<html>
<head>
    <meta charset='utf-8'>
    <meta http-equiv='X-UA-Compatible' content='IE=edge'>
    <title>Page Title</title>
    <meta name='viewport' content='width=device-width, initial-scale=1'>
    <link rel='stylesheet' type='text/css' media='screen' href='main.css'>
    <script src='main.js'></script>
</head>
<body>
    <p>teste</p>
</body>
</html>
```


C) Criar um docker-compose.yml de acordo como o exemplo abaixo

```
version: '3'

services:
  web:
    image: nginx
    volumes:
      - ./cfg:/etc/nginx/
      - ./src:/var/www/localhost/htdocs
      - ./crt:/etc/ssl/certs
      - ./ssl:/etc/ssl/private
    ports:
      - 8000:80
      - 443:443
```

D) dar uma entrada no ficheiro hosts (/etc/hosts) com um nome para o nosso dominio ficticio. Para o exemplo criou-se o domínio local.web.com com nome alternativo ao localhost

```
127.0.0.1       localhost
127.0.0.1       local.web.com
127.0.1.1       paulo-leite

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

E) depois de arrancar com o nginx editar o ficheiro "default.conf" para que fique de acordo como exemplo abaixo:

```
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    # New root location
    location / {
            root /var/www/localhost/htdocs; 
            # return 404;
    }
    # You may need this to prevent return 404 recursion.
    location = /404.html {
            internal;
    }
}
```

F) Verificar se as configuraçẽos estão conrretas com o recurso ao comando:

```
nginx -t
```

que deverá devolver qualquer coisa semelhante a:

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Estando tudo correto faz-se um reload das configurações como comando:

```
nginx -s reload
```

Neste momento tem o servidor nginx a trabalhar com http apenas.


G) entrar no container do servidor e executar o seguinte comando para gerar os certificados:

```
openssl req -x509 -nodes -days 365 -subj "/C=CA/ST=QC/O=Company, Inc./CN=local.web.com" -addext "subjectAltName=DNS:local.web.com" -newkey rsa:2048 -keyout /etc/ssl/private/nginx-selfsigned.key -out /etc/ssl/certs/nginx-selfsigned.crt;
```

Nota: o dominio que escolhemos no passo C) deverá estar refletido nesta linha de comando.

H) Edita-se novamente o ficheiro default.conf para que o servidor fique à escuta da porta 443 (ssL) e aponte para os respetivos certificados.

```
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    listen 443 ssl http2 default_server;
    listen [::]:443 ssl http2 default_server;
    ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
    ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;
    # New root location
    location / {
            root /var/www/localhost/htdocs; 
            # return 404;
    }
    # You may need this to prevent return 404 recursion.
    location = /404.html {
            internal;
    }
}
```

I) Verificar se as configuraçẽos estão conrretas com o recurso ao comando:

```
nginx -t
```
Estando tudo correto faz-se um reload das configurações como comando:
```
nginx -s reload
```

J) Test-se localmente (no container) se o servidor está a responder a HTTPS como comando:

```
curl https://localhost --insecure;
```
o que deverá dar como resposta o código html da página que criamos no passo B)

K) Se navegar pelo browser (no host local) pelo endereço https://local.web.com (o dominio que escolhemos), vai dar um aviso de site inseguro, isto porque o browser não pdoe verificar junto do CA se realmente o portal é quem diz ser. Para se ultrapassar isto devemos ir ás definições de segurança do browser -> certidicados e importar o certificado que está na pasta ssl na nossa pasta de projeto (ficheiro nginx-selfsigned.key)

Posteriormente fechamos o browser, voltamos a abrir e ao navegar para o nosso endereço vemos que já é considerado um site seguro, ou seja passamos a utilizar HTTP em cima de SSL na sua porta standard 443


