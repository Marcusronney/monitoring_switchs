# Dashboard Grafana — Monitoramento de Switches com Zabbix + HTML Graphics

## 1. Objetivo da dashboard

Esta dashboard foi criada para monitorar switches de rede a partir de itens coletados no Zabbix via SNMP. O foco principal é dar uma visão operacional rápida sobre:

- status geral dos switches via ICMP;
- latência/perda ICMP;
- tráfego total RX/TX da LAN;
- tráfego RX/TX por switch;
- porta de cascateamento/uplink monitorada por switch;
- status operacional da porta monitorada;
- erros, descartes e CRC relacionados às interfaces;
- identificação de portas com instabilidade.

A dashboard é importada no Grafana por meio de um arquivo `.json` e utiliza o plugin **HTML Graphics** para renderizar a interface com HTML, CSS e JavaScript customizados.

---

## 2. Tecnologias usadas

A dashboard depende principalmente de:

| Componente | Uso |
|---|---|
| Grafana | Visualização e execução das queries |
| Zabbix datasource | Fonte de dados dos switches |
| Zabbix SNMP | Coleta das métricas dos equipamentos |
| HTML Graphics Panel | Renderização personalizada com HTML/CSS/JavaScript |
| JavaScript `onRender` | Processamento das séries retornadas pelo Grafana |
| HTML | Estrutura visual da tabela, cards e cabeçalho |
| CSS | Estilo visual, cores, layout, responsividade |
| Custom properties | Configurações editáveis sem alterar o JavaScript |

---

## 3. Visão geral da interface

A tela principal é composta por quatro áreas:

1. **Cabeçalho**
   - mini time series de RX e TX;
   - horário da última atualização;
   - legenda de status.

2. **Cards KPI**
   - total de switches identificados;
   - switches online;
   - switches offline;
   - portas com instabilidade;
   - tráfego total RX + TX.

3. **Barra de pesquisa e filtros**
   - pesquisa por switch, porta ou status;
   - filtros rápidos: todos, online, offline, porta down, erros, descartes e problemas.

4. **Tabela de switches**
   - status do switch;
   - ICMP;
   - total RX;
   - total TX;
   - porta monitorada;
   - status da porta;
   - download/upload da porta;
   - erros RX/TX;
   - descartes RX/TX;
   - última coleta.

---

## 4. Como a dashboard recebe os dados

A dashboard usa queries do datasource Zabbix.

### Query A — switches e portas

A query principal busca os itens dos switches no Zabbix.

Exemplo conceitual:

```text
Group = $zbx_group
Host  = $zbx_host
Item  = $zbx_item_filter
```

Essa query retorna séries de itens SNMP dos switches. O JavaScript do painel percorre essas séries e identifica o que cada item representa.

Exemplos de itens reconhecidos:

```text
ICMP ping
ICMP response time
ICMP loss

Interface Gi0/1(): Bits received
Interface Gi0/1(): Bits sent
Interface Gi0/1(): Operational status

Interface Gi0/1(): Inbound packets with errors
Interface Gi0/1(): Outbound packets with errors

Interface Gi0/1(): Inbound packets discarded
Interface Gi0/1(): Outbound packets discarded

Interface Gi0/1(): CRC/FCS errors
```

### Query B — tráfego total calculado no Zabbix

A query B é opcional, mas recomendada para melhorar performance e padronizar o cálculo do tráfego total.

Ela busca itens calculados em um host lógico, por exemplo:

```text
Host: Traffic LAN
Item: RX - Tráfego total dos switchs
Item: TX - Tráfego total dos switchs
```

Exemplo de filtro no campo Item:

```regex
/.*(RX - Tráfego total dos switchs|TX - Tráfego total dos switchs).*/
```

Quando a Query B retorna dados, os mini gráficos do cabeçalho e o card **Tráfego total** usam esses valores calculados pelo Zabbix.

Quando a Query B não retorna dados, o JavaScript usa como fallback a soma feita no próprio painel.

---

## 5. Como o JavaScript soma o tráfego

O JavaScript lê todas as séries retornadas pelo Grafana e classifica os itens por tipo de métrica.

Para tráfego, ele reconhece nomes/chaves que correspondem a:

```text
Bits received
Bits sent
incoming traffic
outgoing traffic
inbound traffic
outbound traffic
net.if.in
net.if.out
ifHCInOctets
ifHCOutOctets
ifInOctets
ifOutOctets
```

Depois, para cada switch, ele soma todas as portas reconhecidas.

Conceito simplificado:

```javascript
let totalRx = 0;
let totalTx = 0;

device.ports.forEach((port) => {
  const rx = metricValueToNumber(port.bitsReceived);
  const tx = metricValueToNumber(port.bitsSent);

  if (Number.isFinite(rx)) totalRx += rx;
  if (Number.isFinite(tx)) totalTx += tx;
});
```

Resultado:

```text
Total RX do switch = soma de RX das portas reconhecidas
Total TX do switch = soma de TX das portas reconhecidas
Tráfego total LAN = soma RX + TX
```

---

## 6. Diferença entre tráfego total e porta monitorada

A dashboard trabalha com dois conceitos diferentes:

### Tráfego total RX/TX do switch

Soma o tráfego de todas as portas reconhecidas daquele switch.

É exibido nas colunas:

```text
TOTAL RX
TOTAL TX
```

### Porta monitorada

É a porta principal de cascateamento/uplink ou porta crítica definida manualmente por switch.

É exibida nas colunas:

```text
PORTA MONITORADA
STATUS PORTA
DOWNLOAD PORTA
UPLOAD PORTA
ERROS RX
ERROS TX
DESCARTES RX
DESCARTES TX
```

Ou seja:

```text
Total RX/TX      = todas as portas reconhecidas
Download/Upload  = somente a porta monitorada
Erros/Descartes  = somente a porta monitorada
```

---

## 7. Configuração da porta monitorada por switch

A porta monitorada é definida nas **Custom properties** do painel HTML Graphics.

Exemplo:

```json
{
  "title": "Switches - Status e Porta Monitorada",
  "subtitle": "Status ICMP, latência/perda ICMP, tráfego total RX/TX e métricas da porta monitorada por switch",
  "monitoredPort": "1",
  "monitoredPortByHost": {
    "/Switch Call Center Mikrotik/": "15",
    "/Switch Cisco Catalyst 2960G/": "47",
    "/Switch D-Link Comercial/": "1",
    "/Switch Engenharia TP-Link/": "20",
    "/Switch ENGENHARIA/": "1",
    "/Switch Fundicao TP-Link/": "1",
    "/Switch HP 1920S SRV/": "2",
    "/Switch Mikrotik Girassol/": "15",
    "/Switch Semiacabado TP-Link 02/": "1",
    "/Switch Semiacabado TP-Link/": "10",
    "/Switch Servidor TP-Link/": "1"
  },
  "fallbackSpeedMbps": 1000,
  "usageWarnPct": 70,
  "usageCriticalPct": 90,
  "errorWarnCount": 1,
  "discardWarnCount": 1,
  "miniSeriesBucketMs": 60000
}
```

### Funcionamento

```text
monitoredPort = porta padrão
monitoredPortByHost = exceções por switch
```

Exemplo:

```json
"/Switch Cisco Catalyst 2960G/": "47"
```

Significa:

```text
Para o host cujo nome contém "Switch Cisco Catalyst 2960G",
a porta monitorada será a porta 47.
```

A chave pode ser:

- nome exato do host;
- parte do nome do host;
- regex entre barras.

Exemplos válidos:

```json
"/Cisco Catalyst/": "47"
"/Switch HP 1920S SRV/": "2"
"Switch Servidor TP-Link": "1"
```

---

## 8. Normalização de nomes de portas

Cada fabricante pode retornar nomes diferentes para as portas.

Exemplos:

```text
Port1
SFP1
g1
Gi0/1
GigabitEthernet0/1
gigabitEthernet 1/0/1
Interface 32()
TRK 1
```

O JavaScript normaliza esses nomes para conseguir comparar com a porta configurada.

Exemplos:

```text
Gi0/47              → 47
GigabitEthernet0/47 → 47
gigabitEthernet 1/0/20 → 20
g1                  → 1
Port15              → 15
Interface 32()      → 32
```

Essa normalização permite usar apenas o número da porta nas Custom properties.

---

## 9. Status do switch

O status do switch é baseado nos itens ICMP do Zabbix.

Itens reconhecidos:

```text
ICMP ping
icmpping
Ping status
status ping
ping availability
disponibilidade icmp
```

Mapeamento:

```text
1 = ONLINE
0 = OFFLINE
```

A dashboard também reconhece latência e perda ICMP:

```text
ICMP response time
icmppingsec
ICMP loss
icmppingloss
packet loss
```

---

## 10. Status da porta

O status da porta usa principalmente o `ifOperStatus`.

Mapeamento IF-MIB:

| Valor | Status |
|---:|---|
| 1 | UP |
| 2 | DOWN |
| 3 | Testing |
| 4 | Unknown |
| 5 | Dormant |
| 6 | Not present |
| 7 | Lower layer down |

Na tabela, esses estados são convertidos para classes visuais:

```text
UP                → verde
DOWN              → vermelho
Testing/Dormant   → alerta
Unknown/Sem dado  → cinza
```

---

## 11. Erros, descartes e CRC

A dashboard procura por nomes de itens relacionados a erros e descartes.

### Erros RX

Reconhece padrões como:

```text
Inbound packets with errors
In errors
ifInErrors
RX errors
CRC errors RX
erros entrada
```

### Erros TX

Reconhece padrões como:

```text
Outbound packets with errors
Out errors
ifOutErrors
TX errors
CRC errors TX
erros saída
```

### Descartes RX

Reconhece padrões como:

```text
Inbound packets discarded
In discards
ifInDiscards
RX discards
descartes RX
```

### Descartes TX

Reconhece padrões como:

```text
Outbound packets discarded
Out discards
ifOutDiscards
TX discards
descartes TX
```

### CRC/FCS

Dependendo do template do Zabbix, CRC pode vir como item próprio ou embutido em nomes como:

```text
CRC/FCS errors
CRC errors RX
CRC errors TX
dot3StatsFCSErrors
```

O objetivo é conseguir capturar variações de nomenclatura entre Cisco, HP, Dell, Mikrotik, TP-Link e outros fabricantes.

---

## 12. Importante: evitar duplicidade de tráfego

Alguns templates podem coletar a mesma porta em dois itens diferentes.

Exemplo:

```text
Interface Port1(): Bits received
Port1 - Inbound traffic
```

Ambos podem representar o RX da mesma porta.

Se a query do painel buscar os dois, o JavaScript pode reconhecer os dois como tráfego e somar duplicado.

Por isso, para tráfego, recomenda-se padronizar o filtro do painel para priorizar:

```text
Bits received
Bits sent
```

e evitar, se possível:

```text
Inbound traffic
Outbound traffic
Incoming traffic
Outgoing traffic
```

Filtro recomendado para reduzir duplicidade e carga:

```regex
/.*(ICMP ping|ICMP response time|ICMP loss|Bits received|Bits sent|Operational status|Inbound packets discarded|Outbound packets discarded|Inbound packets with errors|Outbound packets with errors|CRC|FCS|ifInErrors|ifOutErrors|ifInDiscards|ifOutDiscards).*/
```

---

## 13. Itens calculados no Zabbix para tráfego total

Para melhorar performance, o tráfego total pode ser calculado no Zabbix em vez de ser somado no JavaScript.

Exemplo de host lógico:

```text
Traffic LAN
```

Itens:

```text
RX - Tráfego total dos switchs
TX - Tráfego total dos switchs
Total - Tráfego dos switchs
```

Fórmulas recomendadas, considerando itens padronizados `net.if.in[*]` e `net.if.out[*]`:

### RX total

```text
sum(last_foreach(/*/net.if.in[*]?[group="switch"]))
```

### TX total

```text
sum(last_foreach(/*/net.if.out[*]?[group="switch"]))
```

### Total RX + TX

```text
last(/Traffic LAN/RX_trafego) + last(/Traffic LAN/TX_trafego)
```

Na dashboard, a Query B deve buscar:

```regex
/.*(RX - Tráfego total dos switchs|TX - Tráfego total dos switchs).*/
```

Assim:

```text
Mini gráfico RX → usa item calculado RX
Mini gráfico TX → usa item calculado TX
Card Tráfego total → RX + TX calculados
Tabela → continua usando dados detalhados dos switches
```

---

## 14. Como funciona o HTML do plugin HTML Graphics

O HTML define a estrutura visual estática do painel. Ele não faz cálculo. Ele apenas cria os elementos que o JavaScript vai preencher.

Principais blocos do HTML:

### Root

```html
<div id="sw1-root" class="sw48-root">
```

Elemento raiz do painel.

### Header

Contém:

- mini time series RX;
- mini time series TX;
- horário de atualização;
- legenda.

Elementos importantes:

```html
<div id="miniRxValue"></div>
<svg id="miniRxSvg"></svg>

<div id="miniTxValue"></div>
<svg id="miniTxSvg"></svg>

<div id="lastUpdated"></div>
```

O JavaScript usa esses IDs para preencher os valores e desenhar os gráficos.

### KPI cards

Elementos preenchidos pelo JavaScript:

```html
<strong id="kpiTotal"></strong>
<strong id="kpiOnline"></strong>
<strong id="kpiOffline"></strong>
<strong id="kpiPort1Problems"></strong>
<strong id="kpiTotalTraffic"></strong>
```

### Toolbar

Contém:

```html
<input id="tableSearch">
<button data-filter="online">
<button data-filter="offline">
<button data-filter="errors">
<button data-filter="discards">
```

O JavaScript escuta os eventos desses elementos para aplicar filtros.

### Tabela

O corpo da tabela é preenchido dinamicamente:

```html
<tbody id="switchRows"></tbody>
```

Se não houver dados, aparece:

```html
<div id="emptyState"></div>
```

### Diagnóstico

Área técnica para validar o que foi lido do Zabbix:

```html
<pre id="debugItems"></pre>
```

Ela exibe linhas como:

```text
[OK SWITCH] ...
[OK PORTA] ...
[OK TOTAL RX ZABBIX] ...
[IGNORADO] ...
[SEM PORTA] ...
```

---

## 15. Como funciona o JavaScript `onRender`

O `onRender` é executado sempre que o painel recebe novos dados do Grafana.

Ele faz todo o processamento da dashboard.

### Fluxo geral

```text
1. Lê customProperties
2. Lê séries retornadas pelo Grafana/Zabbix
3. Identifica host, item, porta e métrica
4. Agrupa dados por switch
5. Agrupa dados por porta
6. Calcula totais RX/TX
7. Seleciona porta monitorada de cada switch
8. Calcula status, erros e descartes
9. Renderiza cards KPI
10. Renderiza mini gráficos RX/TX
11. Renderiza tabela
12. Atualiza diagnóstico
```

---

## 16. Principais funções do `onRender`

### `vectorToArray`

Converte os vetores internos do Grafana em arrays JavaScript.

Usada para ler os valores retornados pelas séries.

### `lastNonEmpty`

Pega o último valor válido de uma série.

Evita usar `null`, vazio ou valor ausente.

### `metricValueToNumber`

Converte valores com unidades para número.

Exemplos:

```text
1.01 Gbps → 1010000000
500 Mbps → 500000000
30 Kbps  → 30000
```

### `durationToMs`

Converte latência para milissegundos.

Exemplos:

```text
0.001 → 1 ms
652 µs → 0.652 ms
5.7 ms → 5.7 ms
```

### `hostFrom`

Descobre qual host/switch pertence a uma série.

Ele tenta capturar o host por:

- labels do datasource;
- nome da série;
- display name do item;
- texto completo do campo.

### `metricFrom`

Classifica a série em uma métrica conhecida.

Possíveis retornos:

```text
switchStatus
icmpLatency
icmpLoss
bitsReceived
bitsSent
inErrors
outErrors
inDiscards
outDiscards
operStatus
interfaceType
speed
```

### `portFrom`

Extrai a porta a partir do nome do item.

Exemplos:

```text
Interface Gi0/10(): Bits received → Gi0/10
Interface g1(Onu Engenharia): Bits sent → g1(Onu Engenharia)
GigabitEthernet0/31 - Outbound packets → GigabitEthernet0/31
```

### `normalizePortKey`

Normaliza a porta para comparação.

Exemplos:

```text
Gi0/31 → 31
GigabitEthernet0/31 → 31
g1(Onu Engenharia) → 1
Port23 → 23
```

### `getMonitoredPortForHost`

Lê o mapa `monitoredPortByHost` e retorna a porta que deve ser monitorada para aquele switch.

### `setMetric`

Atualiza uma métrica no objeto da porta ou switch.

Ela evita sobrescrever valores válidos por valores vazios e permite manter o maior valor quando necessário.

### `renderSparkline`

Desenha os mini gráficos RX/TX em SVG no cabeçalho.

### `renderTable`

Renderiza a tabela final com filtros, ordenação e busca.

---

## 17. Diagnóstico interno

A área de diagnóstico é fundamental para manutenção.

Exemplos:

```text
[OK SWITCH] Switch HP 1920S SRV | switchStatus | ICMP ping = 1
[OK PORTA] Switch DELL PowerConnect 2848 | g1(Onu Engenharia) [1] | bitsReceived | Interface g1(Onu Engenharia): Bits received = 152880000
[OK TOTAL RX ZABBIX] Traffic LAN: RX - Tráfego total dos switchs = 1010000000
[IGNORADO] ICMP loss
[SEM PORTA] Switch Cisco Catalyst 2960G | algum item => bitsReceived
```

Interpretação:

| Mensagem | Significado |
|---|---|
| `[OK SWITCH]` | item associado ao switch, como ICMP |
| `[OK PORTA]` | item associado a uma porta |
| `[OK TOTAL RX ZABBIX]` | item calculado RX reconhecido |
| `[OK TOTAL TX ZABBIX]` | item calculado TX reconhecido |
| `[IGNORADO]` | item recebido, mas não classificado |
| `[SEM PORTA]` | métrica reconhecida, mas porta não foi extraída |

---

## 18. Filtros da tabela

A tabela suporta filtros rápidos:

| Filtro | Critério |
|---|---|
| Todos | mostra tudo |
| Online | switch com ICMP online |
| Offline | switch com ICMP offline |
| Porta DOWN | porta monitorada diferente de UP |
| Erros | portas com erros RX/TX |
| Descartes | portas com descartes RX/TX |
| Problemas | switch offline ou porta com instabilidade |

A pesquisa textual usa host, status, porta, ICMP, tráfego, erros e descartes.

---

## 19. Performance e otimização

O maior impacto de performance ocorre quando a query usa:

```regex
/.*/
```

Isso faz o Grafana buscar muitos itens do Zabbix e deixar o JavaScript filtrar no navegador.

Recomendações:

### Use filtro de itens mais específico

```regex
/.*(ICMP ping|ICMP response time|ICMP loss|Bits received|Bits sent|Operational status|Inbound packets discarded|Outbound packets discarded|Inbound packets with errors|Outbound packets with errors|CRC|FCS).*/
```

### Use itens calculados para total de tráfego

Use o host `Traffic LAN` com itens calculados RX/TX para evitar que o Grafana precise somar tudo no frontend.

### Ajuste o refresh

Recomendado:

```text
1m ou 2m
```

Evite refresh muito baixo, como 5s ou 10s, em dashboards com muitos itens SNMP.

### Separe dashboards por grupo

Se houver muitos switches, use grupos menores:

```text
switch-core
switch-servidores
switch-setores
switch-filiais
```

---

## 20. Limitações conhecidas

1. **Fabricantes diferentes usam nomes diferentes**
   - Por isso o JavaScript usa regex amplo.

2. **Pode haver itens duplicados**
   - Exemplo: `Bits received` e `Inbound traffic` para a mesma porta.

3. **Tráfego total de todas as portas pode superestimar o tráfego real**
   - O mesmo pacote pode passar por porta de acesso, uplink e core.

4. **Queue drops detalhados dependem do fabricante**
   - O padrão SNMP mais comum é usar `ifOutDiscards` como proxy.

5. **CRC pode depender de EtherLike-MIB**
   - Nem todos os switches expõem os mesmos OIDs.

6. **O HTML Graphics não é confortável para editar códigos longos**
   - Recomenda-se editar HTML/CSS/JS no VS Code e injetar no JSON da dashboard.

---

## 21. Estrutura recomendada para manter a dashboard como código

Para facilitar manutenção, recomenda-se separar os arquivos:

```text
dashboard-switches/
├─ dashboard.base.json
├─ src/
│  ├─ html.html
│  ├─ style.css
│  └─ onRender.js
├─ build-dashboard.js
└─ dashboard.final.json
```

O fluxo ideal:

```text
1. Exportar JSON do Grafana
2. Separar HTML, CSS e onRender.js
3. Editar no VS Code
4. Gerar dashboard.final.json
5. Importar no Grafana
```

---

## 22. Exemplo de script para montar o JSON

```javascript
const fs = require('fs');

const dashboard = JSON.parse(fs.readFileSync('./dashboard.base.json', 'utf8'));

const html = fs.readFileSync('./src/html.html', 'utf8');
const css = fs.readFileSync('./src/style.css', 'utf8');
const onRender = fs.readFileSync('./src/onRender.js', 'utf8');

const panel = dashboard.panels.find(
  (p) => p.type === 'gapit-htmlgraphics-panel'
);

if (!panel) {
  throw new Error('Painel HTML Graphics não encontrado.');
}

panel.options.html = html;
panel.options.css = css;
panel.options.onRender = onRender;

fs.writeFileSync(
  './dashboard.final.json',
  JSON.stringify(dashboard, null, 2),
  'utf8'
);

console.log('dashboard.final.json gerado com sucesso.');
```

---

## 23. Como validar se está funcionando

Após importar a dashboard:

1. Selecione o grupo dos switches.
2. Selecione `All` nos hosts ou um host específico.
3. Confira se aparecem switches na tabela.
4. Abra o diagnóstico.
5. Procure linhas:

```text
[OK SWITCH]
[OK PORTA]
[OK TOTAL RX ZABBIX]
[OK TOTAL TX ZABBIX]
```

6. Verifique se a porta monitorada mostra o alvo correto:

```text
alvo: 15
alvo: 47
alvo: 20
```

7. Confira se os cards batem com os itens calculados do Zabbix.

---

## 24. Resumo técnico

A dashboard funciona como uma camada de interpretação sobre os itens SNMP do Zabbix.

O Grafana apenas busca as séries.

O plugin HTML Graphics renderiza a interface.

O JavaScript `onRender` faz a inteligência:

```text
série do Zabbix
→ identifica host
→ identifica métrica
→ identifica porta
→ normaliza nome da porta
→ aplica porta monitorada
→ calcula totais
→ detecta instabilidade
→ renderiza tabela/cards/gráficos
```

O desenho permite que a dashboard funcione com múltiplos fabricantes, desde que os nomes dos itens retornados pelas MIBs estejam contemplados nas expressões regulares do JavaScript.

