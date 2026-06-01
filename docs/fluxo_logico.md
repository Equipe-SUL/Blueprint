# Fluxo Lógico — Blueprint

Pipeline completo de processamento de plantas DXF até memorial descritivo e orçamento SINAPI.

## Entrada

Arquivo DXF de projeto arquitetônico/estrutural.

### Camadas estruturais detectáveis (dxf_core)

`PILAR`, `VIGA`/`VIGAS`, `ESTACA`, `LAJE`, `FUNDACAO`, `PAREDE`, `PISO`, `ESQUADRIA`, `HACH`

---

## 1. CAD Engine (processamento geométrico)

**Entrada:** `apps/projetos/ai/cad/engine.py :: process_dxf()`

### 1a. DXF Parser
`apps/projetos/ai/cad/dxf_parser.py`
- Lê todas as entidades (sem whitelist)
- Extrai: linhas, arcos, polylinhas, blocos INSERT (ADC_PORTAS, ADC_LOUCAS...)
- Extrai: TEXT/MTEXT por camada (TXT_AMBIENTE, TXT_AREA, TXT_GERAL)
- Extrai: dimensões ARQ-COTAS
- Saída: `DrawingData` (entities, blocks, dimensions, layers)

### 1b. Flatten
`apps/projetos/ai/cad/curve_resolution.py`
- Transforma curvas em segmentos de reta

### 1c. Healing
`apps/projetos/ai/cad/healer.py`
- Snap de vértices próximos
- Merge de segmentos colineares sobrepostos
- Reduz ~7700 segmentos brutos → ~3600 healed

### 1d. Polygonizer
`apps/projetos/ai/cad/polygonizer.py`
- Constrói grafo de adjacência
- Detecta ciclos (polígonos fechados)
- Saída: ~406 polígonos

### 1e. Validation
`apps/projetos/ai/cad/validation.py`
- Valida anéis (ring winding, self-intersection)
- Repara polígonos inválidos

### 1f. Classifier
`apps/projetos/ai/cad/classifier.py`
- Associa textos TXT_AMBIENTE → ambientes
- Fallback: `text_rooms.py` (emparelhamento TXT_AMBIENTE + TXT_AREA)

### 1g. Metrics
`apps/projetos/ai/cad/metrics.py`
- Calcula área, perímetro de cada ambiente
- Matriz de adjacência entre ambientes
- Detecta `used_text_fallback` (se áreas vieram dos textos)

### 1h. Topology
`apps/projetos/ai/cad/topology.py`
- Grafo NetworkX com vértices/arestas
- Componentes conexos

### 1i. Full Report
Monta relatório completo com camadas, blocos_por_tipo, blocos_detalhados, dimensões, textos_por_camada, total_entidades, unidade.

**Saída:** `CADResult(rooms, geojson, metrics, topology, full_report, texts, stats, success, error, used_text_fallback)`

---

## 2. Extração Estrutural (dxf_core — fallback clássico)

**Entrada:** `apps/projetos/ai/extracaocalculo/dxf_core.py :: extrair_dxf()`
- Lê entidades DXF layer por layer
- Processa: PAREDES, PILARES, VIGAS, LAJES, ESQUADRIAS, HACH, PISO
- Calcula: área_total_m2, comprimento_total_m, volume_m3
- Saída: `MemorialCalculo(resumo_por_camada, ambientes, textos_legenda)`

---

## 3. Análise Estrutural Geométrica (opcional — apenas DXFs estruturais)

**Entrada:** `apps/projetos/ai/cad/structural_analysis.py :: StructuralAnalyzer`
- Segmenta ModelSpace em vistas por histograma de densidade X (HISTOGRAM_BINS=20, valley threshold=8% do pico)
- Por vista: detecta pilares, vigas, estacas, fundações
- Deduplica pilares/estacas (max entre vistas)
- Soma vigas (pavimentos diferentes)
- Estima volumes com seções típicas (viga 15×40cm, estaca prof. 10m)
- Inerte para DXFs arquitetônicos (sem camadas PILAR/VIGAS/ESTACA)

---

## 4. Normas Técnicas (RAG + NBRs)

`apps/projetos/ai/nbrs.py`
- `inject_nbrs_into_prompt(prompt, contexto_obra)`:
  - Se `contexto_obra` fornecido: RAG no ChromaDB (k=5 chunks)
  - Senão: fallback estático com 5 NBRs principais

`apps/projetos/ai/rag/ingest_nbrs.py`
- Extrai texto de PDFs em `nbrs/` via PyMuPDF
- Chunk: 800 chars, 150 overlap
- Indexa em `nbrs_collection` no ChromaDB
- Embedding: `paraphrase-multilingual-MiniLM-L12-v2`

---

## 5. LangGraph — Memorial Descritivo

`apps/projetos/ai/grafo_novo/`

### State
`state_descritivo.py` — `DescritivoState` (DXF path, CAD result, LLM output, PDF path...)

### Nó 2b: extrair_combinado()
`nodes_descritivo.py`
- Executa CAD Engine + dxf_core
- Mescla resultados: ambientes + full_report + resumo_por_camada
- Respeita `used_text_fallback` para áreas

### Nó 3: processar_com_llm()
`nodes_descritivo.py`
- Chama modelo via LangChain (gemma4:31b-cloud)
- Prompt: `SYSTEM_PROMPT_AUDITOR` + `USER_PROMPT_MEMORIAL_DESCRITIVO`
- Processa: ambientes, blocos, dimensões, textos, camadas
- Saída: JSON do memorial descritivo

---

## 6. LLM + PDF

`apps/projetos/ai/prompts_descritivo.py`
- `SYSTEM_PROMPT_AUDITOR`: contexto técnico com 7 seções (layers, blocos ADC, tipologia geminada, hierarquia de confiança)
- `USER_PROMPT_MEMORIAL_DESCRITIVO`: template com dados do CAD

`apps/projetos/ai/services/pdf_generator.py :: gerar_pdf_memorial_descritivo()`
- Gera PDF formatado com reportlab
- Seções: cabeçalho, ambientes, elementos estruturais, instalações, observações técnicas

---

## 7. Orçamento SINAPI

### Adapter
`apps/projetos/ai/services/adapter.py`
- `adaptar_memorial_para_orcamento()`: converte `resumo_por_camada` → itens
- `MAPA_CAMADAS`: 12 camadas (PILAR, VIGA, VIGAS, LAJE, PAREDE, etc)
- Estruturais (PILAR, VIGA, ESTACA, FUNDACAO): usa `volume_m3`
- Laje: `area_total_m2`. Esquadria: `comprimento_total_m`
- Camadas desconhecidas: fallback com descrição formatada
- `MAPA_BLOCOS`: 23 blocos mapeados (CAMAC, CAMAS, BACIA, PORTA*, JANELA*, etc)
- Mobília (SOFA*, TV, MESA*, VEG1): filtrada pré-LLM

### SINAPI Matcher
`apps/projetos/ai/sinapi_matcher.py`
- `_carregar_df()`: carrega `SINAPI_Referencia_2025_12.xlsx` (4.820 linhas)
- `buscar_por_palavras_chave()`: filtro por keywords (com bigrams), peso dobrado na primeira keyword, remoção de score 0
- `ESTIMATIVAS_SERVICOS`: 19 serviços CUB (alvenaria, contrapiso, telhado, pilar R$ 1.000/m³, viga R$ 950/m³, estaca R$ 650/m³, concreto R$ 800/m³...)
- Score-based matching: primeira keyword obrigatória, ratio ≥ 40%
- `match_com_llm()`: verifica mobília/hachura/contorno/estimativas CUB antes de chamar LLM. LLM só é chamado com candidatos reais
- `PROMPT_SINAPI_MATCH`: prioriza KIT PORTA PRONTA sobre folha avulsa, CHUVEIRO COMUM sobre braço, LAVATÓRIO/CUBA sobre serviço abertura
- Regra 6: aceita material/louça como fallback com justificativa MATERIAL

### Orçamento Service
`apps/projetos/ai/services/orcamento_service.py`
- `gerar_sugestoes_orcamento()`: top_k=30 candidatos, LLM top_k=20
- `buscar_itens_para_selecao()`: inclui ESTIMATIVA CUB como primeira opção selecionada mesmo sem candidatos SINAPI
- `calcular_orcamento_final()`: soma quantidades × preços + BDI (25%)

### Export
`apps/projetos/ai/services/export_orcamento.py`
- `exportar_csv()`: saída `.xlsx` com openpyxl
- `exportar_pdf()`: saída PDF com tabela formatada + resumo

---

## 8. LangGraph — Nó de Orçamento

### Nó 4: processar_orcamento()
- Chama adapter → sinapi_matcher → orcamento_service
- Cria `ItemProjeto` no banco (apenas itens com matching SINAPI)
- Gera `.xlsx` para download

---

## Diagrama Resumido

```
DXF (.dxf)
  │
  ├──→ CAD Engine (engine.py)
  │      ├──→ Parser → Healer → Polygonizer → Validator
  │      ├──→ Classifier → Metrics → Topology → Full Report
  │      └──→ rooms, geojson, blocos, full_report
  │
  ├──→ dxf_core (extrair_dxf)
  │      └──→ resumo_por_camada (camadas estruturais)
  │
  ├──→ structural_analysis.py (se DXF estrutural)
  │      ├──→ histograma X → vistas → geometria
  │      ├──→ pilares/estacas deduplicados, vigas somadas
  │      └──→ volumes estimados com seções típicas
  │
  ├──→ nbrs.py (RAG ChromaDB → chunks NBR)
  │      └──→ 5 chunks mais relevantes para o prompt LLM
  │
  ├──→ Nó 2b (nodes_descritivo.py)
  │      └──→ Mescla CAD Engine + dxf_core
  │      ↓
  ├──→ Nó 3 (processar_com_llm)
  │      ├──→ LLM → Memorial JSON (com normas NBR)
  │      └──→ pdf_generator.py → pdfs/memorial_*.pdf
  │
  └──→ Nó 4 (processar_orcamento)
         ├──→ adapter.py (converte blocos CAD + estrutural)
         ├──→ sinapi_matcher.py (keyword + bigrams → candidatos SINAPI)
         │      ├──→ ESTIMATIVAS_SERVICOS (19 CUB para itens sem código)
         │      ├──→ blocklist mobília/hachura/contorno
         │      └──→ match_com_llm (fallback)
         ├──→ orcamento_service.py (calcula preços + BDI 25%)
         └──→ export_orcamento.py → .xlsx
```

## Arquivos Principais

| Módulo | Arquivo | Função |
|---|---|---|
| CAD Engine | `cad/engine.py` | Orquestrador CAD |
|  | `cad/dxf_parser.py` | Parser DXF |
|  | `cad/healer.py` | Healing segmentos |
|  | `cad/polygonizer.py` | Poligonização |
|  | `cad/validation.py` | Validação |
|  | `cad/classifier.py` | Classificação |
|  | `cad/metrics.py` | Métricas |
|  | `cad/topology.py` | Topologia |
|  | `cad/text_rooms.py` | Fallback textual |
|  | `cad/structural_analysis.py` | Análise geométrica estrutural |
| Modelos | `cad/ir.py` | DrawingData, CADResult |
| Extração | `extracaocalculo/dxf_core.py` | Extração clássica |
| LangGraph | `grafo_novo/state_descritivo.py` | Estado |
|  | `grafo_novo/nodes_descritivo.py` | Nós |
| Serviços | `services/adapter.py` | Adaptador CAD → orçamento |
|  | `services/orcamento_service.py` | Cálculo orçamento |
|  | `services/export_orcamento.py` | Export .xlsx |
|  | `services/pdf_generator.py` | PDF memorial |
| Matching | `sinapi_matcher.py` | Matching SINAPI (keyword + LLM + CUB) |
| Prompts | `prompts_descritivo.py` | Prompts LLM |
| RAG | `nbrs.py` | Integração NBRs |
| Cliente | `client.py` | Cliente Ollama |
