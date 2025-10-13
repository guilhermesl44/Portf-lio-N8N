# 🤖 Raspagem Inteligente de Leads com IA

> **Categoria:** Comercial | **Complexidade:** ⭐⭐⭐⭐⭐ | **Status:** ✅ Em Produção

## 📝 Visão Geral

Desenvolvi esta automação para resolver um problema que enfrentávamos na prospecção: gastar muito tempo buscando leads manualmente e, pior ainda, descobrir que muitos dos contatos encontrados tinham números inválidos ou inativos no WhatsApp.

A solução que criei usa IA (GPT-4o-mini) para fazer buscas inteligentes em duas fontes diferentes (busca web geral e Google Places), e o mais importante: **valida automaticamente se o número de WhatsApp existe** antes de salvar na planilha. Isso elimina o retrabalho de tentar contatar números que não existem.

## 🎯 Por Que Criei Esta Automação

**O problema:**
- Buscar leads manualmente era demorado
- Muitos números encontrados não tinham WhatsApp ativo
- Dados vinham desorganizados de várias fontes
- Precisávamos validar cada contato antes de passar para o comercial

**A solução:**
Um workflow que busca automaticamente, organiza os dados, valida o WhatsApp e entrega tudo pronto no Google Sheets.

## 🏗️ Como Funciona

### Fluxo do Processo

```
1. Inicio o workflow manualmente
   ↓
2. Defino os termos de pesquisa (ex: "loja de móveis", "loja de colchões")
   ↓
3. Para cada termo, a IA faz buscas em:
   - SERPER Search (busca geral na web)
   - SERPER Places (Google Maps/Places)
   ↓
4. Os resultados das duas fontes são unidos
   ↓
5. Remove leads que não têm telefone
   ↓
6. Para cada lead, valida se o WhatsApp existe
   ↓
7. Se o WhatsApp existir, salva no Google Sheets
   ↓
8. Repete para o próximo lead
```

### Diagrama Visual

![Fluxo Completo](./Imagens/Captura de tela 2025-10-13 160135.jpg)

*Nota: Por questões de confidencialidade, o arquivo JSON não está disponível publicamente.*

## ⚙️ Componentes e Integrações

### Tecnologias Utilizadas:
- **N8N:** Plataforma de automação
- **OpenAI GPT-4o-mini:** IA para busca inteligente de leads
- **SERPER API:** Para fazer buscas no Google (Search + Places)
- **Evolution API:** Para validar números de WhatsApp
- **Google Sheets:** Para armazenar os leads validados

### Nodes do Workflow:
- **Manual Trigger:** Inicio manualmente quando preciso de leads
- **Edit Fields:** Configuro os parâmetros (termos, cidade, quantidade)
- **AI Agent:** Dois agentes de IA fazendo buscas paralelas
- **Split Out:** Separa arrays para processar individualmente
- **Loop Over Items:** Loop pelos termos de pesquisa
- **Merge:** Une resultados das duas fontes de busca
- **Filter:** Remove leads sem telefone
- **Split In Batches:** Processa leads um por um
- **Evolution API:** Valida se o WhatsApp existe
- **IF:** Só salva se o WhatsApp for válido
- **Google Sheets:** Salva o lead validado
- **Wait:** Delays para não sobrecarregar APIs

## 🔧 Funcionalidades Principais

### 1. Busca Inteligente com IA

Criei dois agentes de IA que trabalham em paralelo:

**Agente 1 - SERPER Search:**
- Busca informações gerais na web
- Procura em sites, diretórios, redes sociais
- A IA decide quais queries fazer baseada no termo fornecido

**Agente 2 - SERPER Places:**
- Busca estabelecimentos no Google Maps
- Foca em dados locais e empresas físicas
- Complementa a busca geral

**Por que dois agentes?**
Porque percebi que cada fonte traz tipos diferentes de leads. A busca web encontra empresas com presença online, enquanto o Places encontra negócios locais que podem não ter site.

### 2. Instruções para a IA

Dei instruções específicas para os agentes de IA:

- Buscar: nome, WhatsApp, e-mail, site, Instagram
- **Telefone é obrigatório** - se não encontrar, não incluir na lista
- Formatar números no padrão E.164 (ex: 5518997324996)
- Usar o formato correto de localização: "Cidade, State of Estado, Brazil"
- Fazer quantas buscas precisar para atingir o objetivo

### 3. Validação de WhatsApp

Antes de salvar qualquer lead, o workflow:
1. Envia o número para a Evolution API
2. Verifica se o WhatsApp existe (`exists: true`)
3. Só prossegue se o número for válido
4. Se for inválido, volta pro loop e tenta o próximo

**Por que isso é importante?**
Evita que a equipe comercial perca tempo tentando contatar números que não existem.

### 4. Formato de Saída Estruturado

Configurei a IA para retornar os dados sempre neste formato:

```json
{
  "leads": [
    {
      "name": "Nome da Empresa",
      "email": "contato@empresa.com",
      "Telefone": "5511999999999",
      "Site": "https://site.com",
      "Instagram": "@empresa",
      "Cidade": "Osasco"
    }
  ]
}
```

Isso garante consistência e facilita o salvamento no Google Sheets.

## 📋 Parâmetros Configuráveis

Quando inicio o workflow, defino:

**Termos de Pesquisa:**
```json
[
  "loja de móveis",
  "loja de colchões", 
  "loja de iluminação",
  "loja de móveis de alto padrão"
]
```

**Outros Parâmetros:**
- **Número de Leads:** Quantos leads quero por termo (ex: 200)
- **Cidade:** Onde buscar (ex: "Osasco")

O workflow processa cada termo individualmente em loop.

## 🔄 Processamento em Lotes

Para não sobrecarregar as APIs e evitar bloqueios, estruturei o workflow com:

### Loop Externo (Termos de Pesquisa):
- Processa um termo por vez
- Evita fazer muitas requisições simultâneas

### Loop Interno (Leads Individuais):
- Valida WhatsApp de um lead por vez
- **Delay de 25 segundos** entre validações de WhatsApp
- **Delay de 50 segundos** após salvar no Google Sheets

**Por que os delays?**
- Evita rate limiting das APIs
- Previne bloqueios por uso excessivo
- Simula comportamento humano

## 🛡️ Tratamento de Erros

### Evolution API (Validação WhatsApp):
- Configurada com `alwaysOutputData: true`
- `onError: continueErrorOutput`
- Se der erro na validação, continua para o próximo lead

### AI Agents:
- `retryOnFail: false` 
- Se a IA falhar, não trava o workflow inteiro

### Filtros de Segurança:
- Remove leads sem telefone antes de validar
- Só salva no Sheets se WhatsApp existir
- Case sensitive nas validações

## 💾 Salvamento no Google Sheets

Configurei o Google Sheets para salvar na aba "Análise" com estas colunas:

| Coluna | Dado |
|--------|------|
| Nome | Nome da empresa/contato |
| E-mail | Email corporativo |
| Site | Website |
| Instagram | Perfil do Instagram |
| Telefone | Número validado (formato E.164) |
| Cidade | Localização |

O salvamento é por **append** (adiciona ao final), não sobrescreve dados existentes.

## 🔒 Segurança e Credenciais

### Credenciais Configuradas:
- **OpenAI API:** Para os agentes de IA
- **SERPER Search e Places:** Chaves diferentes para cada serviço
- **Evolution API:** Para validação de WhatsApp
- **Google Sheets OAuth2:** Para salvar dados

Todas armazenadas de forma criptografada no N8N.

## 🎓 Conceitos Técnicos Aplicados

### AI Agent Pattern:
- Uso de ferramentas (tools) para a IA chamar APIs
- System prompts customizados para cada agente
- Structured output para garantir formato consistente

### Processamento Paralelo:
- Dois agentes rodando simultaneamente
- Merge dos resultados depois

### Validação em Tempo Real:
- Integração com Evolution API
- Decisão condicional (IF) baseada no retorno

### ETL (Extract, Transform, Load):
- Extract: Busca nas APIs via IA
- Transform: Formatação e validação
- Load: Salvamento no Google Sheets

## 🔍 Detalhes Técnicos Importantes

### Formato de Localização:
A API SERPER exige o formato específico:
```
"Osasco, State of Sao Paulo, Brazil"
"Londrina, State of Parana, Brazil"
```

### Formato de Telefone E.164:
Todos os números devem estar no formato internacional sem formatação:
```
Correto: 5518997324996
Errado: +55 (18) 99732-4996
```

### Estrutura do Node Loop:
```javascript
batchSize: 1  // Processa 1 item por vez
reset: {{ $('Loop 1').context.done }}  // Reseta quando terminar
```

## 🚀 Como Usar

1. **Configure as credenciais** necessárias
2. **Ajuste os parâmetros** no node "Edit Fields3":
   - Termos de pesquisa (array)
3. **Ajuste nos nodes "Edit Fields"** e "Edit Fields1":
   - Número de leads desejado
   - Cidade alvo
4. **Execute o workflow** clicando em "Test workflow"
5. **Aguarde o processamento** (pode levar alguns minutos)
6. **Confira os resultados** no Google Sheets

## ⚠️ Limitações e Considerações

### Limitações Conhecidas:
- Dependente da qualidade das buscas do Google
- Nem sempre encontra todos os dados de contato
- APIs podem ter rate limits
- Validação de WhatsApp consome créditos Evolution API
- Processo pode ser demorado com muitos leads

### Recomendações:
- Comece com poucos termos para testar
- Monitore os créditos das APIs
- Revise os leads salvos periodicamente
- Ajuste os delays conforme necessário

## 📚 Aprendizados

### O que funcionou bem:
- Busca paralela em duas fontes aumenta cobertura
- Validação de WhatsApp elimina retrabalho
- IA consegue formatar dados de forma consistente
- Delays previnem bloqueios

### Desafios que enfrentei:
- Configurar prompts corretos para a IA
- Definir delays adequados entre requisições
- Garantir formato consistente dos dados
- Lidar com erros de API sem travar tudo

### Melhorias que planejo:
- Adicionar mais fontes de busca
- Implementar cache para evitar buscas duplicadas
- Criar dashboard de métricas
- Adicionar enriquecimento de dados

---

## 📄 Notas Importantes

> ⚠️ **Confidencialidade:** O código-fonte (JSON) não está disponível publicamente por questões de propriedade intelectual.

> 🔐 **Privacidade:** Esta automação coleta apenas dados públicos disponíveis na web.

> 💼 **Interesse em projeto similar?** Entre em contato para discutirmos a implementação.

---

**Criado em:** 2025  
**Última atualização:** 13/10/2025  
**Status:** ✅ Funcionando em produção
