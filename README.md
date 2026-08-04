# Weconecta-data-model
Schema PostgreSQL para formulários dinâmicos com ramificação condicional, árvore de nós reordenáveis e soft delete para preservar respostas durante edição estrutural. Projeto AGES II/PUCRS.

# Banco de Dados WeConecta – PostgreSQL

O projeto **WeConecta** utiliza o **PostgreSQL** como sistema gerenciador de banco de dados.  
A escolha se deve à sua robustez, forte consistência transacional (ACID), suporte nativo a JSON, integrações seguras e bom desempenho em modelagens relacionais com uso de JSON.



## Modelo Conceitual

Criado utilizando o software **brModelo**:
<img width="2342" height="1096" alt="conceitual" src="https://github.com/user-attachments/assets/543a63a0-6419-42bf-9d74-7ca613c4d654" />



## Modelo Lógico

Modelado utilizando o **dbdiagram.io**:
<img width="2102" height="1534" alt="logico" src="https://github.com/user-attachments/assets/e470090d-00c6-4a0a-992e-6e655a8120f7" />



# Estrutura das Tabelas

A seguir estão as tabelas e descrições detalhadas do banco de dados.



# ➤ Tabela: `users`

Armazena todos os usuários do sistema.

| Coluna        | Tipo             | Restrições                     | Descrição |
|---------------|------------------|--------------------------------|-----------|
| id            | `uuid`           | PK                              | Identificador único. |
| email         | `varchar(255)`   | UNIQUE, NOT NULL               | E-mail de acesso. |
| password      | `varchar(255)`   | NOT NULL                       | Hash da senha. |
| phone         | `varchar(11)`    | UNIQUE, NULLABLE               | Telefone do usuário. |
| friendly_name | `varchar(255)`   | NULLABLE                       | Nome de exibição. |
| role          | `enum(UserRole)` | NOT NULL                       | Permissão (`ADMIN` ou `MEMBER`). |
| blocked_at    | `timestamp`      | NULLABLE                       | Data/hora de bloqueio. |
| first_login   | `boolean`        | NOT NULL, DEFAULT: true        | Indica se é o primeiro acesso. |
| created_at    | `timestamp`      | NOT NULL, DEFAULT: CURRENT_TIMESTAMP | Data de criação. |

**Relacionamentos:**

- 1:N com `surveys` (usuário criador).
- 1:N com `survey_answers` (como client_id).
- N:N com `surveys` via `users_surveys`.

---

# ➤ Tabela: `surveys`

Representa um questionário cadastrado no sistema.

| Coluna              | Tipo           | Restrições                             | Descrição |
|---------------------|----------------|-----------------------------------------|-----------|
| id                  | `integer`      | PK                                      | Identificador único do questionário. |
| first_survey_element| `integer`      | FK → survey_elements, NULLABLE          | Primeiro elemento do fluxo. |
| title               | `varchar(255)` | NOT NULL                                | Título. |
| description         | `varchar(500)` | NULLABLE                                | Descrição. |
| url                 | `varchar(255)` | NULLABLE                                | URL pública. |
| flow                | `jsonb`        | NULLABLE                                | Estrutura de fluxo. |
| is_published        | `boolean`      | NOT NULL, DEFAULT: false                | Status de publicação. |
| user_id             | `uuid`         | FK → users, NOT NULL                    | Criador (owner). |
| created_at          | `timestamp`    | NOT NULL, DEFAULT: CURRENT_TIMESTAMP    | Criação. |
| updated_at          | `timestamp`    | NOT NULL, DEFAULT: CURRENT_TIMESTAMP    | Última alteração. |

**Relacionamentos:**

- N:1 com `users` (owner).
- 1:N com `survey_answers`.
- 1:N com `users_surveys`.
- N:1 com `survey_elements` via `first_survey_element`.

---

# ➤ Tabela: `users_surveys`

Tabela que representa o relacionamento N:N entre usuários e questionários (usuários colaboradores).

| Coluna    | Tipo      | Restrições                      | Descrição |
|-----------|-----------|----------------------------------|-----------|
| id        | `integer` | PK                               | Identificador único. |
| user_id   | `uuid`    | FK → users, NOT NULL             | Usuário colaborador. |
| survey_id | `integer` | FK → surveys, NOT NULL           | Questionário associado. |

**Relacionamentos:**

- N:1 com `users`.
- N:1 com `surveys`.

---

# ➤ Tabela: `survey_elements`

Representa um elemento do questionário (pergunta, mensagem, input, múltipla escolha).

| Coluna      | Tipo                    | Restrições     | Descrição |
|-------------|--------------------------|----------------|-----------|
| id          | `integer`               | PK             | Identificador do elemento. |
| description | `varchar(500)`          | NOT NULL       | Enunciado/texto do elemento. |
| type        | `enum(SurveyElementType)` | NOT NULL     | Tipo do elemento. |
| deleted_at  | `timestamp`             | NULLABLE       | Soft delete. |

**Relacionamentos:**

- 1:N com `options`.
- 1:N com `survey_answers`.
- 1:N com `surveys` como elemento inicial.

---

# ➤ Tabela: `options`

Opções associadas a um elemento do questionário.

| Coluna            | Tipo           | Restrições                        | Descrição |
|-------------------|----------------|------------------------------------|-----------|
| id                | `integer`      | PK                                 | Identificador da opção. |
| survey_element_id | `integer`      | FK → survey_elements, NOT NULL     | Elemento ao qual pertence. |
| description       | `varchar(255)` | NOT NULL                           | Texto da opção. |
| deleted_at        | `timestamp`    | NULLABLE                           | Soft delete. |

**Relacionamentos:**

- N:1 com `survey_elements`.
- 1:N com `survey_answers`.

---

# ➤ Tabela: `survey_answers`

Armazena as respostas fornecidas pelos participantes.  
Cada registro representa **uma opção marcada** ou **uma resposta de input**.

### Chave Primária Composta:
- `identifier`
- `client_id`
- `survey_id`
- `survey_element_id`
- `option_id`

| Coluna            | Tipo        | Restrições                             | Descrição |
|-------------------|-------------|-----------------------------------------|-----------|
| identifier        | `varchar`   | PK                                      | Identificador do respondente. |
| client_id         | `uuid`      | PK, FK → users                          | Usuário membro que gerou o link. |
| survey_id         | `integer`   | PK, FK → surveys                        | Survey respondido. |
| survey_element_id | `integer`   | PK, FK → survey_elements                | Elemento respondido. |
| option_id         | `integer`   | PK, FK → options                        | Opção marcada. |
| input_response    | `varchar`   | NULLABLE                                | Resposta textual (para perguntas de input). |
| created_at        | `timestamp` | NOT NULL, DEFAULT: CURRENT_TIMESTAMP    | Criação. |
| updated_at        | `timestamp` | NOT NULL, DEFAULT: CURRENT_TIMESTAMP    | Última alteração. |

**Observações importantes:**

- Perguntas **MULTIPLE_CHOICE** geram várias linhas com o mesmo `identifier + client_id + survey_id + survey_element_id`, variando apenas o `option_id`.
- `client_id` garante segregação das respostas por usuário que distribuiu o link.
