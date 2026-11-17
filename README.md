# 🏠 Automação de Atendimento Imobiliário com n8n + IA

> Agente de IA que lê e-mails de clientes interessados em imóveis, consulta um catálogo no Google Sheets, responde automaticamente em HTML e registra os leads em uma planilha, tudo sem intervenção humana.

![n8n](https://img.shields.io/badge/n8n-workflow-EA4B71?logo=n8n&logoColor=white)
![Google Gemini](https://img.shields.io/badge/LLM-Google%20Gemini-4285F4?logo=googlegemini&logoColor=white)
![Gmail](https://img.shields.io/badge/Gmail-trigger%20%2B%20reply-EA4335?logo=gmail&logoColor=white)
![Google Sheets](https://img.shields.io/badge/Google%20Sheets-cat%C3%A1logo%20%2B%20leads-34A853?logo=googlesheets&logoColor=white)

---

## 📌 Sumário

- [Visão geral](#-visão-geral)
- [Visão do workflow completo](#-visão-do-workflow-completo)
- [Arquitetura](#-arquitetura)
- [Etapas do workflow](#-etapas-do-workflow)
- [Planilhas utilizadas](#-planilhas-utilizadas)
- [Como o prompt do agente foi pensado](#-como-o-prompt-do-agente-foi-pensado)
- [Exemplos de execução](#-exemplos-de-execução)
- [Como rodar o projeto](#-como-rodar-o-projeto)
- [Limitações e próximos passos](#-limitações-e-próximos-passos)
- [Tecnologias](#-tecnologias)
- [Autora](#-autora)

---

## 🎯 Visão geral

Imobiliárias recebem muitos e-mails repetitivos: "tem apartamento de 2 quartos na Zona Sul?", "aceita pet?", "que documentos preciso para alugar?". Responder tudo manualmente é lento, e leads quentes podem esfriar enquanto esperam.

Este projeto simula esse cenário para uma imobiliária fictícia, a **Nova Lar Imóveis** (São Paulo), e automatiza o fluxo inteiro:

1. **Recebe** o e-mail do cliente (Gmail Trigger).
2. **Interpreta** o pedido com um agente de IA (Ted, o assistente virtual da Nova Lar).
3. **Consulta** o catálogo de imóveis em tempo real (Google Sheets).
4. **Responde** na mesma conversa, com recomendações formatadas em HTML.
5. **Registra ou atualiza** o lead em uma planilha de CRM, sem duplicar clientes.

### O que o agente sabe fazer

| Capacidade | Como funciona |
|---|---|
| Recomendar imóveis | Consulta a planilha de catálogo e filtra por modalidade, disponibilidade, pet friendly e mobiliado |
| Explicar processo de compra/aluguel | Etapas e prazos médios definidos no prompt |
| Informar documentação | Lista de documentos para compra e para aluguel |
| Responder dúvidas frequentes | FGTS, negociação de preço, taxa de corretagem, pets |
| Captar leads | Registra interesse real na planilha de leads |
| Encaminhar para visita | Convida o cliente a agendar via WhatsApp |

---

## 🗺️ Visão do workflow completo

<!-- 📸 PRINT: tela inteira do canvas do n8n mostrando todos os nós conectados -->

![Workflow completo no n8n](images/workflow.png)


---

## 🧩 Arquitetura

```mermaid
flowchart LR
    A[📧 Gmail Trigger<br/>a cada 1 min] --> B[🤖 AI Agent<br/>Ted - Nova Lar]
    M[🧠 Simple Memory<br/>por threadId] -.-> B
    L[✨ Google Gemini<br/>Chat Model] -.-> B
    B <-.->|consulta| C[🔧 buscar_imoveis<br/>Google Sheets]
    B <-.->|registra/atualiza| D[🔧 registrar_lead<br/>Google Sheets]
    B --> E[📤 Gmail: Reply to a message]
```

**Linha contínua** = fluxo principal de execução. **Linha pontilhada** = sub-nós conectados ao agente (modelo, memória e ferramentas), que ele aciona quando precisa.

---

## 🔍 Etapas do workflow

### 1️⃣ Gmail Trigger: captura do e-mail

Monitora a caixa de entrada com *polling* a cada minuto. O modo `simple` está desligado, então o n8n entrega a mensagem completa (corpo, cabeçalhos, `threadId`), e esses dados são usados nas etapas seguintes.

| Configuração | Valor |
|---|---|
| Frequência | A cada minuto |
| Modo simplificado | Desativado (retorna cabeçalhos e corpo completos) |
| Filtros | Nenhum |

<!-- 📸 PRINT: configuração do nó Gmail Trigger (aba Parameters) -->
![Gmail Trigger - configuração](images/receber-email.png)

<!-- 📸 PRINT: saída (output) do nó após receber um e-mail de teste -->
![Gmail Trigger - saída](images/responder%20mensagem.png)

---

### 2️⃣ AI Agent: o cérebro do fluxo

O agente recebe o texto do e-mail (`{{ $json.text }}`) e segue um **system prompt** detalhado que define persona, tom de voz, regras de negócio, quando usar cada ferramenta e o formato da resposta (HTML).

Ele é composto por quatro sub-nós:

| Sub-nó | Tipo | Papel |
|---|---|---|
| **Google Gemini Chat Model** | Modelo de linguagem | Gera as respostas e decide quais ferramentas chamar |
| **Simple Memory** | Memória de janela | Guarda as últimas 30 mensagens, separadas por `threadId` (cada conversa de e-mail tem sua própria memória) |
| **buscar_imoveis** | Ferramenta (Google Sheets) | Lê o catálogo de imóveis |
| **registrar_lead** | Ferramenta (Google Sheets) | Cria ou atualiza leads |

<!-- 📸 PRINT: aba Parameters do AI Agent, mostrando o System Message -->
![AI Agent - system prompt](images/agente-ia.png)

---

### 3️⃣ Google Gemini Chat Model

Modelo de linguagem que alimenta o agente. É ele quem interpreta o e-mail, decide se precisa consultar o catálogo ou registrar um lead e redige a resposta final.

<!-- 📸 PRINT: configuração do nó Google Gemini Chat Model -->
![Gemini - configuração](images/chat-gemini.png)

---

### 4️⃣ Simple Memory: contexto por conversa

Usa o `threadId` do Gmail como chave de sessão. Assim, se o cliente responde o mesmo e-mail dias depois ("gostei do segundo, tem como visitar?"), o agente lembra o que já foi conversado naquela thread, sem misturar com outros clientes.

| Configuração | Valor |
|---|---|
| Tipo de sessão | Chave customizada |
| Session Key | `{{ $json.threadId }}` |
| Janela de contexto | 30 mensagens |

<!-- 📸 PRINT: configuração do nó Simple Memory -->
![Simple Memory - configuração](images/memoria.png)

---

### 5️⃣ buscar_imoveis: catálogo no Google Sheets

Ferramenta que dá ao agente acesso de leitura à planilha de imóveis. O prompt obriga o agente a **sempre** consultá-la antes de recomendar algo, evitando que ele invente imóveis.

<!-- 📸 PRINT: configuração do nó buscar_imoveis -->
![buscar_imoveis - configuração](images/buscar-imoveis.png)

<!-- 📸 PRINT: a planilha de catálogo aberta no Google Sheets -->
![Planilha de catálogo](images/planilha-catalogo.png)
---

### 6️⃣ registrar_lead: CRM simples no Google Sheets

Ferramenta com operação **Append or Update**, usando o **E-mail como chave de correspondência**. Na prática:

- Cliente **novo** → cria uma nova linha com status "Novo lead".
- Cliente **recorrente** → atualiza a linha existente (novo imóvel de interesse, orçamento, próximo passo), sem duplicar.

Os campos são preenchidos pelo próprio agente a partir da conversa (via `$fromAI`), exceto o e-mail, que vem direto do cabeçalho da mensagem.

<!-- 📸 PRINT: configuração do nó registrar_lead (mapeamento de colunas) -->
![registrar_lead - configuração parte 1](images/registrar-lead1.png)
![registrar_lead - configuração parte 2](images/registrar-lead2.png)

<!-- 📸 PRINT: a planilha de leads com algumas linhas preenchidas -->
![Planilha de leads](images/planilha-leads.png)

---

### 7️⃣ Reply to a message: resposta na mesma thread

Envia a resposta gerada pelo agente (`{{ $json.output }}`) como **reply** ao e-mail original, mantendo a conversa encadeada na caixa do cliente.

<!-- 📸 PRINT: configuração do nó Reply to a message -->
![Reply to a message - configuração](images/responder%20mensagem.png)

---

## 📊 Planilhas utilizadas

### Catálogo de imóveis (`buscar_imoveis`)

| Coluna | Descrição |
|---|---|
| ID | Identificador único do imóvel |
| Tipo | Apartamento, casa, sala comercial, studio |
| Modalidade | Venda ou Aluguel |
| Bairro | Localização |
| Metragem (m²) | Área do imóvel |
| Quartos / Suítes / Vagas | Características |
| Preço (R$) | Valor de venda ou aluguel |
| Condomínio (R$/mês) | Taxa condominial |
| Destaques | Diferenciais do imóvel |
| Pet Friendly | Sim / Não |
| Mobiliado | Sim / Parcial / Não |
| Disponível | Sim / Não (indisponíveis nunca são recomendados) |

### Leads (`registrar_lead`)

| Coluna | Preenchida por |
|---|---|
| Data | Agente (data atual) |
| Nome | Agente (extraído do e-mail) |
| E-mail | Cabeçalho do Gmail (chave de busca) |
| Telefone | Agente ("Não informado" se ausente) |
| Interesse (Compra/Aluguel) | Agente |
| Região de Preferência | Agente |
| Quartos | Agente |
| Orçamento Máximo (R$) | Agente |
| Imóvel de Interesse (ID) | Agente |
| Resumo da Conversa | Agente (1-2 frases) |
| Status | Agente ("Novo lead" ou atualizado) |
| Próximo Passo | Agente (ex.: "Agendar visita") |

---

## 🧠 Como o prompt do agente foi pensado

O system prompt é organizado em blocos, para deixar o comportamento previsível:

| Bloco | Objetivo |
|---|---|
| **Quem você é** | Persona (Ted, assistente da Nova Lar) e contexto do negócio |
| **Tom de voz** | Profissional e acolhedor, direto ao ponto, tratamento por "você" |
| **Informações da empresa** | Regiões, faixas de preço, horários, contatos |
| **Ferramentas** | Quando e como usar `buscar_imoveis` e `registrar_lead` |
| **Regras de leitura da planilha** | Filtrar por modalidade, disponibilidade, pet e mobiliado |
| **Documentação e processos** | Respostas padronizadas para compra e aluguel |
| **FAQ** | Respostas para dúvidas recorrentes |
| **Regras importantes** | Limites: máximo de 3 imóveis por resposta, nunca inventar, recusar assuntos fora do escopo |
| **Formatação** | Resposta sempre em HTML válido, com blocos visuais separados |

### Decisões de design que valem destacar

- **"Nunca responda de memória"**: força a consulta à planilha, evitando alucinação de imóveis.
- **Verificar antes de registrar o lead**: evita linhas duplicadas para o mesmo cliente.
- **Não registrar perguntas genéricas**: mantém a planilha de leads limpa, só com interesse real.
- **Máximo de 3 opções por resposta**: e-mail objetivo, sem sobrecarregar o cliente.
- **Resposta em HTML**: o Gmail renderiza o HTML, permitindo negrito, listas e links clicáveis.

---

## ▶️ Exemplos de execução

### Cenário 1: cliente procurando imóvel para alugar

**E-mail enviado pelo cliente:**
<!-- 📸 PRINT: e--mail enviado pelo cliente e resposta da agente -->
![Cenário 1 - e-mail recebido](images/teste1.png)
![Cenário 1 - agente consultando o catálogo](images/teste2.png)

<!-- 📸 PRINT: Mesmo cliente responde na mesma thread, mudando o orçamento. O agente usa a memória da conversa e **atualiza** a linha existente, sem criar uma nova. -->
![Cenário 1 - resposta enviada](images/teste3.png)

<!-- 📸 PRINT: resposta do Ted recebida pelo cliente -->
![Cenário 1 - resposta enviada](images/teste4.png)
![Cenário 1 - resposta enviada](images/teste5.png)


<!-- 📸 PRINT: linha criada na planilha de leads -->
![Cenário 1 - lead registrado](images/lead.png)
---

### Cenário 2: cliente recorrente (atualização de lead)

Mesmo cliente responde na mesma thread, mudando o orçamento. O agente usa a memória da conversa e **atualiza** a linha existente, sem criar uma nova.

```
Gostei da segunda opção! Posso subir um pouco o orçamento, até R$ 4.500.
Como funciona para agendar uma visita?
```

<!-- 📸 PRINT: segundo e-mail do cliente na mesma thread -->
![Cenário 2 - e-mail de follow-up](docs/images/ex2-01-email-followup.png)

<!-- 📸 PRINT: resposta do Ted sugerindo agendamento via WhatsApp -->
![Cenário 2 - resposta do agente](docs/images/ex2-02-resposta-ted.png)

<!-- 📸 PRINT: planilha de leads mostrando a MESMA linha atualizada (orçamento e próximo passo) -->
![Cenário 2 - lead atualizado](docs/images/ex2-03-lead-atualizado.png)

---

### Cenário 3: pergunta genérica (sem registro de lead)

```
Assunto: Dúvida

Qual o horário de atendimento de vocês?
```

O agente responde a dúvida, mas **não** registra lead, já que não há interesse real em um imóvel.

<!-- 📸 PRINT: resposta do agente e planilha de leads sem nova linha -->
![Cenário 3 - sem registro de lead](docs/images/ex3-01-sem-lead.png)

---

### Cenário 4: pedido fora do escopo

Quando o cliente pergunta algo sem relação com imóveis, o agente redireciona para o foco do negócio.

<!-- 📸 PRINT: e-mail fora de escopo + resposta redirecionando -->
![Cenário 4 - fora do escopo](docs/images/ex4-01-fora-do-escopo.png)

---

## 🚀 Como rodar o projeto

### Pré-requisitos

- Instância do [n8n](https://n8n.io/) (cloud ou self-hosted)
- Conta Google com acesso ao Gmail e ao Google Sheets
- Chave de API do [Google Gemini](https://aistudio.google.com/)

### Passo a passo

1. **Importe o workflow**
   No n8n: *Workflows → Import from File* e selecione `Simulação - Automação Imobiliária.json`.

2. **Crie as duas planilhas no Google Sheets**
   - Uma de **catálogo de imóveis**, com as colunas descritas [acima](#catálogo-de-imóveis-buscar_imoveis).
   - Uma de **leads**, com as colunas descritas [acima](#leads-registrar_lead).

3. **Configure as credenciais no n8n**
   - Gmail OAuth2 (nós *Gmail Trigger* e *Reply to a message*)
   - Google Sheets OAuth2 (nós *buscar_imoveis* e *registrar_lead*)
   - Google Gemini (PaLM) API (nó *Google Gemini Chat Model*)

4. **Aponte os nós de planilha para as suas planilhas**
   Nos nós `buscar_imoveis` e `registrar_lead`, selecione o documento e a aba corretos.

5. **Teste**
   Envie um e-mail para a conta Gmail conectada e execute o workflow manualmente (*Test workflow*) ou ative-o (*Active*).

> 💡 Os dados da imobiliária (nome, telefone, regiões, faixas de preço) estão no **System Message** do nó AI Agent. Adapte para o seu caso.

---

## ⚠️ Limitações e próximos passos

Este projeto é uma **simulação** de portfólio. Alguns pontos para evoluir antes de usar em produção:

- [ ] **Filtro no Gmail Trigger**: hoje não há filtros, então o fluxo reage a qualquer e-mail recebido. Filtrar por rótulo, remetente ou assunto evitaria respostas a newsletters e spam.
- [ ] **Revisão humana**: incluir uma etapa de aprovação (ou rascunho em vez de envio direto) para casos sensíveis.
- [ ] **Tratamento de erros**: adicionar um fluxo de erro (*Error Trigger*) para avisar quando uma execução falhar.
- [ ] **Campo E-mail do lead**: o valor vem do cabeçalho `from`, que pode incluir o nome junto com o endereço. Normalizar para guardar só o endereço deixaria a chave de busca mais confiável.
- [ ] **Agendamento automático**: integrar com Google Calendar para marcar visitas direto pelo e-mail.
- [ ] **Notificação para corretores**: avisar um humano (Slack/WhatsApp) quando um lead quente for registrado.
- [ ] **Escalonamento para humano**: detectar reclamações ou casos complexos e encaminhar para atendimento humano.

---

## 🛠️ Tecnologias

- **[n8n](https://n8n.io/)**: orquestração do workflow
- **Google Gemini**: modelo de linguagem do agente
- **LangChain (nós nativos do n8n)**: AI Agent, memória e ferramentas
- **Gmail API**: gatilho e envio de respostas
- **Google Sheets API**: catálogo de imóveis e base de leads

---

## 👩‍💻 Autora

**Thaiza**
Estudante de Análise e Desenvolvimento de Sistemas · Desenvolvedora backend (Python/FastAPI)

- GitHub: [github.com/thaiza-d](https://github.com/thaiza-d)

---

<sub>Projeto fictício para fins de estudo e portfólio. "Nova Lar Imóveis", os dados do catálogo e os contatos são exemplos.</sub>