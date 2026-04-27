# NovoSGA para quem usa DNS na rede local.

Instalação do NovoSGA do zero com Docker Compose

Objetivo: 
Este passo a passo considera uma primeira instalação, com banco de dados vazio, usando:
- Docker Compose
- MySQL 5.7
- Mercure
- NovoSGA
- Painel
Triagem
- Caddy como proxy reverso com HTTPS interno
- DNS local (ex.: Pi-hole), que foi a forma mais estável no ambiente testado (não acompanha na instalação).
---
Cenário considerado
- Servidor Docker: `192.168.1.2`  
- DNS local: `192.168.1.11`  

Hosts usados:
- `sga.seudominio.local`
- `painel.seudominio.local`
- `triagem.seudominio.local`
---
1. Pré-requisitos.

Antes de começar, o servidor precisa ter:
- Docker instalado
- Docker Compose disponível no comando `docker compose`
- DNS local funcionando, apontando os nomes para o IP do servidor
- Portas 80 e 443 liberadas no servidor

Teste rápido.
```bash
docker --version
docker compose version
```
---
2. Criar a estrutura de pastas.
   
Crie a pasta principal do projeto e as pastas de persistência.
```bash
mkdir -p /root/novosga
mkdir -p /root/novosga/dados/mysql
mkdir -p /root/novosga/dados/imagens
```
Estrutura esperada.
```text
/root/novosga/
├── docker-compose.yml
├── Caddyfile
├── .env
└── dados/
    ├── mysql/
    └── imagens/
```
---
3. Limpar as pastas se for uma instalação nova.

Se a ideia for começar do zero, deixe a pasta do banco completamente vazia.
```bash
find /root/novosga/dados/mysql -mindepth 1 -delete
find /root/novosga/dados/imagens -mindepth 1 -delete
```
Confirmar se a pasta do banco está vazia.
```bash
ls -la /root/novosga/dados/mysql
```
O resultado ideal deve mostrar apenas:
```text
.
..
```
Se existir qualquer arquivo ali, o MySQL pode falhar na inicialização.
---
4. Salvar os arquivos da instalação
Dentro de `/root/novosga`, salve os arquivos:
`docker-compose.yml`
`Caddyfile`
`.env`
---
5. Exemplo do arquivo `.env`.
   
Crie o arquivo:
```bash
nano /root/novosga/.env
```
Conteúdo sugerido:
```env
TZ=America/Sao_Paulo
APP_LANGUAGE=pt_BR

SERVER_IP=192.168.1.2
PRIMARY_URL=https://sga.seudominio.local

MYSQL_DATABASE=novosga2
MYSQL_USER=novosga
MYSQL_PASSWORD=MySQL_App_P4ssW0rd
MYSQL_ROOT_PASSWORD=MySQL_r00t_P4ssW0rd

MERCURE_JWT_SECRET=ChangeThisMercureHubJWTSecretKey

NOVOSGA_ADMIN_USERNAME=admin
NOVOSGA_ADMIN_PASSWORD=123456
NOVOSGA_ADMIN_FIRSTNAME=Administrador
NOVOSGA_ADMIN_LASTNAME=Global

NOVOSGA_UNITY_NAME=Minha unidade
NOVOSGA_UNITY_CODE=U01

NOVOSGA_NOPRIORITY_NAME=Normal
NOVOSGA_NOPRIORITY_DESCRIPTION=Atendimento normal
NOVOSGA_PRIORITY_NAME=Prioridade
NOVOSGA_PRIORITY_DESCRIPTION=Atendimento prioritário

NOVOSGA_PLACE_NAME=Mesa
```
---
6. Configurar o DNS local
No Pi-hole ou DNS local, crie os registros:
`sga.seudominio.local` -> `192.168.1.2`
`painel.seudominio.local` -> `192.168.1.2`
`triagem.seudominio.local` -> `192.168.1.2`
Testar resolução no servidor
```bash
nslookup sga.seudominio.local 192.168.1.11
nslookup painel.seudominio.local 192.168.1.11
nslookup triagem.seudominio.local 192.168.1.11
```
Todas devem responder com `192.168.1.2`.
---
7. Validar o compose antes de subir
Entre na pasta do projeto:
```bash
cd /root/novosga
```
Valide o arquivo:
```bash
docker compose config
```
Se esse comando não retornar erro, o compose está sintaticamente válido.
---
8. Subir primeiro o banco
Suba apenas o MySQL primeiro:
```bash
docker compose up -d mysqldb
```
Acompanhe os logs:
```bash
docker compose logs -f mysqldb
```
Espere aparecer algo como:
```text
mysqld: ready for connections
```
---
9. Subir o restante da stack
Depois que o MySQL estiver pronto:
```bash
docker compose up -d
```
Acompanhe os logs do proxy:
```bash
docker compose logs -f proxy
```
O Caddy deve gerar os certificados locais para:

- `sga.seu_dominio_interno.local`
- `painel.seu_dominio_interno.local`
- `triagem.seu_dominio_interno.local`
---
10. Fazer a instalação inicial do NovoSGA.
    
Como o banco é novo, somente subir os containers não basta.  

É obrigatório executar o instalador do NovoSGA para criar a estrutura do banco.
Rode:
```bash
docker exec -it novosga-novosga-1 sh -lc 'cd /var/www/html && php bin/console novosga:install'
```
Esse comando cria as tabelas iniciais, incluindo a tabela `metadata`, que é necessária para a tela de login funcionar.
Se quiser mais detalhes na execução.
```bash
docker exec -it novosga-novosga-1 sh -lc 'cd /var/www/html && php bin/console novosga:install -vvv'
```
---
11. Limpar o cache após a instalação.
Depois do instalador:
```bash
docker exec -it novosga-novosga-1 sh -lc 'cd /var/www/html && php bin/console cache:clear --env=prod'
```
---
12. Validar se as tabelas foram criadas.
Confira no banco:
```bash
docker exec -it novosga-mysqldb-1 mysql -unovosga -pMySQL_App_P4ssW0rd -D novosga2 -e "show tables;"
```
Se aparecerem várias tabelas, a instalação do banco foi concluída.
---
13. Testar os acessos.
    
Teste pelo terminal:
```bash
curl -vk --resolve sga.seu_dominio_interno.local:443:192.168.1.14 https://sga.seu_dominio_interno.local
curl -vk --resolve painel.seu_dominio_interno.local:443:192.168.1.14 https://painel.seu_dominio_interno.local
curl -vk --resolve triagem.seu_dominio_interno.local:443:192.168.1.14 https://triagem.seu_dominio_interno.local
```
No navegador.
Acesse:
- `https://sga.seu_dominio_interno.local`
- `https://painel.seu_dominio_interno.local`
- `https://triagem.seu_dominio_interno.local`
---
14. Confiar no certificado local.
    
Como o Caddy usa `tls internal`, o navegador vai indicar que o certificado não é confiável até a CA ser importada.

Baixe o certificado raiz por:
```text
https://sga.seu_dominio_interno.local/caddy-root.crt
```
No Firefox
Configurações
Privacidade e Segurança
Certificados
Ver certificados
Autoridades
Importar
Selecione `caddy-root.crt`
Marque para confiar nesta CA para identificar sites
---
15. Comandos úteis
Ver containers
```bash
docker compose ps
```
Reiniciar tudo
```bash
docker compose restart
```
Parar tudo
```bash
docker compose down
```
Ver logs do SGA
```bash
docker compose logs -f novosga
```
Ver logs do proxy
```bash
docker compose logs -f proxy
```
Ver logs do banco
```bash
docker compose logs -f mysqldb
```
---
16. Erros comuns e causas
Erro: `Table 'novosga2.metadata' doesn't exist`
Causa: banco novo sem instalação do NovoSGA.  
Solução:
```bash
docker exec -it novosga-novosga-1 sh -lc 'cd /var/www/html && php bin/console novosga:install'
```
Erro: `--initialize specified but the data directory has files in it`
Causa: a pasta do MySQL não está vazia.  
Solução:
```bash
docker compose down
find /root/novosga/dados/mysql -mindepth 1 -delete
docker compose up -d mysqldb
```
Erro 500 no login logo após subir tudo
Causa mais comum: banco não foi inicializado pelo instalador do NovoSGA.
---
17. Ordem correta da primeira instalação
Resumo do processo completo:
```bash
mkdir -p /root/novosga/dados/mysql
mkdir -p /root/novosga/dados/imagens

find /root/novosga/dados/mysql -mindepth 1 -delete
find /root/novosga/dados/imagens -mindepth 1 -delete

cd /root/novosga
docker compose config
docker compose up -d mysqldb
docker compose logs -f mysqldb
docker compose up -d

docker exec -it novosga-novosga-1 sh -lc 'cd /var/www/html && php bin/console novosga:install'
docker exec -it novosga-novosga-1 sh -lc 'cd /var/www/html && php bin/console cache:clear --env=prod'
docker exec -it novosga-mysqldb-1 mysql -unovosga -pMySQL_App_P4ssW0rd -D novosga2 -e "show tables;"
```
---
18. Conclusão
Para uma instalação nova, o ponto mais importante é:
subir os containers não basta.  
Depois de subir, é necessário executar:
```bash
php bin/console novosga:install
```
Sem isso, o banco fica vazio e o sistema retorna erro 500 no login.
