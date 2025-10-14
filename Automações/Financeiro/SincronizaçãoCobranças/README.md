# 🔄 Sincronização Automática de Cobranças

> **Categoria:** Financeiro | **Complexidade:** ⭐⭐⭐⭐ | **Status:** ✅ Em Produção

## 📝 Visão Geral

Criei este workflow para manter um dashboard atualizado de todas as cobranças do Banco Inter. Enquanto o sistema principal gera e envia boletos, este workflow sincroniza periodicamente TODAS as cobranças existentes, atualizando status, valores e dados dos clientes em uma planilha centralizada.

É como um "refresh" completo que busca tudo que está no banco e atualiza a planilha, garantindo que você sempre tenha uma visão consolidada e atualizada de todas as cobranças.

## 🎯 Por Que Criei Esta Automação

**O problema:**
- Sistema principal só processa cobranças no dia do envio
- Não tem visão consolidada de TODAS as cobranças
- Status podem mudar e não serem refletidos imediatamente
- Difícil saber o total de pendências e recebimentos
- Impossível ter dashboard em tempo real

**A solução:**
Um workflow que roda diariamente e:
- Busca TODAS as cobranças dos últimos 90 dias e próximos 30 dias
- Atualiza dados completos de cada uma (com paginação)
- Sincroniza tudo no Google Sheets
- Fornece visão consolidada para análise e dashboards

## 🏗️ Como Funciona

### Fluxo do Processo

```
1. Schedule Trigger (08:10 AM diariamente)
   ↓
2. Define período de busca:
   ├─ Data inicial: hoje - 90 dias
   └─ Data final: hoje + 30 dias
   ↓
3. Limpa planilha "Cobranças" (mantém cabeçalho)
   ↓
4. Autentica no Banco Inter (OAuth2)
   ↓
5. Busca lista de cobranças (paginada):
   ├─ Página 0, 1, 2, 3...
   ├─ Delay de 1 minuto entre páginas
   └─ Continua até última página
   ↓
6. Para cada cobrança listada:
   ├─ Busca detalhes completos (API individual)
   ├─ Formata dados (datas, valores)
   └─ Salva no Google Sheets
   ↓
7. Atualiza para próxima página
   ↓
8. Repete até processar todas as páginas
```

### Diagrama Visual

![Fluxo de Sincronização](./Imagens/fluxo-sync.png)

*Nota: Por questões de confidencialidade, o arquivo JSON não está disponível publicamente.*

## ⚙️ Componentes e Integrações

### Tecnologias Utilizadas:
- **N8N:** Plataforma de automação
- **Banco Inter API:** Listagem e detalhamento de cobranças
- **Google Sheets:** Armazenamento consolidado
- **Schedule:** Execução diária automática

### Nodes do Workflow:
- **Schedule Trigger:** Dispara às 08:10 diariamente
- **Set Credenciais:** Configurações da API
- **Range vencimento:** Calcula período de busca
- **Google Sheets (Clear):** Limpa dados antigos
- **Request Token:** Autenticação OAuth2
- **Edit Fields1:** Controla paginação
- **Request Listar Cobrancas:** Busca lista paginada
- **Split Out:** Separa cobranças individuais
- **Request Recuperar Cobranca:** Busca detalhes de cada uma
- **Edit Fields3:** Formata datas
- **Edit Fields2:** Organiza dados finais
- **Google Sheets1:** Salva/atualiza dados
- **Edit Fields:** Controla fluxo de páginas
- **IF:** Verifica se é última página
- **Wait:** Delay de 1 minuto entre páginas

## 🔧 Funcionalidades Principais

### 1. Período de Busca Inteligente

O workflow define automaticamente um período que cobre cobranças relevantes:

**Cálculo da data inicial (90 dias atrás):**
```javascript
dataInicial = new Date(Date.now() - 90 * 24 * 60 * 60 * 1000)
  .toISOString()
  .split('T')[0]

// Resultado: "2025-07-16" (se hoje é 14/10/2025)
```

**Cálculo da data final (30 dias à frente):**
```javascript
dataFinal = new Date(Date.now() + 30 * 24 * 60 * 60 * 1000)
  .toISOString()
  .split('T')[0]

// Resultado: "2025-11-13" (se hoje é 14/10/2025)
```

**Por que este período?**
- **90 dias atrás:** Cobre boletos em atraso que ainda podem ser pagos
- **30 dias à frente:** Pega boletos já gerados com vencimento futuro
- Total: ~120 dias de cobranças sempre visíveis

### 2. Limpeza Inteligente da Planilha

Antes de sincronizar, limpa dados antigos mas mantém estrutura:

**Operação Clear:**
```javascript
operation: "clear"
keepFirstRow: true  // Mantém cabeçalho
```

**Por que limpar antes?**
- Evita dados duplicados
- Remove cobranças antigas fora do período
- Garante que a planilha reflete exatamente o que está no banco
- Refresh completo a cada execução

### 3. Paginação Automática

A API do Banco Inter retorna resultados em páginas. Implementei loop automático:

**Controle de página:**
```javascript
// Inicialização
paginaAtual = 0

// Após cada busca
paginaAtual = paginaAtual + 1

// Verificação
if (ultimaPagina === true) {
  // Para o loop
} else {
  // Aguarda 1 minuto e busca próxima página
  wait(60000)
  continua...
}
```

**Fluxo de paginação:**
```
Página 0 → Processa cobranças → Não é última → Wait 1 min
  ↓
Página 1 → Processa cobranças → Não é última → Wait 1 min
  ↓
Página 2 → Processa cobranças → Não é última → Wait 1 min
  ↓
Página 3 → Processa cobranças → É última → FIM
```

### 4. Busca Detalhada de Cada Cobrança

A API de listagem retorna dados resumidos. Para ter tudo, busco individualmente:

**Endpoint de listagem:**
```
GET /cobranca/v3/cobrancas?dataInicial=X&dataFinal=Y&paginacao.paginaAtual=N
```

**Retorna:**
```json
{
  "cobrancas": [
    {
      "cobranca": {
        "codigoSolicitacao": "123456",
        // Dados resumidos
      }
    }
  ],
  "ultimaPagina": false
}
```

**Split Out:** Separa cada cobrança

**Para cada uma, busca detalhes:**
```
GET /cobranca/v3/cobrancas/{codigoSolicitacao}
```

**Retorna dados completos:**
```json
{
  "cobranca": {
    "codigoSolicitacao": "123456",
    "situacao": "EMITIDO",
    "dataSituacao": "2025-10-14",
    "dataVencimento": "2025-10-25",
    "valorNominal": 150.00,
    "pagador": {
      "nome": "Cliente Exemplo",
      "email": "cliente@email.com",
      "telefone": "11987654321"
    }
  }
}
```

### 5. Formatação de Datas

Converte datas ISO para formato brasileiro:

**Entrada da API:**
```javascript
dataVencimento: "2025-10-25"
dataSituacao: "2025-10-14T10:30:00"
```

**Conversão:**
```javascript
vencimento = new Date($json.cobranca.dataVencimento)
  .toLocaleDateString('pt-BR')
// Resultado: "25/10/2025"

data = new Date($json.cobranca.dataSituacao)
  .toLocaleDateString('pt-BR')
// Resultado: "14/10/2025"
```

**Por que formatar:**
- Facilita leitura humana na planilha
- Compatível com fórmulas do Google Sheets
- Padronização brasileira (DD/MM/YYYY)

### 6. Salvamento com Upsert

Usa operação `appendOrUpdate` para inteligência no salvamento:

**Configuração:**
```javascript
operation: "appendOrUpdate"
matchingColumns: ["Nome"]  // Chave de identificação
```

**Comportamento:**
- Se "Nome" existe: atualiza a linha
- Se "Nome" não existe: adiciona nova linha
- Mantém dados organizados sem duplicatas por nome

**Colunas salvas:**
- Id (codigoSolicitacao)
- Nome
- Situação (status atual)
- Data Situação
- Vencimento
- Valor
- Email
- Telefone
- Conta (número da conta corrente)

### 7. Rate Limiting com Delays

Para não sobrecarregar a API:

**Delay entre páginas:**
```javascript
wait: 1 minuto
```

**Por que 1 minuto?**
- API do Banco Inter tem limites conservadores
- Cada página pode ter muitas cobranças
- Cada cobrança gera uma requisição de detalhe
- Segurança contra bloqueios

**Exemplo de volume:**
```
Página com 50 cobranças
  ↓
50 requisições de detalhe
  ↓
Wait 1 minuto antes da próxima página
```

## 📊 Estrutura do Google Sheets

### Aba "Cobranças" (Sincronizada)

| Coluna | Descrição | Exemplo |
|--------|-----------|---------|
| Id | Código único do boleto | 00000000-0000-0000... |
| Nome | Nome do cliente | João Silva |
| Situação | Status atual | EMITIDO, PAGO, VENCIDO, CANCELADO |
| Data Situação | Quando mudou status | 14/10/2025 |
| Vencimento | Data de vencimento | 25/10/2025 |
| Valor | Valor do boleto | 150.00 |
| Email | Email do cliente | joao@email.com |
| Telefone | Telefone do cliente | 11987654321 |
| Conta | Conta corrente | 236181483 |

### Possíveis Situações:

- **EMITIDO:** Boleto gerado, aguardando pagamento
- **PAGO/RECEBIDO:** Cliente já pagou
- **VENCIDO:** Passou da data, não pago
- **CANCELADO:** Boleto foi cancelado
- **EXPIRADO:** Boleto expirou
- **AGUARDANDO:** Aguardando processamento

## 🔄 Exemplo Completo de Execução

**08:10 - Início:**
```
Schedule dispara → Define período (16/07 a 13/11)
  ↓
Limpa planilha → Autentica no banco
  ↓
Busca página 0
```

**08:11 - Primeira página:**
```
Retorna 100 cobranças
  ↓
Para cada uma:
  - Busca detalhes
  - Formata dados
  - Salva no Sheets
  ↓
Verifica: não é última página
  ↓
Wait 1 minuto
```

**08:12 - Segunda página:**
```
Busca página 1
  ↓
Retorna 100 cobranças
  ↓
Processa todas...
  ↓
Wait 1 minuto
```

**08:15 - Última página:**
```
Busca página 3
  ↓
Retorna 42 cobranças
  ↓
Processa todas...
  ↓
ultimaPagina = true → FIM
```

**Resultado final:**
- 342 cobranças sincronizadas
- Tempo total: ~5 minutos
- Planilha totalmente atualizada

## 📊 Propósito: Base para Análise e Dashboards

**Este workflow NÃO é apenas para armazenar dados - é para criar uma BASE DE DADOS estruturada que permite:**

### Geração de Insights:
- Análise de tendências de pagamento
- Identificação de padrões de inadimplência
- Projeção de fluxo de caixa
- Segmentação de clientes por comportamento
- Análise de aging (quanto tempo boletos ficam em aberto)

### Criação de Dashboards:
Com os dados estruturados no Google Sheets, você pode conectar ferramentas como:

**Google Data Studio / Looker Studio:**
- Dashboard em tempo real
- Gráficos de status de cobranças
- KPIs financeiros automáticos
- Filtros por período, situação, cliente

**Power BI:**
- Conecta direto com Google Sheets
- Visualizações avançadas
- Relatórios automatizados

**Excel / Google Sheets Nativos:**
- Tabelas dinâmicas
- Gráficos personalizados
- Fórmulas de análise

### Exemplos de Análises Possíveis:

**KPIs Calculados:**
```
- Total a Receber = SOMASE(Situação, "EMITIDO", Valor)
- Total Recebido no Mês = SOMASES(Situação, "PAGO", Data, MêsAtual)
- Taxa de Inadimplência = CONT.SE(Situação, "VENCIDO") / CONT.VALORES(Id)
- Ticket Médio = MÉDIA(Valor)
```

**Dashboards Visuais:**
- Gráfico de pizza: Distribuição por situação
- Gráfico de barras: Valores por mês
- Linha do tempo: Evolução de recebimentos
- Tabela: Top clientes inadimplentes

**Alertas Automatizados:**
- Formatação condicional para boletos vencidos
- Notificações quando inadimplência > X%
- Destaque para valores altos pendentes

## 🎯 Casos de Uso

### Dashboard de Gestão:
- **Objetivo:** Visualizar saúde financeira em tempo real
- **O que ver:** Total pendente, total recebido, inadimplência
- **Frequência:** Atualizado diariamente às 08:10

### Análise Financeira:
- **Objetivo:** Projetar recebimentos e identificar riscos
- **O que analisar:** 
  - Aging de boletos (0-30, 31-60, 60+ dias)
  - Sazonalidade de pagamentos
  - Clientes com histórico de atraso
- **Ação:** Ajustar estratégias de cobrança

### Operacional:
- **Objetivo:** Ações práticas de cobrança
- **O que fazer:**
  - Lista de contatos para follow-up (filtro: VENCIDO)
  - Boletos a serem reenviados (filtro: 5+ dias vencidos)
  - Clientes para cobrança ativa (filtro: valor > R$ 500)

### Estratégico:
- **Objetivo:** Decisões de negócio
- **O que decidir:**
  - Ajustar política de crédito
  - Identificar necessidade de antecipação de recebíveis
  - Avaliar impacto de mudanças de vencimento

## 📊 Da Sincronização ao Dashboard: Fluxo Completo

```
1. N8N sincroniza dados (08:10 diariamente)
   ↓
2. Google Sheets armazena estruturado
   ↓
3. Conecta ferramenta de BI:
   ├─ Google Data Studio (gratuito)
   ├─ Power BI
   ├─ Tableau
   └─ Ou fórmulas nativas do Sheets
   ↓
4. Cria visualizações e KPIs
   ↓
5. Monitora e toma decisões
```

### Exemplo de Dashboard Completo:

**Cabeçalho (Cards de KPIs):**
```
┌─────────────────┬─────────────────┬─────────────────┬─────────────────┐
│ Total Pendente  │ Total Recebido  │ Inadimplência   │ A Vencer (7d)   │
│   R$ 45.230     │   R$ 128.450    │     12.5%       │   R$ 8.950      │
└─────────────────┴─────────────────┴─────────────────┴─────────────────┘
```

**Gráficos:**
```
[Gráfico de Pizza]          [Gráfico de Barras]
Status das Cobranças        Recebimentos por Mês
- PAGO: 68%                 Jan: R$ 85k
- EMITIDO: 20%              Fev: R$ 92k
- VENCIDO: 10%              Mar: R$ 78k
- CANCELADO: 2%             ...
```

**Tabela Filtrada:**
```
Boletos Vencidos (Ação Necessária)
┌──────────────────┬────────────┬──────────┬──────────┐
│ Cliente          │ Vencimento │ Valor    │ Dias     │
├──────────────────┼────────────┼──────────┼──────────┤
│ Empresa A        │ 01/10/2025 │ R$ 1.250 │ 13 dias  │
│ Empresa B        │ 28/09/2025 │ R$ 850   │ 16 dias  │
│ Empresa C        │ 15/09/2025 │ R$ 2.100 │ 29 dias  │
└──────────────────┴────────────┴──────────┴──────────┘
```

### vs. Buscar manualmente:
- ✅ Automático (sem intervenção)
- ✅ Sempre atualizado
- ✅ Dados completos
- ✅ Histórico consolidado

### vs. Só callbacks:
- ✅ Pega cobranças antigas
- ✅ Não depende de webhooks
- ✅ Sincroniza tudo, não só mudanças
- ✅ Backup diário completo

### vs. Sistema principal:
- ✅ Visão de TODAS as cobranças
- ✅ Não só as do dia
- ✅ Consolidação centralizada
- ✅ Base para dashboards

## 🛡️ Tratamento de Erros

### Falha na autenticação:
- Workflow para
- Não processa páginas
- Pode ser executado novamente

### Falha em página específica:
- Página não processada
- Próxima execução tenta novamente
- Dados das páginas anteriores foram salvos

### API indisponível:
- Retry automático (não configurado, mas pode adicionar)
- Próxima execução (24h depois) normaliza

## 🎓 Conceitos Técnicos Aplicados

### Pagination Pattern:
- Loop controlado por flag `ultimaPagina`
- Incremento automático de página
- Condição de parada

### ETL (Extract, Transform, Load):
- **Extract:** API do banco (paginada + detalhes)
- **Transform:** Formatação de datas e dados
- **Load:** Google Sheets com upsert

### Rate Limiting:
- Delays estratégicos
- Respeito aos limites da API
- Processamento estável

### Data Synchronization:
- Full refresh diário
- Clear + Load
- Garante consistência

## 💻 Configurações

### Credenciais necessárias:

```javascript
{
  baseUrl: "https://cdpj.partners.bancointer.com.br",
  clientId: "SEU_CLIENT_ID",
  clientSecret: "SEU_CLIENT_SECRET",
  contaCorrente: "SUA_CONTA",
  planilha: "URL_DA_PLANILHA"
}
```

### Certificado SSL:
- Obrigatório
- Mesmo certificado do sistema principal

### Google Sheets:
- OAuth2 configurado
- Permissões de leitura/escrita
- Aba "Cobranças" criada

## 🚀 Como Usar

### Setup Inicial:

1. **Configure credenciais** no node "Set Credenciais"
2. **Crie aba "Cobranças"** na planilha com cabeçalho
3. **Teste manualmente** antes de ativar schedule
4. **Ative o schedule** para execução diária

### Monitoramento:

- Acompanhe execuções no N8N
- Verifique planilha após execução
- Confirme que dados estão atualizando

### Ajustes de Período:

Para buscar mais/menos dias:
```javascript
// Aumentar retrospectiva (120 dias)
dataInicial = Date.now() - 120 * 24 * 60 * 60 * 1000

// Aumentar projeção (60 dias)
dataFinal = Date.now() + 60 * 24 * 60 * 60 * 1000
```

## ⚠️ Considerações Importantes

### Performance:
- Execução pode levar 5-15 minutos
- Depende do volume de cobranças
- Mais cobranças = mais páginas = mais tempo

### Limites da API:
- Respeite rate limits
- Não reduza muito o delay
- API pode bloquear temporariamente

### Volume de Dados:
- Planilha fica com muitas linhas
- Considere arquivar cobranças muito antigas
- Google Sheets tem limite de ~5 milhões de células

## 📚 Aprendizados

### O que funcionou bem:
- Limpeza prévia garante dados atualizados
- Paginação automática é essencial
- Busca detalhada individual dá dados completos
- Upsert mantém planilha organizada
- Delays previnem bloqueios

### Desafios que enfrentei:
- Entender estrutura de paginação da API
- Calcular período adequado de busca
- Balancear velocidade vs. rate limiting
- Formatar datas corretamente
- Garantir que loop de páginas não trave

### Melhorias futuras:
- Processamento paralelo (com cuidado no rate limit)
- Histórico incremental ao invés de full refresh
- Notificação quando há muitas pendências
- **Automação de criação de dashboard no Data Studio**
- **Envio automático de relatório semanal por email com insights**
- **Alertas inteligentes baseados em ML para prever inadimplência**
- **Integração com ferramentas de BI para atualização em tempo real**

---

## 📄 Notas Importantes

> ⚠️ **Complementaridade:** Este workflow complementa o sistema principal de cobrança, não o substitui.

> 🔄 **Sincronização:** Roda independentemente - mesmo que sistema principal pare, continua sincronizando.

> 📊 **Dashboard:** Use os dados sincronizados para criar dashboards no Google Data Studio ou similar.

> ⏰ **Horário:** Executa às 08:10 para ter dados frescos no início do dia.

---

**Criado em:** 2025  
**Última atualização:** 13/10/2025  
**Status:** ✅ Em produção  
**Frequência:** Diária (08:10 AM)
