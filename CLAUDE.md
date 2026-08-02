# CLAUDE.md — App de Gestão SER.RA

> Este ficheiro dá contexto ao Claude Code sobre o projecto. Mantém-no actualizado quando as regras de negócio ou a estrutura mudarem. Última revisão: 2026-08-02.

## O que é este projecto

Aplicação web interna da **SER.RA** (empresa de construção, zona de Sintra/Lisboa/Cascais) para gestão de trabalhadores em obra. Evoluiu de um tracker em Excel para uma aplicação web com persistência na cloud.

**Funções principais:**
- Alocação diária de trabalhadores a obras
- Registo de presenças com vários estados (presente, falta, férias, baixa, etc.)
- Código de cores por obra
- Registo de adiantamentos e pagamentos
- Dados de referência de payroll por trabalhador
- Resumo de custos por obra
- Exportação para Excel
- **Em desenvolvimento:** check-in por GPS/QR code com validação de localização e workflow de notificação ao admin quando o check-in é feito fora da área da obra

**Utilizadores:** o dono da empresa (admin) e, no módulo de check-in, os trabalhadores no terreno via telemóvel.

## Stack técnica

- **Frontend:** HTML/CSS/JavaScript vanilla, historicamente num único ficheiro HTML. Sem framework, sem build step.
- **Backend/BD:** Supabase (projecto SER.RA — atenção: existiu confusão com outro projecto Supabase chamado "Personal Trainer"; confirmar sempre que se está a trabalhar no projecto certo antes de correr SQL ou migrações).
- **Hosting:** GitHub Pages (ser-ra.github.io). Deploy = push para o branch publicado.
- **Domínio:** ser-ra.pt gerido em PowerDNS; www aponta por CNAME para Vercel (site institucional — NÃO é esta app).

> ⚠️ **VERIFICAR SEMPRE:** a secção seguinte foi preenchida via MCP do Supabase em 2026-08-02; antes de qualquer alteração à BD, reconfirmar via MCP que o esquema não mudou entretanto. Não assumir nomes de memória.

## Esquema Supabase (confirmado em 2026-08-02 via MCP)

**Projecto:** `SER-RA - Portal` (id `ysuukjbejadpgdzbotgj`, região eu-west-1, Postgres 17). É o único projecto Supabase acessível — a confusão com o "Personal Trainer" já não se aplica.

A BD evoluiu muito para além do tracker original: é hoje um portal completo com RH, obras, financeiro, documentos e tarefas. ~48 tabelas no schema `public`, todas com **RLS activo**. Colunas de auditoria comuns: `criado_em`, `actualizado_por`, `actualizado_em`.

### Núcleo — trabalhadores e presenças
- `trabalhadores` — id, nome, alcunha, numero, categoria_id→categorias, fornecedor_id→fornecedores, activo, user_id, custo_dia_real, custo_dia_extra, custo_hora, dias_ferias_ano + dados pessoais (nif, niss, iban, cartao_cidadao, morada, …)
- `categorias` — id, nome, tipo, valor_dia_referencia, valor_dia_extra_referencia
- `alocacoes` — id, trabalhador_id, obra_id, data, estado, origem, acrescimo_pct, picagem_id→picagens. **É a tabela relacional de presenças actual.**
- `presencas` — key, data (jsonb), updated_at. **Legado**: formato chave/valor da app antiga; não confundir com `alocacoes`.
- `ausencias` — id, trabalhador_id, tipo, data_inicio, data_fim, dias_uteis, estado, aprovado_por (workflow de aprovação)
- `adiantamentos` — id, trabalhador_id, data, valor, nota
- `picagens` (check-in GPS/QR) — id, trabalhador_id, obra_id, tipo, momento, latitude, longitude, precisao_metros, distancia_metros, qr_valido, estado, justificacao, foto_path, nota_admin, validado_por/validado_em (workflow de validação pelo admin)
- `notas_dia` — data, nota

### RH / contratual
- `contratos_trabalho` — trabalhador_id, tipo, datas, categoria_id, retribuicao_base, subsídios, valor_real_pago, sem_recibo, activo
- `parametros_rh` — vigente_desde, taxa_ss_trabalhador, taxa_tsu_patronal, seguro_at_mes, sst_mes, formacao_ano, dias_uteis_mes, horas_dia (**taxas fiscais vêm daqui, não hardcoded**)
- `documentos_trabalhador`, `formacoes`, `declaracoes`, `pedidos_alteracao` (workflow de pedidos do trabalhador)

### Obras
- `obras` — id, codigo, nome, cliente_id→clientes, estado, cor, morada, fase, **latitude, longitude, raio_metros, qr_token** (check-in), modelo_faturacao, fee_percentagem, regime_iva_mao_obra, taxa_iva + áreas/tipologia
- `obra_precos` — obra_id, categoria_id, valor_dia, valor_dia_extra (preço de venda por categoria e obra)
- `tarefas` (+ `tarefa_ficheiros`), `documentos_obra` (+ `documento_ficheiros`), `comentarios_obra`, `aprovacoes_obra` (portal do cliente), `orcamentos` + `orcamento_linhas`, `horas_escritorio`, `fases_projecto`, `especialidades`, `engenheiros`

### Financeiro
- `clientes` (+ `cliente_contribuintes`), `fornecedores`
- `pagamentos` — obra_id, fornecedor_id, tipo_documento, numero_documento, **atcud, qr_bruto** (leitura de QR de facturas), base_tributavel, taxa_iva, total, autoliquidacao, refactura_cliente, estado_pagamento
- `recebimentos` — obra_id, cliente_id, contribuinte_id, atcud, inclui_fee, estado, valor_liquidado
- `autos` — obra_id, tipo, periodo, dias/custo/venda mão de obra e despesas, fee, total_cliente; `auto_faturas` liga autos↔pagamentos
- `categorias_movimento` + `subcategorias_movimento`
- `imp_clientes`, `imp_fornecedores`, `imp_pagamentos`, `imp_recebimentos` — staging de importação do PowerApps (texto cru)

### Sistema
- `perfis` — user_id, **papel** (admin/escritorio/trabalhador/cliente — base do RLS), trabalhador_id, cliente_id, obra_id
- `auditoria` (log jsonb), `lixeira` (soft-delete, só superadmin), `config` (key/jsonb), `prazos_sistema`, `assinaturas` + `caixas_partilhadas`, `ligacoes_microsoft` (tokens OAuth — **RLS activo mas 0 políticas = inacessível via API; não mexer**)

### RLS (padrão)
Papéis vindos de `perfis.papel`. Padrões de política: `admin_total` (quase todas), `escritorio_*` (le/escreve/actualiza em alocações, docs, tarefas, pagamentos próprios), `trab_*` (vê as suas picagens/ausências/declarações, cria pedidos), `cliente_*` (lê docs/tarefas/fases visíveis, comenta e aprova), `lixeira_superadmin`. Antes de criar tabela nova, seguir este padrão.

> ⚠️ O ficheiro `trabalhadores_1.html` (Downloads) é uma versão **antiga** da ficha de trabalhadores: espera colunas (`empresa`, `venc_base`, `sub_natal`, `nickname`, `valor_real`, …) que **já não existem** na tabela `trabalhadores` actual. Não usar como referência do esquema; o payroll vive agora em `contratos_trabalho` + `parametros_rh`.

## Regras de negócio (importantes — não alterar sem instrução explícita)

### Custos de trabalhadores
- O custo real diário de um trabalhador inclui: salário base + **TSU patronal a 23,75%** + seguro de acidentes de trabalho + subsídio de alimentação + bónus/prémios quando aplicável.
- A margem por trabalhador compara o custo real diário com a taxa diária facturada ao cliente.
- Valores monetários em EUR, formato português (1 234,56 €).

### Presenças
- Cada trabalhador tem um estado por dia (presente numa obra, falta, férias, baixa, etc. — confirmar lista exacta no código).
- Cada obra tem uma cor associada usada consistentemente em toda a UI.

### Check-in GPS/QR (módulo em desenvolvimento)
- O trabalhador faz check-in via QR code e/ou GPS no local da obra.
- Se o check-in for feito **fora da área definida da obra**, não é rejeitado automaticamente — entra num workflow de **notificação/aprovação pelo admin**.
- Ter em conta a precisão limitada do GPS em telemóveis (não usar raios demasiado apertados).

### Fiscal/legal (contexto Portugal)
- Facturas portuguesas têm obrigatoriamente **QR code e ATCUD** — é a base do módulo de leitura de documentos na app de gestão de projectos (projecto paralelo, não confundir com esta app).

## Convenções

- **Idioma da UI e de todos os textos visíveis: Português de Portugal** (pré-acordo ortográfico onde o dono assim escreve, ex.: "projecto", "actual"). Nunca usar Português do Brasil.
- Datas em formato dd/mm/aaaa; semana começa à segunda-feira.
- Código: comentários em português são aceitáveis; nomes de variáveis/funções podem estar em inglês ou português — seguir o padrão já existente no ficheiro.
- Preservar TODAS as funcionalidades existentes em cada alteração. O histórico deste projecto inclui casos de funcionalidades perdidas em reescritas — fazer alterações cirúrgicas, nunca reescrever ficheiros inteiros sem necessidade.
- Exportação Excel: manter compatibilidade com o formato actual (o dono usa os exports em processos reais de payroll).

## Como trabalhar neste repo

1. **Antes de alterar:** ler o código relevante e, se envolver dados, confirmar o esquema real no Supabase.
2. **Alterações à BD:** propor o SQL e explicar o impacto ANTES de executar. Nunca correr `DROP` ou migrações destrutivas sem confirmação explícita.
3. **Testar:** abrir o HTML localmente ou descrever exactamente como testar. Não assumir que "compila logo funciona".
4. **Commits:** mensagens curtas em português, uma alteração lógica por commit.
5. **Deploy:** push só depois de confirmado com o dono, porque o GitHub Pages publica directamente para produção (não há ambiente de staging).
6. **Dúvidas sobre regras de negócio:** perguntar. Não inventar valores fiscais, taxas ou lógica de pagamento.

## O que NÃO fazer

- Não tocar em chaves/segredos do Supabase nem os expor em commits.
- Não alterar a lógica de cálculo de custos (TSU, subsídios) sem instrução explícita.
- Não mudar o idioma da UI.
- Não converter o projecto para um framework (React, etc.) sem discussão prévia — a simplicidade do stack é intencional.
- Não confundir este repo com: o site institucional (Vercel), a app de defeitos de obra (Firebase), a app de gestão de projectos/leitura de facturas, ou a ferramenta de análise de payroll — são projectos separados.

## Roadmap conhecido

- [ ] Finalizar check-in GPS/QR com workflow de aprovação de check-ins fora de área
- [ ] Possível modularização do ficheiro HTML monolítico (discutir estrutura antes de executar)
- [ ] (adicionar aqui à medida que surgir)
