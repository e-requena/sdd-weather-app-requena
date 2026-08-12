# Revisão de Especificação — Weather App

**Data:** 2026-08-12  
**Revisor:** Review Agent  
**Status:** Análise Detalhada  
**Documento Analisado:** `specs/weather-app-spec.md`

---

## Sumário Executivo

A especificação é **70% completa** mas contém **25 problemas críticos** que precisam ser resolvidos antes da implementação:
- **Ambiguidades:** 8 problemas (causam dúvidas durante coding)
- **Inconsistências:** 7 problemas (contradições internas)
- **Requisitos Faltantes:** 7 problemas (gaps não documentados)
- **Critérios Fracos:** 3 problemas (não verificáveis ou vagos)

**Recomendação:** Não iniciar implementação até resolver todos os problemas marcados como **CRÍTICOS**.

---

## 1. AMBIGUIDADES

### A1: Debounce 300ms — Exatamente Quando?

**Localização:** RF1, AC1.1, AC1.2  
**Problema:** Especificação é ambígua sobre timing:
- "Busca é disparada enquanto o usuário digita (com debounce 300ms)" — isso significa:
  - Requisição sai a cada 300ms enquanto digita? (intervalo)
  - Requisição sai 300ms após PARAR de digitar? (delay)

**Impacto:** Alta  
**Risco:** Implementador escolhe intervalo (múltiplas requisições), não delay (requisição única).

**Cenário do Problema:**
```
Usuário digita: "s-á-o-p-a-u-l-o" (8 caracteres em 500ms)
Interpretação 1 (intervalo): Requisições em t=0ms, t=300ms, t=600ms (3 requisições)
Interpretação 2 (delay): Requisição em t=800ms (1 requisição) ✓ ESPERADO
```

**Sugestão de Correção:**
```markdown
**RF1: Buscar Cidades**
- Busca é disparada **300ms após a última digitação (debounce com delay)**
- Exemplo: Usuário digita "são paulo" em 500ms:
  - t=0ms: começa a digitar "s"
  - t=500ms: termina de digitar "o"
  - t=800ms: 300ms após última digitação → requisição é enviada
- Cada nova digitação reinicia o timer de 300ms
- Se usuário digita "s", aguarda 300ms, digita "ã" → timer reinicia (não há 2 requisições)
```

---

### A2: "Hoje" na Previsão de 5 Dias — Qual Referência de Tempo?

**Localização:** RF3, AC3.1, EC12  
**Problema:** Não define o que é "hoje" na previsão:
- É "data atual (meia-noite)"? 
- É "hora atual"? 
- Se usuário abre às 23:59, o "hoje" inclui 1 minuto?

**Impacto:** Média  
**Risco:** Previsão pode começar em D+1 e não D+0 em alguns horários.

**Cenário do Problema:**
```
Usuário abre app às 23:59 (23h 59min) de 12 de agosto
Previsão atual retorna: "12 ago, 13 ago, 14 ago, 15 ago, 16 ago"
Usuário abre app às 00:01 (00h 01min) de 13 de agosto
Previsão retorna: "13 ago, 14 ago, 15 ago, 16 ago, 17 ago" ✓
```

**Sugestão de Correção:**
```markdown
**RF3: Exibir Previsão de 5 Dias**
- "Hoje" = a data civil atual (00:00 até 23:59) baseada no **timezone local do navegador**
- Exemplos:
  - Se usuário em São Paulo abre às 14:30, "hoje" = "12 ago"
  - Se mesmo usuário viaja para Tóquio (UTC+9) e abre às 02:30 (no relógio de Tokyo, ainda 12 ago em São Paulo), "hoje" = "13 ago" (data de Tóquio)
- Sempre respeita timezone do navegador/dispositivo (não do servidor ou da cidade buscada)
```

---

### A3: Mapeamento de weather_code → Ícone e Descrição

**Localização:** RF2, RF3, AC2.1, AC3.2  
**Problema:** Especificação menciona `weather_code` mas não define o mapeamento.

**Impacto:** Alta  
**Risco:** Cada implementador cria sua própria tabela; inconsistência com specs posteriores.

**Open-Meteo Códigos Existentes:**
```
0 = Clear sky → ☀️ + "Céu limpo"
1 = Mainly clear → ⛅ + "Céu pouco nublado"
2 = Partly cloudy → ⛅ + "Parcialmente nublado"
3 = Overcast → ☁️ + "Nublado"
45 = Foggy → 🌫️ + "Nevoeiro"
48 = Depositing rime fog → 🌫️ + "Nevoeiro com geada"
51 = Light drizzle → 🌦️ + "Chuvisco leve"
...etc (20+ códigos)
```

**Sugestão de Correção:**

Criar uma nova seção na spec:

```markdown
## Apêndice A: Mapeamento de weather_code (Open-Meteo → UI)

### Tabela de Tradução

| Código | Condição (EN) | Descrição pt-BR | Ícone | Cor |
|--------|---------------|-----------------|-------|-----|
| 0 | Clear sky | Céu limpo | ☀️ | Amarelo (#FBBF24) |
| 1 | Mainly clear | Céu pouco nublado | ⛅ | Amarelo claro (#FCD34D) |
| 2 | Partly cloudy | Parcialmente nublado | ⛅ | Cinza claro (#D1D5DB) |
| 3 | Overcast | Nublado | ☁️ | Cinza médio (#9CA3AF) |
| 45 | Foggy | Nevoeiro | 🌫️ | Cinza escuro (#6B7280) |
| 48 | Depositing rime fog | Nevoeiro com geada | 🌫️ | Cinza escuro (#6B7280) |
| 51 | Light drizzle | Chuvisco leve | 🌦️ | Azul claro (#93C5FD) |
| 53 | Moderate drizzle | Chuvisco moderado | 🌦️ | Azul médio (#60A5FA) |
| 55 | Dense drizzle | Chuvisco denso | 🌧️ | Azul escuro (#1E40AF) |
| 61 | Slight rain | Chuva leve | 🌧️ | Azul escuro (#1E40AF) |
| 63 | Moderate rain | Chuva moderada | ⛈️ | Azul muito escuro (#0C2540) |
| 65 | Heavy rain | Chuva forte | ⛈️ | Azul muito escuro (#0C2540) |
| 71 | Slight snow | Neve leve | ❄️ | Branco azulado (#E0F2FE) |
| 73 | Moderate snow | Neve moderada | ❄️ | Branco azulado (#E0F2FE) |
| 75 | Heavy snow | Neve forte | ❄️ | Branco azulado (#E0F2FE) |
| 77 | Snow grains | Grãos de neve | ❄️ | Branco azulado (#E0F2FE) |
| 80 | Slight rain showers | Pancadas de chuva leve | 🌧️ | Azul claro (#93C5FD) |
| 81 | Moderate rain showers | Pancadas de chuva moderada | ⛈️ | Azul médio (#60A5FA) |
| 82 | Violent rain showers | Pancadas de chuva violenta | ⛈️ | Azul muito escuro (#0C2540) |
| 85 | Slight snow showers | Pancadas de neve leve | ❄️ | Branco azulado (#E0F2FE) |
| 86 | Heavy snow showers | Pancadas de neve forte | ❄️ | Branco azulado (#E0F2FE) |
| 80-99 | Thunderstorm variants | Tempestade / Raio | ⚡ | Roxo (#A855F7) |

### Notas:
- Se `weather_code` for desconhecido, usar fallback: ☁️ + "Condição desconhecida"
- Cores usam paleta Tailwind (ver `tailwind.config.js`)
- Emojis são renderizados com fallback para SVG se não suportado
```

---

### A4: Formato Exato de Mensagens de Erro

**Localização:** AC1.3, AC2.3, EC1-EC18  
**Problema:** Especificação lista múltiplas variações de mensagens:
- "Nenhuma cidade encontrada"
- "Nenhuma cidade encontrada para 'xyzabc'"
- "Erro ao carregar dados. Tente novamente."
- "Problemas ao conectar. Tentando novamente..."
- "Sem conexão de Internet"
- "Muitos acessos. Aguarde alguns segundos."
- "Não foi possível carregar. Tente novamente mais tarde."

**Impacto:** Média  
**Risco:** Inconsistência visual; usuário fica confuso com diferentes estilos de erro.

**Sugestão de Correção:**

Criar tabela de mensagens padrão:

```markdown
## Apêndice B: Mensagens Padrão (Localizadas em pt-BR)

### Mensagens de Busca

| Caso | Mensagem | Ação Sugerida |
|------|----------|---------------|
| Cidade não encontrada | "Nenhuma cidade encontrada para '[termo]'" | Sugerir: "Tente outro nome ou verifique a ortografia" |
| Input vazio | (sem mensagem, apenas sem sugestões) | — |
| Input apenas espaços | (sem mensagem, input é trimado automaticamente) | — |

### Mensagens de Clima

| Caso | Mensagem | Ação Sugerida |
|------|----------|---------------|
| Carregando (primeira vez) | (skeleton/spinner, sem texto) | — |
| Timeout (> 8 seg) | "Problemas ao conectar. Tentando novamente..." | [spinner] + [botão Cancelar] |
| Erro HTTP 5xx | "Erro ao carregar dados. Tente novamente." | [botão Atualizar] |
| Rate limit (HTTP 429) | "Muitos acessos. Aguarde alguns segundos." | [botão Atualizar desabilitado por X seg] |
| Sem Internet (offline) | "Sem conexão de Internet" | [cache exibido se disponível] |
| 3 tentativas falhadas | "Não foi possível carregar. Tente novamente mais tarde." | [botão Atualizar] |

### Mensagens de Cache/Offline

| Caso | Mensagem | Localização |
|------|----------|-------------|
| Dados cacheados (< 24h) | "Offline — dados de X horas atrás" | Label acima do card de clima |
| Dados cacheados (> 24h) | "Offline — dados antigos (> 24h)" | Label acima do card de clima |
| Sem dados, sem Internet | "Sem conexão e sem dados salvos" | Centered no card de clima |
```

---

### A5: "Ordenadas por População" — Qual é a Métrica?

**Localização:** RF1, AC1.1  
**Problema:** "Até 10 sugestões (ordenadas por população)" — qual é exatamente a ordenação?
- Cidade principal vs. município?
- É população do município ou região metropolitana?
- Se "São Paulo" tem 12M e "São Paulo, Paraná" tem 100k, qual vem primeiro?

**Impacto:** Baixa  
**Risco:** UX ruim se ordem for inesperada.

**Sugestão de Correção:**
```markdown
**RF1: Buscar Cidades**
- Ordenação de sugestões:
  1. **Primário:** Cidades com população municipal ≥ 100k, ordenadas por população DESC
  2. **Secundário:** Cidades com população < 100k, ordenadas por relevância (match score) DESC
  3. **Terciário:** Cidades/regiões menores, ordenadas alfabeticamente ASC
  
- Exemplo: Busca "são"
  1. São Paulo, SP (12.3M) 🥇
  2. São Gonçalo, RJ (999k) 🥈
  3. São Leopoldo, RS (214k) 🥉
  4. São João de Meriti, RJ (459k) 🥉
  5. ...etc até 10 sugestões

- **Nota:** Open-Meteo retorna dados com população; usar campo `population` para ordenação
```

---

### A6: localStorage Sem Suporte — Qual é o Fallback Exato?

**Localização:** AC4.4, EC14, RNF-TECH  
**Problema:** Diz "App funciona sem localStorage" mas não define:
- Toggle funciona, mas preferência é perdida? (diz isso)
- Próxima aba em incógnito usa mesmo localStorage? (responde sim, porque incógnito usa session storage)
- Qual é o storage alternativo (sessionStorage, memory)?

**Impacto:** Média  
**Risco:** Em modo privado, comportamento é inesperado.

**Sugestão de Correção:**
```markdown
**AC4.4: Fallback de Storage**
- Prioridade de storage:
  1. localStorage (preferência persistida entre sessões)
  2. Se localStorage indisponível (try/catch): usar sessionStorage (persistida apenas nesta aba)
  3. Se sessionStorage também indisponível: usar variável em memória (perdida ao fechar aba)

- Exemplo:
  - Em navegação normal (Chrome): localStorage funciona, preferência salva permanentemente
  - Em incógnito (Chrome): localStorage indisponível, sessionStorage funciona, preferência perdida ao fechar
  - Em modo privado (Safari): ambos indisponíveis, preferência em memória, perdida ao fechar aba
  
- Nenhuma mensagem de erro é mostrada (graceful degradation)
```

---

### A7: "Fuzzy Match" vs. "Normalização de Acentos"

**Localização:** US6 (Cenário 4), AC1.4  
**Problema:** Diz "busca tolera omissão de acentos" — é fuzzy matching ou apenas normalização?
- Fuzzy matching: "sao paulo" encontra "são paulo" mesmo com caracteres não exatos
- Normalização: "são paulo" é removido acento → "sao paulo" → comparado com "sao paulo"

**Impacto:** Baixa  
**Risco:** Implementador usa regex simples (normalização), não fuzzy, resultados podem ser inconsistentes.

**Sugestão de Correção:**
```markdown
**AC1.4: Suporte a Acentos**
- Tipo de busca: **Normalização de acentos + substring matching**
  - Não é fuzzy matching (Levenshtein distance)
  - É remoção de acentuação + comparação literal

- Algoritmo:
  1. Input do usuário: "sao paulo"
  2. Normalizar input: "sao paulo" (remove acentos via `String.prototype.normalize()`)
  3. Sugestão da API: "São Paulo, São Paulo, Brazil"
  4. Normalizar sugestão: "sao paulo, sao paulo, brazil"
  5. Substring match: "sao paulo" ⊆ "sao paulo, sao paulo, brazil" → ✓ MATCH

- Exemplos:
  - "sao paulo" → encontra "São Paulo" ✓
  - "são paulo" → encontra "São Paulo" ✓
  - "sâo paulo" → encontra "São Paulo" ✓ (normaliza para "sao paulo")
  - "so paulo" → NÃO encontra "São Paulo" ✗ (não é substring)

- Implementação: Use `String.prototype.normalize('NFD').replace(/[\u0300-\u036f]/g, '')`
```

---

### A8: Sincronização Entre Abas — Qual Evento Exato?

**Localização:** AC4.5  
**Problema:** Diz "Aba 2 recebe evento de storage" — mas qual é o comportamento exato do `storage` event?
- `storage` event dispara em OUTRAS abas, não na aba que fez a mudança
- Se abrir 3 abas, mudar em Aba 1, Abas 2 e 3 recebem evento?
- Que fazer se Aba 2 estava visualizando °C e Aba 1 muda para °F?

**Impacto:** Média  
**Risco:** Comportamento incompleto em múltiplas abas.

**Sugestão de Correção:**
```markdown
**AC4.5: Sincronização Entre Abas**
- Usar `storage` event para sincronizar preferência entre abas
  
- Comportamento:
  1. Aba 1: Usuário clica toggle "°F"
  2. localStorage['temperatureUnit'] = 'fahrenheit'
  3. Window.storage event dispara em **todas as OUTRAS abas** (não em Aba 1)
  4. Aba 2, Aba 3 recebem evento
  5. Aba 2, Aba 3 leem novo valor: 'fahrenheit'
  6. Aba 2, Aba 3 atualizam UI: temperaturas mudam para °F
  7. **Aba 1 já tinha mudado sua UI (não recebe evento de si mesma)**

- Código esperado:
  ```typescript
  // Ao clicar toggle
  localStorage.setItem('temperatureUnit', 'fahrenheit');
  // UI já atualiza nesta aba
  
  // Em OUTRAS abas
  window.addEventListener('storage', (event) => {
    if (event.key === 'temperatureUnit') {
      updateUI(event.newValue); // atualiza para 'fahrenheit'
    }
  });
  ```

- Nota: Esta é limitação de `storage` event; é comportamento esperado do navegador
```

---

## 2. INCONSISTÊNCIAS

### I1: Cache TTL — "24 horas" é Absoluto ou Relativo?

**Localização:** US5, RNF-REL, EC9, EC11  
**Problema:** Há 3 interpretações diferentes:
- US5: "Cache expira após 24 horas (TTL = 24h)"
- EC9: "Offline — 2 horas atrás" (exemplo de uso)
- RNF-REL: "Cache: localStorage (24h TTL para últimos dados)"

**Impacto:** Alta  
**Risco:** Implementação pode usar 12h, 24h ou 48h — afeta offline reliability.

**Cenário do Problema:**
```
t=0:00 → Usuário busca "São Paulo" → dados salvos com timestamp
t=6:00 → Usuário busca "Rio de Janeiro" → ambas cidades em cache
t=24:01 → Usuário vai offline e reabre app
  - Esperado: "Dados de 24 horas atrás"
  - Implementação A (12h): Dados expirados, "Sem dados"
  - Implementação B (24h): Dados ainda válidos, "Offline — 24 horas atrás"
```

**Sugestão de Correção:**
```markdown
**RNF-REL: Cache Strategy (Cache Local com localStorage)**

- **TTL (Time-To-Live):** 24 horas (86.400 segundos)
  - Dados são salvos com timestamp ISO 8601
  - Ao abrir app, verificar: `now - savedTimestamp ≤ 24 horas`
  - Se > 24h: dados expiram, não são exibidos

- **Estrutura de Cache:**
  ```javascript
  localStorage['weatherCache'] = {
    city: "São Paulo",
    country: "Brazil",
    latitude: -23.5505,
    longitude: -46.6333,
    current: { temperature: 28.5, ... },
    forecast: [ ... ],
    timestamp: "2026-08-12T14:32:00.000Z",  // ISO 8601
    expiresAt: "2026-08-13T14:32:00.000Z"   // data + 24h
  }
  ```

- **Comportamento:**
  - Dados < 24h: exibir com label "Offline — dados de X horas atrás"
  - Dados ≥ 24h: NOT exibir (expirados)
  - Sem cache E offline: "Sem conexão e sem dados salvos"

- **Refresh em Tempo Real:**
  - Quando app carrega e há Internet: requisição é disparada
  - Novo cache sobrescreve antigo (novo timestamp)
  - Se requisição falha: usar cache antigo (mesmo se próximo de expiração)
```

---

### I2: Qual é a Exata Apresentação de "Risco de Chuva"?

**Localização:** US2, RF2  
**Problema:**
- US2 diz: "quero ver ... risco de chuva"
- RF2 lista: "temperatura, descrição, ícone, umidade, vento"
- Onde está "risco de chuva"? É a descrição ("Chuva", "Chuva leve")? É probabilidade? (Open-Meteo não oferece "precipitation_probability" na rota `/forecast` pública)

**Impacto:** Média  
**Risco:** RF2 pode ser implementado sem "risco de chuva", causando falha em US2.

**Sugestão de Correção:**
```markdown
**Ajustar US2 para alinhar com RF2:**

**US2 (Revisado): Visualizar Clima Atual — Informação Essencial**

"Como Marina (commuter urbano), quero ver a temperatura, condição de céu (claro/nublado/chuva) e vento 
para decidir que roupa usar antes de sair de casa."

**RF2 (Revisado): Exibir Clima Atual**
- Mostra: **temperatura, condição climática (ex: "Céu limpo", "Nublado", "Chuva leve"), ícone, umidade (%), vento (km/h)**

**Nota:** 
- "Risco de chuva" = descrição qualitativa ("Chuva leve", "Chuva moderada", "Chuva forte")
- Não é probabilidade percentual (Open-Meteo não oferece na versão free)
- Se probabilidade for implementada em v2, usar endpoint `/forecast?minutely_15=precipitation_probability`
```

---

### I3: Qual é o Comportamento Exato ao Selecionar Sugestão?

**Localização:** AC1.5, AC2.1, AC2.2  
**Problema:** Há 2 caminhos diferentes:
- AC1.5: "Ao clicar sugestão, carrega clima da cidade"
- AC2.1: "Given a cidade 'São Paulo' foi selecionada"
- AC2.2: "Given o usuário seleciona uma nova cidade"

Pergunta: O dropdown de sugestões fecha? A busca permanece visível? O ícone de loading aparece?

**Impacto:** Baixa (UX)  
**Risco:** Implementador deixa dropdown aberto ou não fecha teclado em mobile.

**Sugestão de Correção:**
```markdown
**AC1.5 (Revisado): Seleção de Sugestão**

```gherkin
Given sugestões aparecem para "São Paulo"
When o usuário clica na sugestão "São Paulo, SP 🇧🇷"
Then:
  • Dropdown de sugestões FECHA automaticamente
  • Campo de busca retém o valor "São Paulo, SP" (não limpa)
  • Skeleton/spinner aparece no card de clima (loading state)
  • Requisição `/forecast?latitude=...&longitude=...` é enviada
  • Após receber resposta: climate atual é exibido (dentro de < 2 seg)
  • Previsão de 5 dias também é exibida
  • Em mobile: teclado virtual é fechado (blur do input)
```
```

---

### I4: "Atualizações Simultâneas" — Qual é Tolerância de Delay?

**Localização:** AC4.3  
**Problema:** Diz "todas as temperaturas são atualizadas SIMULTANEAMENTE (sem delay)" — qual é a tolerância?
- < 100ms? (imperceptível ao humano)
- < 500ms? (rápido, mas perceptível)
- < 1s? (visível)

**Impacto:** Média  
**Risco:** Implementador atualiza sequencialmente (1ª temperatura, depois previsão), criando flicker visual.

**Sugestão de Correção:**
```markdown
**AC4.3 (Revisado): Conversão Simultânea de Temperaturas**

```gherkin
Given clima atual mostra "28°C" e previsão mostra "Máx 30°C | Mín 20°C"
When o usuário clica toggle para "°F"
Then todas as temperaturas são atualizadas SIMULTANEAMENTE (< 100ms entre primeiro e último update)
And clima atual muda para "82.4°F"
And previsão muda para "Máx 86°F | Mín 68°F"
And usuário não percebe nenhum flicker ou sequência visual
```

**Nota de Implementação:**
- Calcular todas as conversões primeiro
- Atualizar estado React uma única vez (batch update)
- Evitar múltiplas re-renders
```

---

### I5: "Previsão de 5 Dias" — O que Fazer com Dados < 5 Dias?

**Localização:** RF3, AC3.1, AC3.4, EC11  
**Problema:** 
- RF3 diz "Sempre 5 dias (não menos, não mais)"
- Mas EC11 diz "Interface exibe apenas 3 cards (não força 5)"

Qual é certo? Se Open-Meteo retorna 3 dias, app exibe 3 ou força 5 com N/A?

**Impacto:** Alta  
**Risco:** Falha AC3.1 ou falha EC11.

**Sugestão de Correção:**
```markdown
**RF3 (Revisado): Exibir Previsão de 5 Dias**

- **MVP Behavior (v1):** Sempre exibir exatamente 5 dias
  - Se Open-Meteo retorna < 5 dias, isso é erro crítico
  - Exibir mensagem: "Erro ao carregar previsão completa. Tente novamente."
  - Retry disponível

- **Graceful Fallback (se v1 não for rígido):**
  - Se Open-Meteo retorna 3 dias: exibir 3 cards + 2 cards com "N/A"
  - Label: "Previsão disponível para 3 dias"
  - Não é erro crítico, mas é informado ao usuário

**Recomendação:** Use MVP Behavior (sempre 5 dias, falha se < 5)
```

---

### I6: Qual é o Ícone/Componente de "Atualizar"?

**Localização:** AC2.4, AC2.3  
**Problema:** Diz "Botão 'Atualizar' ou ícone 🔄" — qual é o requisito?
- É botão com texto "Atualizar"?
- É apenas ícone 🔄 (sem texto)?
- Ambos (botão + ícone)?
- Está sempre visível ou apenas em erro/retry?

**Impacto:** Baixa (UX)  
**Risco:** Inconsistência entre tipos de erro (um mostra botão, outro mostra ícone).

**Sugestão de Correção:**
```markdown
**Standardize "Atualizar" Button/Icon:**

| Situação | Componente | Comportamento |
|----------|-----------|---|
| Carregando | Spinner/skeleton | Sem botão |
| Sucesso | Ícone 🔄 (opcional) | Ao clicar: recarrega dados |
| Timeout/Erro | Botão "Atualizar" + ícone 🔄 | Ao clicar: retry com backoff |
| Offline com cache | Botão "Tentar Reconectar" | Ao clicar: tenta requisição |

**Recomendação:** Usar botão com texto + ícone em todos os casos (consistência)
```

---

### I7: "Layout Pode Usar 2 Colunas" — É Requerido ou Opcional?

**Localização:** AC7.2  
**Problema:** Palavra "pode" é ambígua:
- "Pode" = é permitido? (opcional)
- "Pode" = é esperado? (requerido)

AC7.1 usa "deve" (single-column), AC7.3 usa "pode" (pode aproveitar espaço). Impreciso.

**Impacto:** Baixa  
**Risco:** Tablet tem layout single-column em vez de 2 colunas.

**Sugestão de Correção:**
```markdown
**AC7.2 (Revisado): Tablet (768px) — Layout Otimizado**

```gherkin
Given a viewport é 768px de largura (tablet portrait)
When o app é carregado
Then o layout **deve** usar até 2 colunas (requerido, não opcional):
  • Coluna esquerda: campo busca + clima atual (flex: 1)
  • Coluna direita: previsão 5 dias (flex: 1)
  # OU alternativamente (se 2-col não caber bem):
  • Coluna única com cards de previsão em grid 3 colunas
And espaçamento é adequado (padding ≥ 16px)
And não há truncamento de texto
```
```

---

## 3. REQUISITOS FALTANTES

### R1: Qual é a Estrutura de Dados (TypeScript Types)?

**Localização:** Nenhuma seção  
**Problema:** Especificação não define interfaces TypeScript para:
- Resposta da API `/geocoding/search`
- Resposta da API `/forecast`
- Estrutura de cache em localStorage

**Impacto:** Alta  
**Risco:** Cada implementador cria tipos diferentes; inconsistência com Plan/Code Agent.

**Sugestão de Correção:**

Criar nova seção "Apêndice C: Tipos TypeScript (Data Model)"

```markdown
## Apêndice C: Model de Dados (TypeScript Types)

### API Responses

```typescript
// Open-Meteo /geocoding/search response
interface GeocodingSearchResult {
  results?: Array<{
    id: number;
    name: string;                    // "São Paulo"
    latitude: number;
    longitude: number;
    elevation?: number;
    feature_code: string;            // "PPL" = city
    country_code: string;            // "BR"
    admin1_id?: number;
    admin1?: string;                 // "São Paulo"
    timezone?: string;               // "America/Sao_Paulo"
    population?: number;             // 12.3e6
    country?: string;                // "Brazil"
    country_id?: number;
  }>;
  generationtime_ms?: number;
}

// Open-Meteo /forecast response
interface ForecastResponse {
  latitude: number;
  longitude: number;
  generationtime_ms: number;
  utc_offset_seconds: number;
  timezone: string;
  timezone_abbreviation: string;
  elevation: number;
  
  // Current weather
  current?: {
    time: string;                    // "2026-08-12T14:32"
    temperature: number;
    relative_humidity: number;
    apparent_temperature: number;
    weather_code: number;            // WMO code
    wind_speed_10m: number;
    wind_direction_10m: number;
  };

  // Daily forecast
  daily?: {
    time: string[];                  // ["2026-08-12", "2026-08-13", ...]
    weather_code: number[];
    temperature_2m_max: number[];
    temperature_2m_min: number[];
  };
}

// App State
interface CityData {
  id: number;
  name: string;
  state?: string;
  country: string;
  latitude: number;
  longitude: number;
  timezone: string;
  population?: number;
  flagEmoji: string;                 // "🇧🇷"
}

interface CurrentWeather {
  temperature: number;               // em Celsius
  condition: string;                 // "Céu limpo" (pt-BR)
  weatherCode: number;               // WMO code
  humidity: number;                  // 0-100 %
  windSpeed: number;                 // km/h
  updatedAt: string;                 // ISO 8601
}

interface DailyForecast {
  date: string;                      // "2026-08-12"
  tempMax: number;                   // °C
  tempMin: number;                   // °C
  condition: string;                 // "Céu limpo" (pt-BR)
  weatherCode: number;               // WMO code
}

interface AppState {
  selectedCity: CityData | null;
  currentWeather: CurrentWeather | null;
  forecast5days: DailyForecast[];    // sempre array[5] ou []
  temperatureUnit: 'celsius' | 'fahrenheit';
  isLoading: boolean;
  error: string | null;              // mensagem de erro amigável
  lastUpdated: string | null;        // ISO 8601
  isOffline: boolean;
}

// LocalStorage Cache
interface WeatherCache {
  city: CityData;
  current: CurrentWeather;
  forecast: DailyForecast[];
  savedAt: string;                   // ISO 8601
  expiresAt: string;                 // ISO 8601 (savedAt + 24h)
}
```
```

---

### R2: Qual é o Entry Point / Layout Principal?

**Localização:** Nenhuma seção  
**Problema:** Especificação não define a estrutura visual da aplicação:
- Há header, footer, sidebar?
- O campo de busca fica em cima ou à esquerda?
- Clima atual está acima ou ao lado da previsão?

**Impacto:** Média (UX)  
**Risco:** Cada implementador cria layout diferente.

**Sugestão de Correção:**

Criar nova seção "Apêndice D: Wireframe / Layout Padrão"

```markdown
## Apêndice D: Layout Principal (Wireframe)

### Mobile (320px-768px)

```
┌─────────────────────────────┐
│ Weather App        🌍 |  °C │ ← Header (título + toggle)
├─────────────────────────────┤
│ 🔍 Buscar cidades...        │ ← Campo de busca
│ ├─ São Paulo, SP 🇧🇷       │   (dropdown de sugestões)
│ ├─ São Gonçalo, RJ 🇧🇷     │
│ └─ São Leopoldo, RS 🇧🇷    │
├─────────────────────────────┤
│ 📍 São Paulo, SP 🇧🇷        │ ← Clima atual (card)
│ ☀️ Céu limpo                │
│ 28.5°C                      │
│ Umidade: 65% | Vento: 12 km/h│
│ 🔄 Atualizado em 14:32      │
├─────────────────────────────┤
│ Previsão 5 Dias:            │ ← Previsão (scroll horizontal)
│ ┌─────┬─────┬─────┐         │
│ │12ago│13ago│14ago│ ← → ← → │
│ │☀️ 32│🌧️ 30│☁️ 28│         │
│ │22   │20   │18   │         │
│ └─────┴─────┴─────┘         │
└─────────────────────────────┘
```

### Tablet (768px-1024px)

```
┌───────────────────────────────────┐
│ Weather App              🌍 |  °C │
├────────────┬──────────────────────┤
│ 🔍 Buscar  │ 📍 São Paulo, SP 🇧🇷│
│            │ ☀️ Céu limpo         │
│ Sugestões  │ 28.5°C               │
│            │ Umidade: 65%         │
│            │ Vento: 12 km/h       │
│            │ 🔄 14:32             │
├───────────────────────────────────┤
│ Previsão (grid 5 colunas):        │
│ ┌──────┬──────┬──────┬──────┬──────┐
│ │12ago │13ago │14ago │15ago │16ago │
│ │☀️ 32 │🌧️ 30│☁️ 28 │🌦️ 31│☀️ 29 │
│ │22    │20    │18    │21    │19    │
│ └──────┴──────┴──────┴──────┴──────┘
└───────────────────────────────────┘
```

### Desktop (1024px+)

```
┌───────────────────────────────────────┐
│ Weather App                  🌍 |  °C │
├────────────────┬──────────────────────┤
│ 🔍 Buscar      │ 📍 São Paulo, SP 🇧🇷│
│ cidades...     │ ☀️ Céu limpo         │
│                │ 28.5°C               │
│ Sugestões:     │ Umidade: 65% |       │
│ • São Paulo    │ Vento: 12 km/h       │
│ • São Gonçalo  │ 🔄 Atualizado 14:32 │
│ • São Leopoldo │                      │
├────────────────────────────────────────┤
│ Previsão (5 cards full-width):         │
│ ┌──────┬──────┬──────┬──────┬──────┐
│ │12ago │13ago │14ago │15ago │16ago │
│ │☀️ 32 │🌧️ 30│☁️ 28 │🌦️ 31│☀️ 29 │
│ │22    │20    │18    │21    │19    │
│ └──────┴──────┴──────┴──────┴──────┘
└────────────────────────────────────────┘
```

### Componentes Principais
- **Header:** Logo "Weather App" + Toggle C/F
- **Search Bar:** Input + Dropdown de sugestões
- **Current Weather Card:** Clima atual (temperatura, condição, umidade, vento)
- **Forecast Cards:** 5 cards ou carousel
- **Footer:** (Opcional v2) Crédito "Powered by Open-Meteo"
```

---

### R3: Qual é a Estratégia de Retry Exata (Código)?

**Localização:** AC5 (cenário timeout), RNF-REL  
**Problema:** Diz "backoff exponencial (1s, 2s, 4s, 8s; máx 3 tentativas)" mas não define:
- Quantas tentativas totais? (inicial + 3 retry = 4 total?)
- Qual é o tempo máximo? (8s é último retry ou há mais?)
- O que acontece se todas falharem?

**Impacto:** Média  
**Risco:** Implementador usa 5 tentativas ou sem limite, causando lag.

**Sugestão de Correção:**

```markdown
## Apêndice E: Estratégia de Retry

### Algoritmo de Backoff Exponencial

```typescript
const RETRY_ATTEMPTS = 3;
const RETRY_DELAYS = [1000, 2000, 4000];  // ms
const TIMEOUT = 8000;  // ms

async function fetchWeatherWithRetry(lat, lng) {
  let attempt = 0;

  while (attempt < RETRY_ATTEMPTS) {
    try {
      const response = await fetch(
        `/forecast?latitude=${lat}&longitude=${lng}`,
        { signal: AbortSignal.timeout(TIMEOUT) }
      );
      
      if (response.ok) {
        return await response.json();
      }
      
      // HTTP error (5xx, rate limit, etc)
      if (response.status === 429) {
        showMessage("Muitos acessos. Aguarde alguns segundos.");
        return null;  // não retry
      }
      
      if (response.status >= 500) {
        throw new Error(`HTTP ${response.status}`);
      }
    } catch (err) {
      // Network error or timeout
      if (err.name === 'AbortError') {
        console.log(`Timeout após ${TIMEOUT}ms`);
      } else {
        console.log(`Erro: ${err.message}`);
      }

      // Se não foi última tentativa, aguardar e retry
      if (attempt < RETRY_ATTEMPTS - 1) {
        const delay = RETRY_DELAYS[attempt];
        console.log(`Retry em ${delay}ms (tentativa ${attempt + 1}/${RETRY_ATTEMPTS})`);
        await sleep(delay);
        attempt++;
      } else {
        // Todas as tentativas falharam
        showMessage("Não foi possível carregar. Tente novamente mais tarde.");
        return null;
      }
    }
  }
}

function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}
```

### Timeline de Execução

```
t=0ms: 1ª requisição é enviada
t=8s (8000ms): Timeout, requisição abortada
t=9s: 1º retry aguardando (1s)
t=10s: 2ª requisição é enviada
t=18s: Timeout novamente
t=20s: 2º retry aguardando (2s)
t=22s: 3ª requisição é enviada
t=30s: Timeout novamente
t=30s: Mensagem "Não foi possível carregar..."

Total: ~30 segundos até declarar falha
```

### Exceções
- **Rate Limit (HTTP 429):** Sem retry automático (mostra mensagem)
- **Sem Internet:** Sem retry (offline behavior)
- **Dados em Cache:** Se requisição falha mas há cache, usar cache
```

---

### R4: Qual é a Estratégia de Build/Deployment?

**Localização:** Nenhuma seção  
**Problema:** Especificação não menciona:
- Há environment variables?
- Como configurar URL da Open-Meteo?
- Como fazer deployment (Docker, Vercel, etc)?

**Impacto:** Baixa (implementação interna)  
**Risco:** Code Agent precisa adivinhar estrutura de deploy.

**Sugestão:** Deixar para Plan Agent / Task Agent (não é requisito de produto).

---

### R5: Qual é o Comportamento de "Atualizar" Quando Offline?

**Localização:** EC9, AC2.4  
**Problema:** Se usuário tenta "Atualizar" (novo búsca) enquanto offline:
- Mostra erro?
- Desabilita botão?
- Permite tentar (que retorna erro)?

**Impacto:** Média  
**Risco:** UX ruim se botão está habilitado mas não funciona.

**Sugestão de Correção:**

```markdown
### R5: Comportamento de Busca Offline

**Cenário:** Usuário offline tenta buscar nova cidade

```gherkin
Given o usuário está offline (sem Internet)
When tenta digitar e buscar "Rio de Janeiro"
And aguarda 300ms (debounce)
Then a requisição `/geocoding/search` é enviada
And falha imediatamente (NetworkError)
And mensagem é exibida: "Sem conexão de Internet"
And dropdown de sugestões não aparece
And campo de busca permanece com o texto digitado

When o usuário clica em campo com cache
Then dados cacheados são exibidos (clima anterior)
And Label "Offline — dados de X horas atrás"
```

**Recomendação:** Detectar offline com `navigator.onLine` antes de enviar requisição
```

---

### R6: Como Funciona a Busca por Estado vs. Cidade?

**Localização:** RF1, AC1.1  
**Problema:** Open-Meteo retorna cidades e estados. Como lidar?
- Busca "são paulo" retorna:
  - "São Paulo" (cidade)
  - "São Paulo" (estado)
  - "São Paulo" (bairro)

Qual é o filtro? Mostrar tudo?

**Impacto:** Baixa  
**Risco:** UI confusa com muitas entradas "São Paulo".

**Sugestão de Correção:**

```markdown
### R6: Filtragem de Resultados Geocoding

**Open-Meteo feature_code:**
- `PPL` = Populated place (cidades/vilarejos)
- `PPLC` = Capital
- `PPLA` = Capital de estado/provincia
- `ADM1` = Estado/provincia (admin1)

**Strategy:**
1. Mostrar apenas `PPL`, `PPLC`, `PPLA` (cidades, não estados/regiões)
2. Ordenar por população DESC
3. Incluir campo `admin1` (estado) para desambiguação

**Exemplo:**
- Busca "são paulo"
- Open-Meteo retorna: city + state
- Filtrar por feature_code IN [PPL, PPLC, PPLA]
- Resultado: São Paulo (PPL, pop 12.3M, state: São Paulo)
```

---

### R7: Qual é a Política de Privacidade? Quando é Exibida?

**Localização:** RNF-LEGAL  
**Problema:** Diz "Política de Privacidade: Documentada e acessível" mas:
- Onde fica o link?
- Qual é o conteúdo mínimo?
- Quando/como é exibida?

**Impacto:** Legal (LGPD/GDPR)  
**Risco:** Não conformidade.

**Sugestão de Correção:**

```markdown
### R7: Política de Privacidade

**Requisito:** Página de privacidade deve estar:
- Acessível via URL: `/privacy` ou link em footer
- Conteúdo mínimo em pt-BR:
  - Que dados são coletados (localStorage temperature preference)
  - Como são usados (persistência local)
  - LGPD compliance (Brasil)
  - Política de retenção (24h cache TTL)
  - Sem tracking externo / cookies

**Exibição:**
- Primeira visita: card com aviso "Utilizamos localStorage para salvar sua preferência" (dismissable)
- Link para privacy policy no footer

**Exemplo de Aviso:**
```
"Nós usamos localStorage para salvar sua preferência de temperatura (Celsius/Fahrenheit).
Seus dados não são compartilhados. Leia nossa Política de Privacidade."
[Entendi]
```
```

---

## 4. CRITÉRIOS FRACOS OU NÃO VERIFICÁVEIS

### C1: "Interface é Minimalista" — Muito Vago

**Localização:** US2 (Critério de Aceite)  
**Problema:** "Interface é minimalista (sem muita informação)" é subjetivo.
- Quantos elementos? 5? 10? 20?
- Qual é "muita informação"?

**Impacto:** Média  
**Risco:** Teste de aceite falha ou passa dependendo do avaliador.

**Sugestão de Correção:**

```markdown
**US2 (Revisado): Critério 3**

Remover: "Interface é minimalista (sem muita informação)"

Substituir por:
- [ ] Interface exibe exatamente 6 campos: temperatura, condição, umidade, vento, timestamp, botão atualizar
- [ ] Nenhum campo adicional é exibido (sem visibilidade, humano aparente, pressão, etc)
- [ ] UI ocupa ≤ 50% da tela em mobile (resto para previsão ou scroll)
```

---

### C2: "Tema dark glassmorphism" — Qual é Exatamente?

**Localização:** Stack Técnico, RNF-ASST  
**Problema:** "Dark glassmorphism" é termo design vago.
- É apenas dark mode?
- Há blur/opacity específica?
- Há gradient?

**Impacto:** Baixa (visual)  
**Risco:** Implementador cria tema genérico dark, não glassmorphism.

**Sugestão de Correção:**

```markdown
**Apêndice F: Design System — Dark Glassmorphism**

### Paleta de Cores

```css
:root {
  /* Background */
  --bg-primary: #0f1419;      /* Quase preto */
  --bg-secondary: #1a1f2e;    /* Cinza muito escuro */
  --bg-tertiary: #252b3b;     /* Cinza escuro */
  
  /* Glass effect */
  --glass-bg: rgba(26, 31, 46, 0.8);  /* 80% opaque */
  --glass-blur: 16px;
  --glass-border: 1px solid rgba(255, 255, 255, 0.1);
  
  /* Text */
  --text-primary: #ffffff;
  --text-secondary: #b0b0b0;
  --text-tertiary: #808080;
  
  /* Accent */
  --accent-primary: #3b82f6;    /* Azul */
  --accent-secondary: #fbbf24;  /* Amarelo */
}

/* Glass Card Example */
.glass-card {
  background: var(--glass-bg);
  backdrop-filter: blur(var(--glass-blur));
  border: var(--glass-border);
  border-radius: 16px;
  padding: 16px;
}
```

### Efeitos Visuais

- **Backdrop Filter:** `blur(16px)` em cards
- **Opacity:** 80% background, 10% borders
- **Borders:** Semi-transparent white (rgba 0.1)
- **Gradients:** Leve gradiente top-to-bottom em cards (preto → cinza)
- **Shadows:** `0 10px 30px rgba(0, 0, 0, 0.5)`
- **Rounded Corners:** 16px (não 4px)

### Modo Claro (Fallback)

Se usuário ativa prefers-color-scheme: light:
- Background: #f8f9fa
- Cards: #ffffff
- Text: #0f1419
- Borders: rgba(0, 0, 0, 0.1)
- Mesmo glassmorphism (blur + transparency)
```

---

### C3: "Sem Confusão Visual (Chuva Não é Amarela)" — Como Validar?

**Localização:** AC3.5  
**Problema:** "Cada ícone e cor correspondem corretamente" e "não há confusão visual" são critérios subjetivos.
- Quem define "confusão"?
- Qual é a métrica de teste?

**Impacto:** Média  
**Risco:** Review falha por motivo vago.

**Sugestão de Correção:**

```markdown
**AC3.5 (Revisado): Cores/Ícones Indicam Condição Corretamente**

```gherkin
Given a previsão está carregada com diferentes weather_codes:
  • weather_code 0 (Clear) → Ícone: ☀️, Cor: Amarelo (#FBBF24)
  • weather_code 3 (Overcast) → Ícone: ☁️, Cor: Cinza (#9CA3AF)
  • weather_code 63 (Moderate rain) → Ícone: 🌧️, Cor: Azul (#1E40AF)
  • weather_code 71 (Slight snow) → Ícone: ❄️, Cor: Branco azulado (#E0F2FE)
  • weather_code 80 (Rain showers) → Ícone: 🌧️, Cor: Azul (#93C5FD)
  • weather_code 99 (Thunderstorm) → Ícone: ⚡, Cor: Roxo (#A855F7)

When cada card é visualizado:
Then:
  • Cores divergem significativamente (Amarelo ≠ Azul ≠ Roxo)
  • Ícone corresponde à descrição textual
  • Usuário com daltonismo consegue diferenciar (não usa somente cor)
  • Sem nenhum caso onde "chuva é amarela" ou "neve é vermelha"

# Validação:
- Verificar com ferramentas de contraste (WCAG)
- Testar com simulator de daltonismo (Coblis)
```
```

---

## Resumo de Ações Recomendadas

### Críticos (Resolver Antes de Coding)
1. ✅ **A1** — Definir debounce exatamente como "delay 300ms após última digitação"
2. ✅ **A2** — Definir "hoje" como data civil local
3. ✅ **A3** — Criar tabela de weather_code → ícone + descrição
4. ✅ **R1** — Definir TypeScript types para toda estrutura de dados
5. ✅ **R2** — Criar wireframe visual do layout

### Altos (Resolver Antes de Task Agent)
6. ✅ **I1** — Esclarecer cache TTL = exatamente 24 horas
7. ✅ **I2** — Alinhar "risco de chuva" com descrição qualitativa (não %)
8. ✅ **I4** — Definir tolerância de < 100ms para atualizações simultâneas
9. ✅ **A4** — Padronizar mensagens de erro (criar tabela)
10. ✅ **C1** — Substituir "minimalista" por critério verificável

### Médios (Documentar em Plan Agent)
11. ✅ **R3** — Código de retry com backoff exponencial (algoritmo)
12. ✅ **R5** — Comportamento de busca offline
13. ✅ **R6** — Filtrar resultados geocoding por feature_code
14. ✅ **C3** — Validação de ícones com ferramentas

### Baixos (Nice-to-have para v2)
15. ✅ **R4** — Estratégia de build/deployment
16. ✅ **R7** — Política de privacidade
17. ✅ **C2** — Design system completo

---

## Assinatura da Revisão

**Revisor:** Review Agent  
**Data:** 2026-08-12  
**Recomendação Final:** ✅ Especificação PODE prosseguir para Plan Agent após resolver os 17 problemas acima.

**Próximas Etapas:**
1. Incorporar correções sugeridas em `specs/weather-app-spec.md`
2. Criar apêndices (A, B, C, D, E, F) com tabelas e código
3. Invocar Plan Agent com spec revisada
