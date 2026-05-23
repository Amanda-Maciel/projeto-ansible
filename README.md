# projeto-ansible
Projeto da matéria de DevOps, integrado a matéria de microserviços. Esse projeto contempla documentação de microserviços + o projeto.

# 📅 Microsserviço de Agenda — Cadastro de Usuários e Eventos

> Serviço web responsável pelo cadastro de usuários e gerenciamento de eventos, provisionado automaticamente via Ansible sobre infraestrutura Vagrant/VirtualBox.

---

## 1. Descrição Funcional

**Nome:** `agenda-service`

**Objetivo:** Disponibilizar uma aplicação web PHP que permite o cadastro de usuários e eventos em uma agenda compartilhada, com persistência em banco de dados MySQL. A infraestrutura do serviço é provisionada e configurada de forma automatizada utilizando **Ansible** e **Vagrant**.

**Responsabilidades principais:**
- Cadastrar novos usuários (nome, e-mail e senha)
- Cadastrar eventos vinculados a um usuário (título, descrição e data)
- Listar todos os eventos com o nome do usuário responsável
- Provisionar automaticamente o servidor web (Apache + PHP + MySQL) via Ansible

---

## 2. Endpoints da API

| Método | URL | Descrição |
|--------|-----|-----------|
| `GET`  | `/` | Página inicial com navegação para as funcionalidades |
| `GET / POST` | `/cadastro_usuario.php` | Exibe o formulário e processa o cadastro de usuário |
| `GET / POST` | `/cadastro_evento.php` | Exibe o formulário e processa o cadastro de evento |
| `GET`  | `/lista_eventos.php` | Lista todos os eventos com o nome do usuário associado |
| `GET`  | `/health` | Verifica se o Apache está respondendo (health check) |

> A aplicação segue o padrão de páginas PHP que renderizam formulário (GET) e processam a submissão (POST) no mesmo endpoint.

---

## 3. Exemplo de Requisição e Resposta

### `POST /cadastro_usuario.php`

**Corpo do formulário (form-data):**
```
nome=Maria Silva
email=maria@email.com
senha=senha123
```

**Resposta de sucesso:**
```
Usuário cadastrado com sucesso!
```

**Resposta de erro (ex: e-mail duplicado):**
```
Erro: Duplicate entry 'maria@email.com' for key 'email'
```

---

### `GET /lista_eventos.php`

**Resposta (tabela HTML renderizada a partir da query SQL):**

| ID | Título | Descrição | Data | Usuário |
|----|--------|-----------|------|---------|
| 1 | Reunião de equipe | Alinhamento semanal | 2026-05-28 | Maria Silva |
| 2 | Entrega do projeto | DevOps — módulo Ansible | 2026-05-30 | João Costa |

> A listagem é ordenada por data crescente (`ORDER BY e.data ASC`), com JOIN entre as tabelas `eventos` e `usuarios`.

---

## 4. Dependências Externas

| Tipo | Nome / Tecnologia | Descrição |
|------|-------------------|-----------|
| Servidor web | Apache 2 | Serve os arquivos PHP da aplicação |
| Linguagem | PHP | Processamento server-side dos formulários |
| Banco de dados | MySQL | Persiste usuários e eventos; usuário `agenda`, banco `agenda` |
| Provisionamento | Ansible | Automatiza instalação e configuração do ambiente |
| Infraestrutura local | Vagrant + VirtualBox | Cria as VMs `ansible` (10.0.10.1) e `web` (10.0.10.2) |
| Broker / Fila | *(não utilizado nesta versão)* | — |
| APIs externas | *(não utilizado nesta versão)* | — |

---

## 5. Responsável pelo Serviço

| Campo | Informação |
|-------|------------|
| **Desenvolvedor(a)** | Adriano Ferruzzi - Aluna : Amanda Maciel |
| **Disciplina** | DevOps — Turma 2026/1 |
| **Repositório** | https://github.com/Amanda-Maciel/projeto-ansible |

---

## 6. Procedimentos Básicos de Operação

### Pré-requisitos

- [Vagrant](https://www.vagrantup.com/) instalado
- [VirtualBox](https://www.virtualbox.org/) instalado
- Rede com suporte a `public_network` (bridge)

### Estrutura das VMs

| VM | Hostname | IP Privado | Função |
|----|----------|------------|--------|
| `ansible` | ansible | 10.0.10.1 | Controlador Ansible |
| `web` | web | 10.0.10.2 | Servidor Apache + PHP + MySQL |

### Executar localmente

```bash
# 1. Clone o repositório
git clone https://github.com/seu-usuario/agenda-service.git
cd agenda-service

# 2. Suba as duas VMs
vagrant up

# As VMs são provisionadas automaticamente:
# - A VM "ansible" instala o Ansible e configura acesso SSH sem senha
# - A VM "web" recebe os arquivos da aplicação via playbook Ansible
```

```bash
# 3. Acesse a VM ansible
vagrant ssh ansible

# 4. Dentro da VM ansible, execute o playbook
cd /home/vagrant/ansible
ansible-playbook playbook_atividade.yml
```

A aplicação estará disponível no IP da VM `web` na porta 80.

### Verificar logs

```bash
# Logs do Apache na VM web
vagrant ssh web -- sudo tail -f /var/log/apache2/error.log

# Logs de acesso
vagrant ssh web -- sudo tail -f /var/log/apache2/access.log
```

### Health check

```bash
# Verificar se o Apache está respondendo na VM web
curl http://10.0.10.2/

# Ou pelo comando Ansible diretamente
ansible web -m uri -a "url=http://localhost/ status_code=200"
```

### Reiniciar o serviço

```bash
# Reiniciar Apache na VM web via Ansible
ansible web -m systemd -a "name=apache2 state=restarted" --become

# Ou acessando a VM diretamente
vagrant ssh web -- sudo systemctl restart apache2
```

### Destruir e recriar o ambiente

```bash
vagrant destroy -f
vagrant up
```

---

## 7. Regras de Negócio

| # | Entidade | Regra |
|---|----------|-------|
| 1 | Usuário | `nome`, `email` e `senha` são obrigatórios |
| 2 | Usuário | O campo `email` deve ser único no banco (constraint `UNIQUE`) |
| 3 | Usuário | A senha é armazenada com hash via `password_hash()` (bcrypt) |
| 4 | Evento | `titulo`, `data` e `usuario_id` são obrigatórios |
| 5 | Evento | O `usuario_id` deve referenciar um usuário existente (foreign key) |
| 6 | Listagem | Eventos são exibidos em ordem cronológica crescente por data |
| 7 | Banco | O banco `agenda` e o usuário MySQL `agenda` são criados pelo script `database.sql` durante o provisionamento |

---

## 8. Eventos Publicados ou Consumidos

Esta versão do serviço **não utiliza broker de mensagens ou eventos assíncronos**. Toda a comunicação é síncrona via requisições HTTP (formulários HTML → PHP → MySQL).

> Em uma evolução futura, o cadastro de usuário poderia publicar um evento `usuario.cadastrado` em uma fila (ex: RabbitMQ) para disparar um e-mail de boas-vindas via serviço de notificação.

---

## 9. Métricas Monitoradas

| Métrica | Descrição | Como verificar |
|---------|-----------|----------------|
| Status do Apache | Serviço ativo ou inativo | `systemctl status apache2` |
| Status do MySQL | Serviço ativo ou inativo | `systemctl status mysql` |
| Total de usuários cadastrados | Quantidade de registros na tabela `usuarios` | `SELECT COUNT(*) FROM usuarios;` |
| Total de eventos cadastrados | Quantidade de registros na tabela `eventos` | `SELECT COUNT(*) FROM eventos;` |
| Logs de erro HTTP | Requisições com erro no Apache | `/var/log/apache2/error.log` |
| Uso de memória das VMs | Cada VM usa 1048 MB de RAM (configurado no Vagrantfile) | `free -h` dentro da VM |

---

## 10. ADR Relacionado — Decisão Arquitetural

### ADR-001 — Provisionamento automatizado com Ansible em vez de configuração manual

**Contexto:** A aplicação precisa ser instalada em um servidor Ubuntu com Apache, PHP e MySQL. Isso poderia ser feito manualmente via `ssh` + comandos avulsos.

**Decisão:** Utilizar **Ansible** para automatizar todo o provisionamento: instalação de pacotes, cópia de arquivos, importação do banco de dados e inicialização dos serviços.

**Justificativa:** A automação via Ansible garante que o ambiente seja **reproduzível e consistente** — qualquer membro da equipe pode recriar o ambiente idêntico executando `vagrant up` + `ansible-playbook`. Elimina o risco de configuração manual inconsistente entre ambientes.

**Consequências:** Necessidade de manter os playbooks atualizados. Em troca, o onboarding de novos desenvolvedores é drasticamente simplificado.

---

## Estrutura do Repositório

```
02_ansible/
├── Vagrantfile                   # Define as VMs ansible e web
├── access.sh                     # Configura acesso SSH sem senha entre as VMs
├── exemplos_comandos.sh          # Referência de comandos Ansible úteis
├── agenda/
│   ├── README.md
│   ├── app/
│   │   ├── index.html            # Página inicial da aplicação
│   │   ├── cadastro_usuario.php  # Cadastro de usuários
│   │   ├── cadastro_evento.php   # Cadastro de eventos
│   │   └── lista_eventos.php     # Listagem de eventos
│   └── db/
│       └── database.sql          # Script de criação do banco e tabelas
└── ansible/
    ├── ansible.cfg               # Configurações do Ansible
    ├── hosts.yml                 # Inventário de hosts
    ├── playbook_atividade.yml    # Playbook principal de provisionamento
    └── playbook_sample.yml       # Exemplos de tasks Ansible
```

---

*Documentação elaborada conforme os requisitos da disciplina de DevOps — 2026/1.*
