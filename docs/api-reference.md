# API Reference — Blueprint Backend

Base URL: `http://127.0.0.1:8000/api/`

## Autenticação

Todas as rotas protegidas exigem header:
```
Authorization: Bearer <access_token>
```

---

### `POST /api/users/cadastro/`
Registro de novo usuário militar.

**Body:**
```json
{
  "first_name": "João Silva",
  "email": "joao@eb.mil.br",
  "telefone": "11999999999",
  "matricula": "123456",
  "patente": "Capitão",
  "unidade": "6° Batalhão de Infantaria Leve do Exército",
  "senha": "sua_senha"
}
```

**Response** `201`:
```json
{ "id": 1, "first_name": "João Silva", "email": "joao@eb.mil.br" }
```

---

### `POST /api/users/login/`
Login — retorna tokens JWT.

**Body:**
```json
{ "username": "joao@eb.mil.br", "password": "sua_senha" }
```

**Response** `200`:
```json
{
  "access": "eyJ...",
  "refresh": "eyJ..."
}
```

---

### `POST /api/users/token/refresh/`
Renovar access token.

**Body:**
```json
{ "refresh": "eyJ..." }
```

**Response** `200`: `{ "access": "eyJ..." }`

---

### `GET /api/users/me/`
Dados do usuário logado.

**Response** `200`:
```json
{
  "id": 1,
  "first_name": "João Silva",
  "email": "joao@eb.mil.br",
  "username": "joao@eb.mil.br"
}
```

---

## Projetos (Obras)

### `GET /api/projetos/`
Listar todas as obras.

**Response** `200`: `[ { "id": 1, "nome_obra": "...", ... } ]`

---

### `POST /api/projetos/`
Criar obra.

**Body:**
```json
{
  "nome_obra": "Edifício Residencial",
  "cidade_obra": "São Paulo",
  "estado_obra": "SP",
  "desc_obra": "Construção de edifício de 12 andares"
}
```

---

### `GET /api/projetos/{id}/`
Detalhes de uma obra.

---

### `PATCH /api/projetos/{id}/`
Atualizar dados da obra.

---

### `DELETE /api/projetos/{id}/`
Excluir obra.

---

### `GET /api/dashboard/stats/`
Estatísticas do dashboard.

**Response** `200`:
```json
{ "total_obras": 5, "total_arquivos": 12, "total_materiais": 48 }
```

---

## Arquivos DXF

### `GET /api/projetos/{projeto_id}/upload/`
Listar arquivos da obra (remove do banco os órfãos do storage).

**Response** `200`:
```json
[
  {
    "id": 1,
    "projeto": 1,
    "nome_original": "planta.dxf",
    "status_processamento": "pendente",
    "tamanho_mb": 2.45,
    "enviado_em": "2025-12-01T10:00:00Z"
  }
]
```

---

### `POST /api/projetos/{projeto_id}/upload/`
Upload de arquivo DXF. Multipart form-data, campo `arquivo`.

**Response** `201`: dados do arquivo criado.

---

### `DELETE /api/projetos/{projeto_id}/upload/{arquivo_id}/`
Remover arquivo (físico + registro).

**Response** `204`.

---

## Processamento

### `POST /api/projetos/{projeto_id}/processar/{arquivo_id}/`
Gerar Memorial Descritivo.

**Body (opcional):**
```json
{
  "use_cad_engine": true,
  "tipo_construcao": "residencial",
  "padrao_acabamento": "alto"
}
```

**Response** `200`:
```json
{
  "projeto_id": 1,
  "arquivo_id": 1,
  "sucesso": true,
  "memorial_db_id": 5,
  "status_processamento": "processado",
  "pdf_path": "/memoriais/descritivo/memorial_1_...pdf",
  "inconsistencias": [],
  "confianca": 0.92,
  "memorial": { ... }
}
```

---

### `GET /api/projetos/{projeto_id}/memorial/{memorial_id}/pdf/`
Servir PDF do memorial descritivo (inline para visualização no navegador).

**Response** `200`: `application/pdf`.

---

### `POST /api/projetos/{projeto_id}/gerar-orcamento/{arquivo_id}/`
Gerar Orçamento SINAPI a partir do DXF.

**Body (opcional):**
```json
{ "taxa_bdi": 25.0 }
```

**Response** `200`: arquivo `.xlsx` (download).

---

### `GET /api/projetos/{projeto_id}/orcamento/{memorial_id}/pdf/`
Servir PDF do orçamento.

**Response** `200`: `application/pdf`.

---

## Itens (Materiais)

### `POST /api/projetos/{projeto_id}/itens/`
Criar item próprio (não-SINAPI).

**Body:**
```json
{
  "descricao_original": "Porta de madeira 80cm",
  "unidade": "un",
  "quantidade": 10,
  "preco_unitario": 250.00
}
```

---

### `GET /api/projetos/{projeto_id}/itens/`
Listar itens da obra.

**Response** `200`:
```json
{
  "message": "Itens do projeto",
  "data": [
    {
      "id": 1,
      "descricao_original": "...",
      "unidade": "m²",
      "quantidade": 150.0,
      "preco_unitario": 85.50,
      "origem": "sinapi",
      "status_mapeamento": "mapeado",
      "sinapi_codigo": "12345",
      "sinapi_descricao": "..."
    }
  ]
}
```

---

### `PATCH /api/projetos/{projeto_id}/itens/{item_id}/`
Atualizar item. Itens SINAPI só permitem alterar `quantidade`; itens PRÓPRIOS permitem todos os campos.

---

### `DELETE /api/projetos/{projeto_id}/itens/{item_id}/`
Excluir item.

**Response** `204`.

---

### `GET /api/projetos/{projeto_id}/exportar-materiais/`
Exportar todos os itens como `.xlsx`.

**Response** `200`: `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`.

---

## Utilitários

### `GET /api/server/`
Health check.

**Response** `200`: `{ "status": "online" }`
