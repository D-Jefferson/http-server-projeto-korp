# http-server-projeto-korp

Solução para o desafio técnico Korp, desenvolvida com foco em boas práticas de DevOps e SRE: infraestrutura automatizada, observável e reproduzível do zero com um único comando.

## Arquitetura

```
                          ┌─────────────────────────────────────────────┐
                          │               korp-network (bridge)          │
                          │                                               │
  Requisição HTTP ──────► │  NGINX :80  ──► App Go :8080                │
  curl localhost/...      │                    │                          │
                          │                    └──► /metrics             │
                          │                              │                │
                          │  Prometheus :9090 ◄──────────┘               │
                          │       │                                       │
                          │  Grafana :3000 ◄──────────────────────────── │
                          └─────────────────────────────────────────────┘
```

| Componente | Tecnologia | Função |
|---|---|---|
| Serviço HTTP | Go 1.21 | API REST + exposição de métricas Prometheus |
| Proxy Reverso | NGINX 1.27 | Ponto de entrada público, roteamento |
| Métricas | Prometheus v2.53 | Coleta e armazenamento de métricas |
| Dashboards | Grafana 11.2 | Visualização (provisionado "as code") |
| Automação | Ansible | Provisionamento completo do ambiente |

## Pré-requisitos

- Docker e Docker Compose (plugin v2)
- Para automação via Ansible: Linux/WSL com Python 3 e `pip install ansible`

## Execução

### Opção 1 — Ansible (ambiente Linux/WSL)

Provisiona tudo com um único comando, incluindo instalação do Docker se necessário:

```bash
# Instalar dependências (apenas na primeira vez)
ansible-galaxy collection install -r ansible/requirements.yml

ansible-playbook -i ansible/inventory.ini ansible/playbook.yml -K
```

### Opção 2 — Docker Compose direto

```bash
docker compose up --build -d
```

## Validação

```bash
# Endpoint principal
curl http://localhost/projeto-korp

# Health check
curl http://localhost/healthz

# Métricas Prometheus
curl http://localhost/metrics
```

**Grafana**: acesse `http://localhost:3000` — usuário `admin`, senha `admin`.  
O dashboard **"App Dashboard"** já estará provisionado automaticamente.

## Endpoints da Aplicação

| Endpoint | Método | Descrição |
|---|---|---|
| `/projeto-korp` | GET | Retorna JSON com nome e horário UTC |
| `/healthz` | GET | Health check — retorna `{"status":"ok"}` |
| `/metrics` | GET | Métricas no formato Prometheus |

## Estrutura do Projeto

```
.
├── app/
│   ├── main.go           # Servidor HTTP com métricas e health check
│   ├── go.mod
│   └── Dockerfile        # Multi-stage build (builder → alpine)
├── config/
│   ├── nginx/conf.d/     # Configuração do proxy reverso
│   ├── prometheus/       # Configuração do scrape
│   └── grafana/
│       └── provisioning/ # Datasource e dashboard provisionados as-code
├── ansible/
│   ├── playbook.yml      # Playbook principal
│   ├── inventory.ini
│   └── roles/
│       ├── docker/       # Instalação do Docker Engine + Compose plugin
│       ├── deploy/       # Build e orquestração via docker compose
│       └── test/         # Validação via requisição HTTP com assert
└── docker-compose.yml
```

## Decisões Arquiteturais

**Multi-stage Dockerfile**: O binário final roda em `alpine:3.19` sem ferramentas de compilação, reduzindo a superfície de ataque e o tamanho da imagem.

**Non-root container**: O processo roda com usuário `appuser` (UID 1000), eliminando privilégios desnecessários.

**HTTP Server com timeouts**: `ReadTimeout`, `WriteTimeout` e `IdleTimeout` configurados para mitigar ataques Slowloris.

**Métricas com labels `method` + `status_code`**: Permite distinguir erros 4xx de 2xx no Grafana, seguindo o padrão RED (Rate, Errors, Duration).

**Grafana "as code"**: Datasources e dashboards provisionados via arquivos YAML/JSON. Nenhuma configuração manual necessária após `docker compose up`.

**Healthcheck + `depends_on: condition: service_healthy`**: Elimina race conditions na inicialização. O NGINX só sobe após o app Go estar saudável.

**Versões de imagens fixadas**: `grafana:11.2.0`, `prometheus:v2.53.0`, `nginx:1.27-alpine` — garantia de reproducibilidade.

**Ansible Roles modulares**: Divisão em `docker`, `deploy` e `test` permite reaproveitamento independente. A role `test` usa `assert` com `fail_msg` e `success_msg` para validação com saída legível.

**Docker Compose v2 no Ansible**: Uso do módulo `community.docker.docker_compose_v2` (não o deprecated v1 via pip).
