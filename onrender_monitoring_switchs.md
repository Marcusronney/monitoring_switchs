# Documentação técnica do `onRender` — Dashboard Monitoring Switchs

**Autor:** Marcus Rônney  
**Repositório GitHub:** https://github.com/Marcusronney/monitoring_switchs  
**LinkedIn:** https://www.linkedin.com/in/marcus-r%C3%B4nney-627657bb/?skipRedirect=true  
**Página pessoal:** http://about.ronney.tech/  
**Arquivo base:** `Código colado.js`  
**Data:** 16/06/2026 12:11

---

## 1. Objetivo do `onRender`

O `onRender` é o JavaScript executado pelo plugin **HTML Graphics** do Grafana sempre que o painel recebe novos dados do datasource Zabbix.

Ele transforma as séries retornadas pelo Zabbix em uma visualização operacional de switches, exibindo:

- status dos switches via ICMP;
- latência e perda ICMP;
- tráfego RX/TX por switch;
- tráfego total da LAN;
- porta monitorada/cascateamento de cada switch;
- status operacional da porta;
- download/upload da porta monitorada;
- erros RX/TX;
- descartes RX/TX;
- indicadores de instabilidade;
- mini gráficos de RX/TX;
- área de diagnóstico técnico.

Fluxo geral:

```text
Dados do Zabbix/Grafana
→ leitura das séries
→ identificação do host
→ identificação da métrica
→ identificação da porta
→ normalização da porta
→ agrupamento por switch e interface
→ cálculo dos indicadores
→ renderização dos cards, gráficos e tabela
```

---

## 2. Estrutura geral

O código é encapsulado em uma função autoexecutável:

```javascript
(function () {
  // lógica do painel
})();
```

Isso evita que variáveis internas vazem para o escopo global do navegador ou do Grafana.

---

## 3. Inicialização do contexto do HTML Graphics

O código começa capturando os objetos disponibilizados pelo plugin:

```javascript
const hg = typeof htmlGraphics !== 'undefined' ? htmlGraphics : {};
const props = hg.customProperties || (typeof customProperties !== 'undefined' ? customProperties : {}) || {};
const panelData = hg.data || (typeof data !== 'undefined' ? data : { series: [] });
```

### Responsabilidade

| Variável | Descrição |
|---|---|
| `hg` | Objeto principal do plugin HTML Graphics |
| `props` | Custom properties configuradas no painel |
| `panelData` | Dados retornados pelas queries do Grafana/Zabbix |

Essa etapa garante que o código funcione mesmo se o plugin entregar os dados em formatos ligeiramente diferentes.

---

## 4. Captura dos elementos HTML

O código captura elementos do HTML usando `querySelector`.

Principais elementos:

```javascript
const root = htmlNode.querySelector('#sw1-root');
const rowsEl = htmlNode.querySelector('#switchRows');
const emptyEl = htmlNode.querySelector('#emptyState');
const searchEl = htmlNode.querySelector('#tableSearch');
```

Elementos dos mini gráficos:

```javascript
const miniRxValueEl = htmlNode.querySelector('#miniRxValue');
const miniTxValueEl = htmlNode.querySelector('#miniTxValue');
const miniRxSvgEl = htmlNode.querySelector('#miniRxSvg');
const miniTxSvgEl = htmlNode.querySelector('#miniTxSvg');
```

Elementos dos cards:

```javascript
const kpiTotalEl = htmlNode.querySelector('#kpiTotal');
const kpiOnlineEl = htmlNode.querySelector('#kpiOnline');
const kpiOfflineEl = htmlNode.querySelector('#kpiOffline');
const kpiPort1ProblemsEl = htmlNode.querySelector('#kpiPort1Problems');
const kpiTotalTrafficEl = htmlNode.querySelector('#kpiTotalTraffic');
```

### Responsabilidade

Essa etapa conecta o JavaScript ao HTML do painel. Sem esses elementos, o código não tem onde renderizar cards, tabela, gráficos e diagnóstico.

O trecho abaixo evita erro caso o HTML esteja incompleto:

```javascript
if (!root || !rowsEl || !emptyEl) return;
```

---

## 5. Configurações da dashboard com `cfg`

O objeto `cfg` centraliza valores padrão e permite sobrescrever tudo pelas **Custom properties** do HTML Graphics.

```javascript
const cfg = Object.assign({
  title: 'Switches - Status e Porta Monitorada',
  subtitle: 'Status ICMP, latência/perda ICMP, tráfego total RX/TX e métricas da porta monitorada',
  monitoredPort: '1',
  monitoredPortByHost: {},
  fallbackSpeedMbps: 1000,
  miniSeriesBucketMs: 60000,
  usageWarnPct: 70,
  usageCriticalPct: 90,
  errorWarnCount: 1,
  discardWarnCount: 1
}, props);
```

### Campos principais

| Campo | Função |
|---|---|
| `title` | Título exibido na dashboard |
| `subtitle` | Subtítulo exibido na dashboard |
| `monitoredPort` | Porta padrão monitorada |
| `monitoredPortByHost` | Mapa de porta específica por switch |
| `fallbackSpeedMbps` | Velocidade padrão usada como fallback |
| `miniSeriesBucketMs` | Janela de agrupamento dos mini gráficos |
| `usageWarnPct` | Percentual de alerta de uso |
| `usageCriticalPct` | Percentual crítico de uso |
| `errorWarnCount` | Limite para destacar erros |
| `discardWarnCount` | Limite para destacar descartes |

### Exemplo de custom properties

```json
{
  "monitoredPort": "1",
  "monitoredPortByHost": {
    "/Switch Cisco Catalyst 2960G/": "47",
    "/Switch HP 1920S SRV/": "2",
    "/Switch Call Center Mikrotik/": "15"
  },
  "miniSeriesBucketMs": 60000
}
```

---

## 6. Estado da tabela

O código mantém estado visual da tabela:

```javascript
const state = root.__switchTableState || {
  sortKey: 'host',
  sortDir: 'asc',
  filter: 'all',
  search: ''
};
root.__switchTableState = state;
```

### Responsabilidade

Preserva:

- coluna de ordenação;
- direção da ordenação;
- filtro ativo;
- texto de busca.

Isso evita que a tabela “resete” completamente a cada atualização de dados.

---

## 7. Segurança na renderização com `safeText`

```javascript
const safeText = (value) => String(value ?? '').replace(/[&<>"']/g, ...);
```

### Responsabilidade

Escapa caracteres especiais antes de inserir valores no HTML.

Evita problemas como:

- quebra de layout;
- interpretação de HTML vindo de nomes de itens;
- caracteres especiais em aliases de interface.

---

## 8. Normalização textual com `normalizeText`

```javascript
const normalizeText = (text) => String(text || '')
  .normalize('NFD')
  .replace(/[\u0300-\u036f]/g, '')
  .toLowerCase();
```

### Responsabilidade

Remove acentos e padroniza minúsculas.

Exemplos:

```text
Tráfego → trafego
Fundição → fundicao
ICMP Latência → icmp latencia
```

É usado para busca, regex, comparação de host e classificação de métricas.

---

## 9. Conversão de vetores do Grafana com `vectorToArray`

```javascript
const vectorToArray = (values) => {
  ...
};
```

### Responsabilidade

O Grafana pode entregar séries em diferentes estruturas internas. Essa função converte tudo para array comum.

Ela suporta:

- arrays;
- objetos com `.toArray()`;
- vetores com `.get(i)`;
- objetos com `.length`.

---

## 10. Último valor válido com `lastNonEmpty`

```javascript
const lastNonEmpty = (field) => {
  ...
};
```

### Responsabilidade

Busca o último valor válido da série.

Se a série estiver sem valor direto, tenta usar cálculos internos do Grafana:

```javascript
lastNotNull
last
mean
max
```

É usada para alimentar cards e tabela com o valor mais recente de cada item.

---

## 11. Tempo da coleta com `getTimeField` e `lastTime`

```javascript
const getTimeField = (series) => ...
const lastTime = (series) => ...
```

### Responsabilidade

Localiza o campo de tempo da série e retorna o último timestamp válido.

É usado para:

- coluna “última coleta”;
- mini gráficos de RX/TX;
- controle de atualização de métricas.

---

## 12. Histórico dos mini gráficos com `addSeriesToHistory`

```javascript
const addSeriesToHistory = (bucketMap, timeField, valueField) => {
  ...
};
```

### Responsabilidade

Percorre uma série temporal e acumula valores em um mapa.

Formato lógico:

```text
timestamp → valor somado
```

É usado para montar os mini gráficos de:

- RX total;
- TX total.

---

## 13. Conversão do histórico em pontos

```javascript
const historyToPoints = (bucketMap) => ...
```

### Responsabilidade

Converte o mapa de histórico em array ordenado:

```javascript
[
  { ts: 1710000000000, value: 1000000 },
  { ts: 1710000060000, value: 1500000 }
]
```

---

## 14. Redução de pontos com `downsamplePoints`

```javascript
const downsamplePoints = (points, maxPoints) => {
  ...
};
```

### Responsabilidade

Reduz a quantidade de pontos do gráfico para melhorar performance.

Exemplo:

```text
500 pontos → 64 pontos
```

---

## 15. Desenho dos mini gráficos com `renderSparkline`

```javascript
const renderSparkline = (svgEl, points, color) => {
  ...
};
```

### Responsabilidade

Desenha um gráfico SVG pequeno, usado no cabeçalho.

A função calcula:

- escala mínima e máxima;
- coordenadas X/Y;
- linha do gráfico;
- área preenchida;
- ponto final destacado.

Usos:

```javascript
renderSparkline(miniRxSvgEl, rxHistoryPoints, '#22c55e');
renderSparkline(miniTxSvgEl, txHistoryPoints, '#a78bfa');
```

---

## 16. Conversão de tráfego com `metricValueToNumber`

```javascript
const metricValueToNumber = (value) => {
  ...
};
```

### Responsabilidade

Converte valores com unidade para número em bps.

Exemplos:

```text
1.01 Gbps → 1010000000
500 Mbps  → 500000000
10 Kbps   → 10000
0 bps     → 0
```

É uma das funções mais importantes do código, pois todos os cálculos de tráfego dependem dela.

---

## 17. Conversão de duração com `durationToMs`

```javascript
const durationToMs = (value) => {
  ...
};
```

### Responsabilidade

Converte latência para milissegundos.

Exemplos:

```text
652 µs → 0.652 ms
5.7 ms → 5.7 ms
0.01 s → 10 ms
```

---

## 18. Identificação do nome da série com `getDisplayName`

```javascript
const getDisplayName = (series, field) => {
  ...
};
```

### Responsabilidade

Escolhe o melhor nome para a série recebida do Zabbix.

Prioriza nomes que contenham:

```text
Interface ...
ICMP
Ping
Bits received
Bits sent
Operational status
packets with errors
packets discarded
```

Isso melhora a chance de identificar corretamente a métrica e a porta.

---

## 19. Construção do texto completo com `getFullSearchText`

```javascript
const getFullSearchText = (series, field, displayName) => {
  ...
};
```

### Responsabilidade

Concatena informações da série, campo e labels.

Esse texto completo serve como fallback para:

- identificar host;
- identificar métrica;
- identificar porta.

---

## 20. Limpeza de nome do host com `cleanHostName`

```javascript
const cleanHostName = (host) => {
  ...
};
```

### Responsabilidade

Remove aspas, espaços duplicados e valores vazios do nome do host.

Se não houver host, retorna:

```text
Switch
```

---

## 21. Identificação do host com `hostFrom`

```javascript
const hostFrom = (series, field, displayName, fullText) => {
  ...
};
```

### Responsabilidade

Descobre qual switch gerou a série.

Tenta localizar o host em:

1. labels do campo;
2. labels da série;
3. nome exibido;
4. texto completo;
5. JSON de labels.

Labels procuradas:

```text
host
hostname
host_name
hostName
zabbix_host
zbx_host
```

---

## 22. Classificação das métricas com `metricFrom`

```javascript
const metricFrom = (text) => {
  ...
};
```

### Responsabilidade

Classifica o item recebido em uma métrica interna da dashboard.

Métricas possíveis:

| Métrica interna | Significado |
|---|---|
| `icmpLatency` | Latência ICMP |
| `icmpLoss` | Perda ICMP |
| `switchStatus` | Status ICMP |
| `inErrors` | Erros RX |
| `outErrors` | Erros TX |
| `inDiscards` | Descartes RX |
| `outDiscards` | Descartes TX |
| `operStatus` | Status operacional da porta |
| `interfaceType` | Tipo da interface |
| `speed` | Velocidade |
| `bitsReceived` | Tráfego RX |
| `bitsSent` | Tráfego TX |

### Observação importante

O código classifica erros e descartes antes de tráfego.

Isso evita que um item como:

```text
Inbound packets with errors
```

seja confundido com tráfego de entrada.

---

## 23. Limpeza do nome da porta com `cleanPortName`

```javascript
const cleanPortName = (value) => {
  ...
};
```

### Responsabilidade

Limpa prefixos e textos extras do nome da porta.

Remove:

- aspas;
- espaços duplicados;
- `port`;
- `porta`;
- `()`;
- prefixos de erro;
- prefixos de descarte;
- prefixos SNMP.

Exemplos:

```text
Port 15 → 15
Interface 32() → 32
ifInErrorsGi0/10 → Gi0/10
CRC Errors RX on port 18 → 18
```

---

## 24. Normalização da porta com `normalizePortKey`

```javascript
const normalizePortKey = (value) => {
  ...
};
```

### Responsabilidade

Padroniza nomes de portas diferentes para uma chave comparável.

Exemplos:

```text
Gi0/47                 → 47
GigabitEthernet0/47    → 47
gigabitEthernet 1/0/20 → 20
g1                     → 1
Port15                 → 15
Interface 32()         → 32
```

Isso permite configurar a porta monitorada apenas pelo número.

### Exceção

Trunks são preservados:

```text
TRK 1 → trk1
```

---

## 25. Definição da porta monitorada com `getMonitoredPortForHost`

```javascript
const getMonitoredPortForHost = (host) => {
  ...
};
```

### Responsabilidade

Retorna qual porta deve ser monitorada para um switch.

A função tenta casar o host por:

1. nome exato;
2. regex;
3. texto parcial.

Se não encontrar regra, usa a porta padrão:

```javascript
cfg.monitoredPort
```

---

## 26. Comparação de porta com `isMonitoredPort`

```javascript
const isMonitoredPort = (portKey, portLabel, wantedPort) => {
  ...
};
```

### Responsabilidade

Compara a porta encontrada com a porta configurada.

Exemplo:

```text
Porta encontrada: GigabitEthernet0/47
Porta configurada: 47
Resultado: corresponde
```

---

## 27. Extração da porta com `portFrom`

```javascript
const portFrom = (displayName, fullText) => {
  ...
};
```

### Responsabilidade

Extrai o identificador da porta a partir do nome do item.

Reconhece padrões como:

```text
Interface Gi0/10(): Bits received
Interface g1(Onu Engenharia): Bits sent
ifInErrorsFa0/18
OutErrorsGi0/10
InDiscardsGi0/10
CRC Errors RX on port 18
GigabitEthernet0/31 - Outbound packets
Port15 - Inbound traffic
```

Se não conseguir identificar a porta, retorna `null`.

---

## 28. Conversão do status da porta com `statusInfo`

```javascript
const statusInfo = (value) => {
  ...
};
```

### Responsabilidade

Converte `ifOperStatus` em status visual.

| Valor | Status |
|---:|---|
| 1 | UP |
| 2 | DOWN |
| 3 | Testing |
| 4 | Unknown |
| 5 | Dormant |
| 6 | Not present |
| 7 | Lower layer down |

---

## 29. Conversão do status do switch com `switchIcmpStatusInfo`

```javascript
const switchIcmpStatusInfo = (value) => {
  ...
};
```

### Responsabilidade

Converte o valor de ICMP ping em status do switch.

```text
1 ou maior → ONLINE
0          → OFFLINE
vazio      → Sem dado
```

Também reconhece textos como `online`, `offline`, `up`, `down`, `available` e `unavailable`.

---

## 30. Formatação dos valores

### `formatBps`

Formata tráfego em bps, Kbps, Mbps ou Gbps.

### `formatCounter`

Formata contadores de erros e descartes.

### `formatLatency`

Formata latência.

### `formatPercent`

Formata percentuais.

### `formatIcmp`

Escolhe se deve mostrar latência ou perda ICMP.

---

## 31. Atualização segura com `setMetric`

```javascript
const setMetric = (target, metric, valueRaw, ts, preferMax) => {
  ...
};
```

### Responsabilidade

Atualiza uma métrica em um switch ou porta.

A função evita sobrescrever valores úteis com valores antigos ou vazios.

Quando `preferMax` está ativo, em empate de timestamp ela preserva o maior valor.

---

## 32. Estruturas principais em memória

```javascript
const devices = new Map();
const totalRxHistory = new Map();
const totalTxHistory = new Map();
const calcRxHistory = new Map();
const calcTxHistory = new Map();
const debugLines = [];
```

| Estrutura | Função |
|---|---|
| `devices` | Guarda switches e portas |
| `totalRxHistory` | RX calculado pela Query A |
| `totalTxHistory` | TX calculado pela Query A |
| `calcRxHistory` | RX vindo do item calculado Traffic LAN |
| `calcTxHistory` | TX vindo do item calculado Traffic LAN |
| `debugLines` | Diagnóstico técnico |

---

## 33. Criação dos objetos de switch com `getDevice`

```javascript
const getDevice = (host) => {
  ...
};
```

### Responsabilidade

Cria ou recupera o switch dentro do mapa `devices`.

Cada switch contém:

- status ICMP;
- latência;
- perda;
- mapa de portas;
- dados brutos;
- última atualização.

---

## 34. Criação dos objetos de porta com `getPort`

```javascript
const getPort = (device, portLabel) => {
  ...
};
```

### Responsabilidade

Cria ou recupera uma porta dentro do switch.

A chave da porta é gerada com:

```javascript
normalizePortKey(portLabel)
```

Cada porta guarda:

- `bitsReceived`;
- `bitsSent`;
- `inErrors`;
- `outErrors`;
- `inDiscards`;
- `outDiscards`;
- `operStatus`;
- `speed`;
- dados brutos.

---

## 35. Loop principal de leitura das séries

O processamento principal percorre todas as séries e campos:

```javascript
(panelData.series || []).forEach((series) => {
  (series.fields || []).forEach((field) => {
    ...
  });
});
```

### Etapas do loop

1. Ignora campos de tempo.
2. Busca o último valor válido.
3. Obtém nome exibido.
4. Monta texto completo.
5. Normaliza texto.
6. Captura campo de tempo.
7. Verifica se é item calculado RX/TX.
8. Classifica a métrica.
9. Adiciona histórico RX/TX.
10. Identifica host.
11. Identifica porta.
12. Atualiza objeto da porta.
13. Adiciona diagnóstico.

---

## 36. Itens calculados Traffic LAN

O código reconhece itens calculados do Zabbix:

```text
RX - Tráfego total dos switchs
TX - Tráfego total dos switchs
```

Quando encontrados, alimentam:

- mini gráfico RX;
- mini gráfico TX;
- card Tráfego Total.

Se não forem encontrados, o código usa como fallback a soma das séries da Query A.

---

## 37. Montagem das linhas da tabela

Depois de processar as séries, o código cria `rows`.

Para cada switch:

1. soma RX de todas as portas;
2. soma TX de todas as portas;
3. busca a porta monitorada;
4. calcula status da porta;
5. lê erros e descartes;
6. define se existe problema;
7. monta o objeto da linha.

---

## 38. Definição de porta problemática

```javascript
const portProblem = !port1 || portStatus.className !== 'up' || hasErrors || hasDiscards;
```

A porta é problemática quando:

- não foi encontrada;
- não está UP;
- possui erros;
- possui descartes.

---

## 39. Diagnóstico interno

O painel monta uma área de diagnóstico com:

- total de séries recebidas;
- switches identificados;
- porta padrão;
- mapa de portas por host;
- linhas de processamento.

Exemplos:

```text
[OK SWITCH] Switch HP 1920S SRV | switchStatus | ICMP ping = 1
[OK PORTA] Switch DELL PowerConnect 2848 | g1 [1] | bitsReceived = ...
[OK TOTAL RX ZABBIX] RX - Tráfego total dos switchs = ...
[IGNORADO] algum item
[SEM PORTA] métrica reconhecida sem porta
```

---

## 40. Cálculo dos KPIs

O código calcula:

```javascript
onlineCount
offlineCount
port1ProblemCount
totalTraffic
```

E atualiza os cards da tela:

- switches identificados;
- online;
- offline;
- portas com instabilidade;
- tráfego total.

---

## 41. Escolha da fonte RX/TX

```javascript
const rxSourceMap = calcRxHistory.size ? calcRxHistory : totalRxHistory;
const txSourceMap = calcTxHistory.size ? calcTxHistory : totalTxHistory;
```

### Responsabilidade

Prioriza os itens calculados do Zabbix.

Se eles não existirem, usa a soma feita no JavaScript.

---

## 42. Renderização dos mini gráficos RX/TX

O código transforma os históricos em pontos:

```javascript
const rxHistoryPoints = downsamplePoints(historyToPoints(rxSourceMap), 64);
const txHistoryPoints = downsamplePoints(historyToPoints(txSourceMap), 64);
```

Depois renderiza:

```javascript
renderSparkline(miniRxSvgEl, rxHistoryPoints, '#22c55e');
renderSparkline(miniTxSvgEl, txHistoryPoints, '#a78bfa');
```

---

## 43. Ordenação da tabela com `compareRows`

```javascript
function compareRows(a, b, key) {
  ...
}
```

Permite ordenar por:

- host;
- status;
- ICMP;
- RX;
- TX;
- porta;
- status da porta;
- download;
- upload;
- erros;
- descartes;
- última coleta.

---

## 44. Destaque visual com `counterClass`

```javascript
const counterClass = (value, threshold) => {
  ...
};
```

Define classe visual para erros e descartes:

```text
unknown
ok
warn
```

Os limites vêm de:

```javascript
cfg.errorWarnCount
cfg.discardWarnCount
```

---

## 45. Renderização da tabela com `renderTable`

```javascript
function renderTable() {
  ...
}
```

Responsável por:

1. aplicar filtros;
2. aplicar busca;
3. ordenar;
4. renderizar linhas;
5. exibir estado vazio;
6. aplicar classes visuais;
7. criar tooltip com dados brutos.

---

## 46. Filtros disponíveis

| Filtro | Critério |
|---|---|
| `all` | Todos |
| `online` | Switch online |
| `offline` | Switch offline |
| `portdown` | Porta monitorada diferente de UP |
| `errors` | Switches com erros |
| `discards` | Switches com descartes |
| `problems` | Switch offline ou porta com problema |

---

## 47. Busca textual

A busca considera:

- nome do switch;
- status;
- ICMP;
- porta;
- tráfego;
- erros;
- descartes.

Como usa `normalizeText`, a busca ignora acentos e maiúsculas/minúsculas.

---

## 48. Eventos de interação

O código registra eventos para:

- campo de busca;
- filtros;
- cabeçalhos da tabela.

A proteção abaixo evita registrar eventos duplicados em cada refresh:

```javascript
if (!root.dataset.bound) {
  ...
  root.dataset.bound = '1';
}
```

---

## 49. Renderização final

No final:

```javascript
renderTable();
```

Isso garante que a tabela seja atualizada após o processamento dos dados.

---

## 50. Resumo do pipeline completo

```text
1. Inicializa contexto do HTML Graphics
2. Lê custom properties
3. Captura elementos do HTML
4. Prepara estado de busca/filtro/ordenação
5. Define funções utilitárias
6. Percorre séries do Grafana/Zabbix
7. Identifica itens calculados Traffic LAN
8. Classifica métricas por regex
9. Identifica host
10. Extrai e normaliza porta
11. Agrupa dados por switch e porta
12. Soma RX/TX por switch
13. Seleciona a porta monitorada
14. Calcula status, erros e descartes
15. Monta linhas da tabela
16. Calcula KPIs
17. Desenha mini gráficos
18. Renderiza tabela
19. Atualiza diagnóstico
20. Registra eventos de interação
```

---

## 51. Pontos fortes

- Compatível com múltiplos fabricantes.
- Reconhece vários padrões de nomes de interfaces.
- Permite porta monitorada diferente por switch.
- Usa fallback quando itens calculados não existem.
- Possui diagnóstico interno.
- Protege renderização HTML com `safeText`.
- Permite busca, filtros e ordenação.
- Usa mini gráficos SVG sem depender de outro painel.
- Centraliza tráfego, ICMP, erros e descartes em uma única visão.

---

## 52. Pontos de atenção

### Regex amplo

A dashboard depende de regex para identificar os itens. Se um fabricante/template usar nomes muito diferentes, será necessário ajustar `metricFrom()` ou `portFrom()`.

### Possível duplicidade de tráfego

Se o Zabbix retornar dois itens equivalentes para a mesma porta, por exemplo:

```text
Bits received
Inbound traffic
```

o JavaScript pode somar os dois. O ideal é padronizar os templates ou filtrar melhor os itens da query.

### Performance

Evite `Item = /.*/` em ambientes grandes. Prefira um regex específico com os nomes realmente usados.

### Significado do tráfego total

Somar todas as portas mede atividade bruta da LAN, mas pode contar o mesmo tráfego mais de uma vez quando ele passa por uplinks e portas de acesso.

---

## 53. Manutenção recomendada no GitHub

Repositório:

```text
https://github.com/Marcusronney/monitoring_switchs
```

Estrutura sugerida:

```text
monitoring_switchs/
├─ README.md
├─ dashboards/
│  └─ dashboard-switches.json
├─ src/
│  ├─ html.html
│  ├─ style.css
│  └─ onRender.js
├─ docs/
│  └─ onrender-documentado.md
└─ scripts/
   └─ build-dashboard.js
```

---

## 54. Autor

**Marcus Rônney**  
GitHub: https://github.com/Marcusronney/monitoring_switchs  
LinkedIn: https://www.linkedin.com/in/marcus-r%C3%B4nney-627657bb/?skipRedirect=true  
Site: http://about.ronney.tech/

