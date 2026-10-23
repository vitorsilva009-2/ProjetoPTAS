# Sistema de Restaurante — API REST

**PTAS3 · Projeto parcial** — CRUD REST em camadas com validação.

# Vitor Hugo da Silva e César Augusto Maia dos Santos

API em **Node.js + Express** para gerenciar o cardápio de um restaurante, com CRUD de duas entidades relacionadas (**Categoria** e **Prato**), organizada em camadas `routes → controllers → services → models`, com persistência inicial em **arquivo JSON** e validação com **zod**.

## Resumo do Problema

Muitos restaurantes de pequeno e médio porte ainda controlam o cardápio em papel ou planilhas soltas, o que gera preços desatualizados, pratos listados mesmo quando estão em falta e retrabalho a cada alteração. Esta API centraliza essas informações: cadastra, consulta, atualiza e remove categorias e pratos, controla a disponibilidade de cada item e entrega o cardápio organizado por categoria para qualquer cliente (React, mobile, Postman).

## 

## Modelo de Dados

Relação **1:N**: uma categoria possui vários pratos; cada prato pertence a uma única categoria.

### Categoria

| Campo     | Tipo   | Regras                                                           |
| --------- | ------ | ---------------------------------------------------------------- |
| id        | number | gerado automaticamente                                           |
| nome      | string | obrigatório, 2–50 caracteres, único (sem diferenciar maiúsculas) |
| descricao | string | opcional, até 200 caracteres                                     |

### Prato

| Campo       | Tipo    | Regras                                                             |
| ----------- | ------- | ------------------------------------------------------------------ |
| id          | number  | gerado automaticamente                                             |
| nome        | string  | obrigatório, 2–80 caracteres                                       |
| descricao   | string  | opcional, até 300 caracteres                                       |
| preco       | number  | obrigatório, maior que zero                                        |
| disponivel  | boolean | opcional, padrão `true`                                            |
| categoriaId | number  | obrigatório, deve referenciar uma categoria existente              |
| excluidoEm  | string  | preenchido pelo soft-delete (data ISO); nunca é retornado pela API |

## Requisitos Funcionais

### CRUD de Categorias

| Código | Requisito                                                                                  |
| ------ | ------------------------------------------------------------------------------------------ |
| RF01   | Cadastrar uma nova categoria (nome e descrição).                                           |
| RF02   | Listar todas as categorias e consultar uma pelo id.                                        |
| RF03   | Editar os dados de uma categoria existente.                                                |
| RF04   | Excluir uma categoria, rejeitando a exclusão (`409`) caso existam pratos vinculados a ela. |

### CRUD de Pratos

| Código | Requisito                                                                                                            |
| ------ | -------------------------------------------------------------------------------------------------------------------- |
| RF05   | Cadastrar um prato (nome, descrição, preço e disponibilidade), obrigatoriamente vinculado a uma categoria existente. |
| RF06   | Listar todos os pratos e consultar um pelo id, retornando junto a categoria (`id` e `nome`).                         |
| RF07   | Editar os dados de um prato, inclusive alterar sua categoria.                                                        |
| RF08   | Excluir um prato (soft-delete), retornando `404` se ele não existir.                                                 |





### Categorias

| Método | Rota              | Descrição              | Sucesso | Erros               |
| ------ | ----------------- | ---------------------- | ------- | ------------------- |
| GET    | `/categorias`     | Lista as categorias    | `200`   | —                   |
| GET    | `/categorias/:id` | Detalha uma categoria  | `200`   | `400`, `404`        |
| POST   | `/categorias`     | Cria uma categoria     | `201`   | `400`, `409`        |
| PUT    | `/categorias/:id` | Atualiza uma categoria | `200`   | `400`, `404`, `409` |
| DELETE | `/categorias/:id` | Remove uma categoria   | `204`   | `400`, `404`, `409` |

### Pratos

| Método | Rota                          | Descrição                     | Sucesso | Erros        |
| ------ | ----------------------------- | ----------------------------- | ------- | ------------ |
| GET    | `/pratos`                     | Lista pratos (aceita filtros) | `200`   | `400`        |
| GET    | `/pratos/:id`                 | Detalha um prato              | `200`   | `400`, `404` |
| POST   | `/pratos`                     | Cria um prato                 | `201`   | `400`        |
| PUT    | `/pratos/:id`                 | Atualiza um prato             | `200`   | `400`, `404` |
| PATCH  | `/pratos/:id/disponibilidade` | Altera só a disponibilidade   | `200`   | `400`, `404` |
| DELETE | `/pratos/:id`                 | Exclui um prato (soft-delete) | `204`   | `400`, `404` |

Parâmetros de `GET /pratos`: `categoriaId` (número), `busca` (texto) e `disponivel` (`true` ou `false`). Podem ser combinados.

### Cardápio

| Método | Rota        | Descrição                              | Sucesso |
| ------ | ----------- | -------------------------------------- | ------- |
| GET    | `/cardapio` | Categorias com seus pratos disponíveis | `200`   |

## Exemplos

**Criar prato** — `POST /api/pratos`

```json
{
  "nome": "Torta de limão",
  "descricao": "Com merengue",
  "preco": 14.9,
  "categoriaId": 4
}
```

Resposta `201 Created`:

```json
{
  "id": 8,
  "nome": "Torta de limão",
  "descricao": "Com merengue",
  "preco": 14.9,
  "disponivel": true,
  "categoriaId": 4,
  "categoria": { "id": 4, "nome": "Sobremesas" }
}
```

**Erro de validação** — `400 Bad Request`

```json
{
  "status": 400,
  "mensagem": "Dados inválidos.",
  "erros": [
    { "campo": "preco", "mensagem": "O preço deve ser maior que zero." },
    { "campo": "categoriaId", "mensagem": "A categoria é obrigatória." }
  ]
}
```

Todas as respostas de erro seguem o formato `{ status, mensagem, erros? }`.

| Status | Quando ocorre                                                                            |
| ------ | ---------------------------------------------------------------------------------------- |
| `400`  | JSON malformado, id inválido, dados que violam o schema zod ou `categoriaId` inexistente |
| `404`  | Categoria, prato ou rota não encontrados                                                 |
| `409`  | Nome de categoria duplicado ou exclusão de categoria com pratos vinculados               |
| `500`  | Erro inesperado (a mensagem é genérica; o detalhe fica só no log do servidor)            |

## Arquitetura em Camadas

```
Requisição → logger → cors → express.json → routes → valida(schema) → controller → service → model → db.js → data/db.json
                                                                                                         ↓
Resposta  ←──────────────────────────────── errorHandler (qualquer erro) ←──────────────────────────────┘
```

| Camada        | Responsabilidade                                                            |
| ------------- | --------------------------------------------------------------------------- |
| `routes`      | Liga método + URL aos middlewares e ao controller                           |
| `middlewares` | `valida(schema)`, `logger`, `notFound`, `errorHandler`                      |
| `controllers` | Lê `req`, chama o service, escolhe o status HTTP e responde                 |
| `services`    | Regras de negócio (unicidade, vínculo com categoria, soft-delete, cardápio) |
| `models`      | Acesso aos dados: consultas e gravações no JSON                             |
| `schemas`     | Regras de validação com zod                                                 |
| `db.js`       | Leitura e gravação do arquivo JSON (gravação atômica e em fila)             |

## Estrutura de Pastas

```
restaurante-api/
├── data/
│   └── db.json                  # persistência (já traz dados de exemplo)
├── src/
│   ├── controllers/
│   │   ├── cardapio.controller.js
│   │   ├── categorias.controller.js
│   │   └── pratos.controller.js
│   ├── middlewares/
│   │   ├── errorHandler.js
│   │   ├── logger.js
│   │   ├── notFound.js
│   │   └── valida.js
│   ├── models/
│   │   ├── categorias.model.js
│   │   └── pratos.model.js
│   ├── routes/
│   │   ├── cardapio.routes.js
│   │   ├── categorias.routes.js
│   │   ├── pratos.routes.js
│   │   └── index.js
│   ├── schemas/
│   │   ├── categoria.schema.js
│   │   └── prato.schema.js
│   ├── services/
│   │   ├── cardapio.service.js
│   │   ├── categorias.service.js
│   │   └── pratos.service.js
│   ├── utils/
│   │   ├── AppError.js
│   │   ├── asyncHandler.js
│   │   ├── normalizar.js
│   │   └── parseId.js
│   ├── app.js                   # monta o Express e os middlewares
│   ├── db.js                    # leitura/gravação do JSON
│   └── server.js                # sobe o servidor
├── tests/
│   └── api.test.js
├── requests.http                # requisições de exemplo
└── package.json
```



## Variáveis de Ambiente

| Variável      | Padrão         | Descrição                                       |
| ------------- | -------------- | ----------------------------------------------- |
| `PORT`        | `3001`         | Porta do servidor                               |
| `CORS_ORIGIN` | `*`            | Origem permitida (ex.: `http://localhost:3000`) |
| `DB_FILE`     | `data/db.json` | Caminho do arquivo de dados                     |

## Como Executar

```bash
# Instalar dependências
npm install

# Iniciar em desenvolvimento (reinicia ao salvar)
npm run dev

# Iniciar normalmente
npm start


```

O servidor sobe em `http://localhost:3001/api`. Para testar, use o arquivo `requests.http` (extensão REST Client do VS Code), o Postman ou o `curl`:

```bash
curl http://localhost:3001/api/cardapio
```

## Próximos Passos

- Trocar `data/db.json` por um banco relacional (SQLite/PostgreSQL) reescrevendo apenas a camada `models`.
- Adicionar autenticação para proteger as rotas de escrita.
- Documentar a API com OpenAPI/Swagger.
