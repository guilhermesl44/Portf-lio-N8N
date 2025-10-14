# 💳 Sistema de Cobrança Automatizada com Banco Inter

> **Categoria:** Financeiro | **Complexidade:** ⭐⭐⭐⭐⭐ | **Status:** ✅ Em Produção

## 📝 Visão Geral

Criei este workflow para automatizar completamente o processo de cobrança recorrente usando a API do Banco Inter. O sistema gerencia todo o ciclo: desde a geração dos boletos até o envio por email e atualização de status baseada em callbacks do banco.

O diferencial é que tudo acontece automaticamente - o workflow roda diariamente, gera os boletos conforme a data de envio programada, envia por email com template personalizado e atualiza os status quando o cliente paga ou o boleto atrasa.

## 🎯 Por Que Criei Esta Automação

**O problema:**
- Gerar boletos manualmente para dezenas de clientes é trabalhoso
- Enviar boletos por email um por um consome horas
- Controlar status de pagamento manualmente gera erros
- Atualizar planilhas de cobrança é repetitivo e sujeito a falhas
- Perder datas de envio prejudica o fluxo de caixa

**A solução:**
Um sistema 100% automatizado que:
- Roda diariamente e verifica quais boletos devem ser enviados
- Gera boletos automaticamente via API do Banco Inter
- Envia emails com template profissional e boleto anexo
- Recebe callbacks do banco quando há mudanças (pagamento, atraso, etc)
- Atualiza automaticamente todas as planilhas de controle

## 🏗️ Como Funciona

### Fluxo Principal (Agendado Diariamente)

```
1. Schedule Trigger (09:05 AM)
   ↓
2. Busca clientes no Google Sheets
   ↓
3. Para cada cliente:
   ├─ Atualiza status para "VERIFICANDO"
   ├─ Calcula data de vencimento
   ├─ Verifica se já existe cobrança
   ├─ Se não existe, cria nova
   ├─ Verifica se é data de envio
   └─ Se sim, processa envio
   ↓
4. Processo de Envio:
   ├─ Autentica no Banco Inter (token OAuth2)
   ├─ Gera boleto via API (se não existir)
   ├─ Busca PDF do boleto
   ├─ Envia email com template HTML
   └─ Atualiza status para "PENDENTE"
   ↓
5. Aguarda 6 segundos e processa próximo cliente
```

### Fluxo de Callbacks (Webhook)

```
1. Banco Inter envia callback
   ↓
2. Identifica tipo de evento:
   ├─ PAGO → Atualiza status + próximo mês
   ├─ ATRASADO → Marca pendência
   ├─ CANCELADO → Libera próximo mês
   └─ EXPIRADO → Libera próximo mês
   ↓
3. Atualiza Google Sheets
   ├─ Aba "Emissões" (status do boleto)
   └─ Aba "Clientes" (status e próxima cobrança)
```

### Diagrama Visual

![Fluxo Completo](./Imagens/EMISSAODEBOLETOS.jpg)

*Nota: Por questões de confidencialidade, o arquivo JSON não está disponível publicamente.*

## ⚙️ Componentes e Integrações

### Tecnologias Utilizadas:
- **N8N:** Plataforma de automação
- **Banco Inter API:** Geração e gestão de boletos
- **Google Sheets:** Banco de dados de clientes e cobranças
- **Gmail:** Envio de emails com templates
- **Redis:** Cache de tokens de autenticação
- **Webhook:** Recebimento de callbacks

### Principais Nodes:
- **Schedule Trigger:** Execução diária às 09:05
- **Webhook (2x):** Callbacks e trigger manual de atualização
- **Google Sheets (9x):** Leitura e atualização de dados
- **HTTP Request (3x):** Chamadas à API do Banco Inter
- **Gmail:** Envio de emails
- **Redis (3x):** Gerenciamento de token OAuth2
- **IF/Switch (8x):** Lógica condicional
- **Wait (4x):** Delays entre requisições
- **Split In Batches:** Loop pelos clientes
- **Code:** Cálculos de datas e IDs

## 🔧 Funcionalidades Principais

### 1. Gestão Inteligente de Tokens OAuth2

O Banco Inter usa autenticação OAuth2 com tokens que expiram. Criei um sistema de cache:

**Como funciona:**
1. Antes de qualquer requisição, busca token no Redis
2. Se não existir ou estiver expirado, solicita novo
3. Armazena no Redis com TTL de 50 minutos
4. Reutiliza o mesmo token em todas as requisições

**Por que é importante:**
Evita fazer autenticação a cada requisição, economizando tempo e prevenindo rate limiting.

**Código de exemplo:**
```javascript
// Busca token no cache
token = await redis.get('banco_inter_token')

// Se não existe, autentica
if (!token) {
  token = await requestNewToken()
  await redis.set('banco_inter_token', token, { ttl: 3000 })
}
```

### 2. Sistema de Cobrança Dupla Verificação

Antes de gerar um boleto, o sistema verifica se já existe:

**Fluxo de verificação:**
1. Calcula ID único: `diaVencimento + mês + ano + CPF/CNPJ`
2. Busca na aba "Emissões" por este ID
3. Se existe: reutiliza dados existentes
4. Se não existe: cria novo registro

**Por que é importante:**
Evita duplicação de boletos para o mesmo cliente/mês.

### 3. Cálculo Automático de Datas

O sistema calcula automaticamente várias datas:

**Data de Vencimento:**
```javascript
vencimento = `${ano}-${mesProxCobrança}-${diaVencimento}`
// Exemplo: 2025-11-15
```

**Data de Envio (antecipada):**
```javascript
dataEnvio = WORKDAY(vencimento, -numDiasAntecedencia)
// Usa função WORKDAY do Sheets para excluir fins de semana
```

**Verificação diária:**
```javascript
if (dataEnvio === dataAtual) {
  // Processa envio do boleto
}
```

### 4. Geração de Boletos via API

Integração completa com API do Banco Inter:

**Payload enviado:**
```json
{
  "valorNominal": "150.00",
  "seuNumero": "1-15-11-2025",
  "dataVencimento": "2025-11-15",
  "numDiasAgenda": "60",
  "pagador": {
    "cpfCnpj": "12345678901",
    "tipoPessoa": "FISICA",
    "nome": "Cliente Exemplo",
    "endereco": "Rua Exemplo, 123",
    "cidade": "São Paulo",
    "uf": "SP",
    "cep": "01234567",
    "bairro": "Centro"
  },
  "multa": {
    "codigo": "PERCENTUAL",
    "taxa": "2.00"
  },
  "mora": {
    "codigo": "TAXAMENSAL",
    "taxa": "1.00"
  }
}
```

**Resposta:**
```json
{
  "codigoSolicitacao": "00000000-0000-0000-0000-000000000000"
}
```

### 5. Busca e Conversão de PDF

Após gerar o boleto, busca o PDF:

**Processo:**
1. Requisição para endpoint `/pdf` com `codigoSolicitacao`
2. Recebe PDF em Base64
3. Converte para arquivo binário
4. Armazena o Base64 no Google Sheets (opcional: upload para Supabase)

**Código de conversão:**
```javascript
// Edit Fields11: prepara dados
{
  pdf: base64String,
  fileName: codigoSolicitacao + ".pdf",
  mimeType: "application/pdf"
}

// Convert to File: transforma em binário
binaryData = convertBase64ToBinary(pdf)
```

### 6. Template de Email Profissional

Email HTML responsivo com todas as informações:

**Estrutura do template:**
- Header com logo/imagem
- Saudação personalizada
- Tabela com dados do boleto (valor, vencimento, referência)
- Informação sobre anexo
- Seção de confirmação de recebimento
- Seção de suporte
- Assinatura
- Footer

**Personalização automática:**
```javascript
Subject: BOLETO [NOME_MÊS] [ANO]
Para: cliente@email.com
CC: copia@email.com (se configurado)
Anexo: PDF do boleto
```

### 7. Sistema de Callbacks do Banco

O Banco Inter envia callbacks quando há mudanças:

**Tipos de eventos:**
- **RECEBIDO** (PAGO): Cliente pagou
- **ATRASADO**: Passou da data de vencimento
- **CANCELADO**: Boleto foi cancelado
- **EXPIRADO**: Boleto expirou

**Processamento:**
```javascript
switch (situacao) {
  case 'RECEBIDO':
    status = 'PAGO'
    mesProxCobrança = mesAtual + 1
    clientStatus = 'OK'
    break
    
  case 'ATRASADO':
    status = 'ATRASADO'
    mesProxCobrança = mesAtual // Não avança
    clientStatus = 'PENDENCIA'
    break
    
  case 'CANCELADO':
  case 'EXPIRADO':
    status = 'CANCELADO'
    mesProxCobrança = mesAtual + 1
    clientStatus = 'OK'
    break
}
```

### 8. Rate Limiting Inteligente

A API do Banco Inter tem limites de requisições. Implementei delays:

**Estratégia:**
```javascript
rateLimit = 6 segundos // Configurável

// Entre cada requisição à API
await wait(rateLimit)

// Entre cada cliente processado
await wait(rateLimit)
```

**Por que 6 segundos:**
- API do Banco Inter permite ~10 req/min
- 6 segundos = segurança para não bater no limite
- Processamento controlado e estável

## 📊 Estrutura do Google Sheets

### Aba "Clientes"

| Coluna | Descrição |
|--------|-----------|
| email | Email do cliente |
| cpfCnpj | CPF ou CNPJ |
| tipoPessoa | FISICA ou JURIDICA |
| nome | Nome completo/razão social |
| endereco | Endereço completo |
| cidade | Cidade |
| uf | Estado (sigla) |
| cep | CEP |
| bairro | Bairro |
| valor | Valor da cobrança mensal |
| diaVencimento | Dia do vencimento (1-31) |
| mesProxCobrança | Próximo mês a cobrar (1-12) |
| status | Status atual (OK, VERIFICANDO, PENDENCIA) |
| CC | Emails em cópia (separados por vírgula) |

### Aba "Emissões"

| Coluna | Descrição |
|--------|-----------|
| id | ID único da cobrança |
| numero | Número de referência |
| nome | Nome do cliente |
| cpfCnpj | CPF/CNPJ |
| valor | Valor do boleto |
| vencimento | Data de vencimento (fórmula) |
| dataEnvio | Data programada envio (fórmula) |
| codigoSolicitacao | Código retornado pelo banco |
| boleto | PDF em Base64 |
| status | AGUARDANDO, PENDENTE, PAGO, ATRASADO, ERRO |
| mensagem | Mensagem de erro/sucesso |

### Aba "Configuração"

| Coluna | Descrição |
|--------|-----------|
| dataAtual | Data atual (fórmula: TODAY()) |
| competencia | Mês/ano da competência |
| numDiasAntecedencia | Dias antes para enviar |
| numDiasAgenda | Dias após vencimento |
| multa | % de multa |
| mora | % de mora mensal |

## 🔄 Ciclo Completo de uma Cobrança

**Dia -7 (envio programado):**
1. Workflow roda às 09:05
2. Verifica que é data de envio
3. Autentica no Banco Inter
4. Gera boleto via API
5. Busca PDF
6. Envia email com boleto
7. Status: AGUARDANDO → PENDENTE

**Dia 0 (vencimento):**
- Boleto disponível para pagamento
- Status permanece: PENDENTE

**Cliente paga:**
1. Banco Inter envia callback
2. Webhook recebe: situacao = "RECEBIDO"
3. Atualiza status: PAGO
4. Avança mês: mesProxCobrança + 1
5. Status do cliente: OK

**Cliente não paga (após vencimento):**
1. Banco Inter envia callback
2. Webhook recebe: situacao = "ATRASADO"
3. Atualiza status: ATRASADO
4. Não avança mês
5. Status do cliente: PENDENCIA

## 🛡️ Tratamento de Erros

### Autenticação:
```javascript
// Se falhar autenticação
onError: continueErrorOutput
// Retorna: sucesso = false, mensagem = "Falha na autenticação"
```

### Geração de Boleto:
```javascript
// Se falhar geração
retryOnFail: true
waitBetweenTries: 5000 // 5 segundos
// Tenta 3x antes de desistir
```

### Envio de Email:
```javascript
// Se falhar envio
onError: continueErrorOutput
// Marca status: ERRO
// Mensagem: "Erro ao enviar o boleto por e-mail"
```

### Fluxo com Erro:
```
Erro detectado
  ↓
Status marcado: ERRO
  ↓
Cliente status: PENDENCIA
  ↓
Próxima execução: tenta novamente
```

## 🎓 Conceitos Técnicos Aplicados

### Autenticação OAuth2:
- Client Credentials Grant
- Token caching com TTL
- Refresh automático

### API REST Integration:
- POST para criar recursos
- GET para buscar dados
- Headers de autenticação
- Certificados SSL

### Webhook Handling:
- Recebimento de callbacks
- Validação de payloads
- Processamento assíncrono

### Batch Processing:
- Loop controlado por cliente
- Rate limiting entre requisições
- Agregação de resultados

### Template Engine:
- HTML dinâmico com interpolação
- Formatação de dados (moeda, data)
- Condicional rendering

## 💻 Configuração Necessária

### Credenciais do Banco Inter:

```javascript
{
  baseUrl: "https://cdpj.partners.bancointer.com.br",
  clientId: "SEU_CLIENT_ID",
  clientSecret: "SEU_CLIENT_SECRET",
  contaCorrente: "SUA_CONTA",
  rateLimit: 6
}
```

### Certificado SSL:
- Obrigatório para API do Banco Inter
- Arquivo .crt e .key fornecidos pelo banco
- Configurar no N8N em SSL Certificates

### Redis:
- Host e porta
- Usado para cache de token

### Google Sheets:
- OAuth2 configurado
- Permissões de leitura/escrita

### Gmail:
- OAuth2 configurado
- Permissões de envio

## 🚀 Como Usar

### Configuração Inicial:

1. **Configure as credenciais** no node "Set Credenciais"
2. **Ajuste a planilha** com estrutura descrita
3. **Configure webhook** do Banco Inter para apontar para sua URL
4. **Teste manualmente** antes de ativar o schedule

### Adicionando Novo Cliente:

1. Adicione linha na aba "Clientes"
2. Preencha todos os campos obrigatórios
3. Defina `diaVencimento` e `mesProxCobrança`
4. Status inicial: deixe em branco

### Monitoramento:

- Aba "Emissões": ver status de cada boleto
- Aba "Clientes": ver status de cada cliente
- Logs do N8N: detalhes de execução

## ⚠️ Limitações e Considerações

### Limitações da API:
- ~10 requisições por minuto
- Certificado SSL obrigatório
- Boletos só podem ser gerados X dias antes do vencimento
- PDF disponível apenas após geração bem-sucedida

### Boas Práticas:
- Não processar muitos clientes de uma vez
- Monitorar logs regularmente
- Manter backup da planilha
- Testar mudanças em ambiente separado
- Verificar status dos emails enviados

### Custos:
- API do Banco Inter (consulte tabela do banco)
- Créditos de envio de email (Gmail API)
- Infraestrutura N8N
- Armazenamento (se usar Supabase)

## 📚 Aprendizados

### O que funcionou bem:
- Cache de token reduziu latência significativamente
- Delays entre requisições preveniram bloqueios
- Callbacks garantem sincronização em tempo real
- Template de email profissional aumentou confiança
- Sistema de dupla verificação evitou duplicatas

### Desafios que enfrentei:
- Configuração correta dos certificados SSL
- Entender fluxo OAuth2 do Banco Inter
- Sincronizar status entre múltiplas planilhas
- Lidar com diferentes cenários de erro
- Garantir idempotência nas operações

### Melhorias que planejo:
- Dashboard de métricas em tempo real
- Notificações via Slack para erros críticos
- Retry inteligente baseado em tipo de erro
- Histórico de todas as tentativas
- Relatórios mensais automatizados

---

## 📄 Notas Importantes

> ⚠️ **Confidencialidade:** O código-fonte (JSON) e credenciais não estão disponíveis publicamente.

> 🏦 **Homologação:** Certifique-se de testar em ambiente de homologação do banco antes de produção.

> 💼 **Compliance:** Este workflow lida com dados financeiros sensíveis. Garanta conformidade com LGPD e regulações bancárias.

> 🔒 **Segurança:** Nunca commite credenciais no Git. Use variáveis de ambiente ou cofre de senhas.

---

**Criado em:** 2025  
**Última atualização:** 13/10/2025  
**Status:** ✅ Em produção  
**Integração:** Banco Inter API v3
