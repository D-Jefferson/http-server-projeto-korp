# Projeto Korp - Desafio Técnico (DevOps & SRE)

Este repositório contém a solução para o desafio técnico "Projeto Korp", desenvolvido com foco em boas práticas de engenharia de software, cultura DevOps e SRE (Site Reliability Engineering). 

A infraestrutura foi pensada para ser automatizada, escalável, monitorável e fácil de manter, atendendo aos requisitos de nível Sênior.

## 🚀 Arquitetura da Solução

A solução foi construída utilizando os seguintes componentes:

1. **Golang HTTP Server (`/app`)**: Um microserviço rápido, leve e tipado, servindo o endpoint `/projeto-korp` e expondo métricas nativamente no `/metrics`.
2. **Docker (Multi-stage build)**: O processo de build da imagem Go garante um artefato final mínimo (baseado em `alpine`) e seguro (rodando com usuário restrito `non-root`), não levando ferramentas de compilação para produção.
3. **Docker Compose**: Orquestração do ambiente local definindo a rede privada `korp-network`.
4. **NGINX (Proxy Reverso)**: Ponto de entrada (Porta 80) que roteia as requisições para a aplicação Go (Porta 8080), mantendo a aplicação isolada da rede externa.
5. **Monitoramento (Prometheus + Grafana)**: 
   - O Prometheus coleta as métricas do serviço Go.
   - O Grafana foi configurado **"as code"** (Provisioning). Os Datasources e o Dashboard principal são injetados na inicialização do container, sem necessidade de configuração manual pela interface.
6. **Ansible (`/ansible`)**: A automação foi dividida em **Roles** (`docker`, `deploy`, `test`) garantindo modularidade. O playbook principal instala o Docker (caso necessário num ambiente Debian/Ubuntu), sobe a stack inteira e realiza um teste de integração batendo na API.

---

## 🛠️ Tecnologias Utilizadas

- **Linguagem**: Go (Golang) 1.21
- **Containers**: Docker & Docker Compose
- **Web Server / Proxy**: NGINX
- **Observabilidade**: Prometheus & Grafana
- **Automação (IaC)**: Ansible

---

## ⚙️ Como Executar

O projeto foi projetado para ser executado de forma totalmente automatizada através do Ansible. 

### Pré-requisitos
- Um ambiente Linux (ou WSL no Windows)
- Ansible instalado (`pip install ansible` ou `apt install ansible`)

### Passo a Passo

1. Clone o repositório:
   ```bash
   git clone <url-do-repositorio>
   cd http-server-projeto-korp
   ```

2. Execute o playbook de automação:
   O comando abaixo irá instalar as dependências (se necessário), orquestrar os containers e validar o funcionamento executando um request HTTP.
   ```bash
   ansible-playbook -i ansible/inventory.ini ansible/playbook.yml -K
   ```
   *(O parâmetro `-K` solicitará a senha de sudo, caso a role de docker precise instalar pacotes)*

3. Validação Manual (Opcional):
   Após a execução bem-sucedida, você pode validar localmente:
   - **API**: `curl http://localhost:80/projeto-korp`
   - **Grafana**: Acesse `http://localhost:3000` no seu navegador para ver o dashboard já configurado com as métricas de tempo de resposta e volume.

---

## 🧠 Decisões Arquiteturais e Boas Práticas

- **Go Modules sem `go.sum` inicial**: No ambiente de teste, assumimos que o host pode não ter Go instalado. O `go mod tidy` é executado na etapa de builder do Docker para garantir as dependências.
- **Segurança em Containers**: O container da aplicação roda com `USER appuser` (non-root), reduzindo drasticamente a superfície de ataque caso haja alguma vulnerabilidade na aplicação.
- **Monitoramento Embutido (Golden Signals)**: A própria aplicação expõe as métricas de Volume (Taxa de Requisições) e Latência (Duração Média) utilizando a biblioteca oficial do Prometheus.
- **Ansible Roles**: Em vez de um único script massivo de playbook, a divisão em roles permite o reaproveitamento de código em cenários do mundo real (ex: usar a role de Docker em outros projetos).
- **Grafana "As Code"**: Provisionar dashboards evita o clássico problema de "funciona na minha máquina, mas perdeu a configuração quando recriou o container".

---
*Desenvolvido como resolução do desafio técnico.*
