# Terceira tela no iPad: plano de implementação

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Servir do PC Z3R0 um painel estilo iPadOS com métricas, sugestões e um conjunto fechado de ajustes aplicáveis pelo iPad.

**Architecture:** Um servidor Node sem dependências coleta métricas (`coleta.js`), executa uma lista fechada de ações (`acoes.js`) e entrega uma página única (`server.js` + `public/index.html`) com dados em Server-Sent Events. Roda como usuário comum, iniciado no login por uma tarefa agendada.

**Tech Stack:** Node 24 (`node:http`, `node:test`, `node:fs`, `node:child_process`), HTML, CSS e SVG sem bibliotecas, PowerShell para instalar.

**Especificação:** `docs/superpowers/specs/2026-11-04-terceira-tela-ipad-design.md`

**Regras de commit do projeto:** mensagem curta `tipo(escopo): resumo`, sem trailer `Co-Authored-By`. O repositório é público: nunca versionar `dados/`.

---

## Estrutura de arquivos

| Arquivo | Responsabilidade |
|---|---|
| `coleta.js` | Interpretar e coletar métricas; somente leitura |
| `acoes.js` | Lista fechada de ações, sugestões e log |
| `server.js` | HTTP, autenticação, rotas e stream |
| `public/index.html` | Interface completa |
| `public/manifest.json`, `public/icone.png` | Instalação na Tela de Início |
| `instalar.ps1` | Firewall e tarefa agendada |
| `test.js` | Testes automáticos |

---

### Task 1: Interpretar métricas

**Files:**
- Create: `coleta.js`
- Create: `test.js`

- [ ] **Step 1: Escrever os testes**

Crie `test.js`:

```js
'use strict';
const test = require('node:test');
const assert = require('node:assert/strict');
const coleta = require('./coleta');

test('usoCpu calcula a media entre leituras', () => {
  const antes = [{ user: 0, nice: 0, sys: 0, idle: 0, irq: 0 }];
  const depois = [{ user: 20, nice: 0, sys: 5, idle: 75, irq: 0 }];
  assert.equal(coleta.usoCpu(antes, depois), 25);
});

test('usoCpu devolve zero sem tempo decorrido', () => {
  const t = [{ user: 1, nice: 0, sys: 1, idle: 1, irq: 0 }];
  assert.equal(coleta.usoCpu(t, t), 0);
});

test('parseGpu le a linha real do nvidia-smi', () => {
  assert.deepEqual(coleta.parseGpu('6, 51, 38.30, 4209, 12227'), {
    uso: 6, temp: 51, watts: 38.3, vramUsada: 4209, vramTotal: 12227,
  });
});

test('parseGpu troca [N/A] por null', () => {
  assert.equal(coleta.parseGpu('6, 51, [N/A], 4209, 12227').watts, null);
});

test('parseGpu rejeita linha incompleta', () => {
  assert.equal(coleta.parseGpu('erro qualquer'), null);
});

// Amostra sintetica no formato documentado pelo Rainmeter. Trocar por saida real
// quando o HWiNFO for instalado.
const HWINFO = [
  'HKEY_CURRENT_USER\\SOFTWARE\\HWiNFO64\\VSB',
  '    Sensor0    REG_SZ    CPU [#0]: AMD Ryzen 7 5700X',
  '    Label0    REG_SZ    CPU (Tctl/Tdie)',
  '    Value0    REG_SZ    52.3 °C',
  '    ValueRaw0    REG_SZ    52.3',
  '    Label1    REG_SZ    Average Effective Clock',
  '    ValueRaw1    REG_SZ    3650,5',
  '    Label2    REG_SZ    CPU Fan',
  '    ValueRaw2    REG_SZ    900',
  '',
].join('\r\n');

test('parseHwinfo encontra temperatura, frequencia e ventoinhas', () => {
  assert.deepEqual(coleta.parseHwinfo(HWINFO), {
    cpuTemp: 52.3,
    cpuClock: 3650.5,
    ventoinhas: [{ rotulo: 'CPU Fan', valor: 900 }],
  });
});

test('parseHwinfo sem sensores devolve vazio', () => {
  assert.deepEqual(coleta.parseHwinfo(''), { cpuTemp: null, cpuClock: null, ventoinhas: [] });
});
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `node --test test.js`
Expected: FAIL com `Cannot find module './coleta'`.

- [ ] **Step 3: Implementar**

Crie `coleta.js`:

```js
'use strict';

function usoCpu(antes, depois) {
  let ocioso = 0;
  let total = 0;
  const soma = (t) => t.user + t.nice + t.sys + t.idle + t.irq;
  depois.forEach((t, i) => {
    ocioso += t.idle - antes[i].idle;
    total += soma(t) - soma(antes[i]);
  });
  return total > 0 ? Math.round(100 * (1 - ocioso / total)) : 0;
}

function parseGpu(linha) {
  const campos = linha.split(',').map((c) => Number(c.trim()));
  if (campos.length !== 5) return null;
  const [uso, temp, watts, vramUsada, vramTotal] = campos.map((n) => (Number.isNaN(n) ? null : n));
  return { uso, temp, watts, vramUsada, vramTotal };
}

function parseHwinfo(saida) {
  const rotulos = {};
  const valores = {};
  for (const linha of saida.split(/\r?\n/)) {
    const m = linha.match(/^\s+(Label|ValueRaw)(\d+)\s+REG_SZ\s+(.*?)\s*$/);
    if (m) (m[1] === 'Label' ? rotulos : valores)[m[2]] = m[3];
  }
  const sensores = Object.keys(rotulos)
    .map((i) => ({ rotulo: rotulos[i], valor: Number(String(valores[i]).replace(',', '.')) }))
    .filter((s) => !Number.isNaN(s.valor));
  const achar = (re) => sensores.find((s) => re.test(s.rotulo))?.valor ?? null;
  return {
    cpuTemp: achar(/Tctl\/Tdie/),
    cpuClock: achar(/Core Clocks|Effective Clock/),
    ventoinhas: sensores.filter((s) => /fan|ventoinha/i.test(s.rotulo)),
  };
}

module.exports = { usoCpu, parseGpu, parseHwinfo };
```

- [ ] **Step 4: Rodar e ver passar**

Run: `node --test test.js`
Expected: 7 testes passando.

- [ ] **Step 5: Commit**

```bash
git add coleta.js test.js
git commit -m "feat(coleta): interpretar CPU, GPU e HWiNFO"
```

---

### Task 2: Coletar métricas continuamente

**Files:**
- Modify: `coleta.js`
- Modify: `test.js`

- [ ] **Step 1: Escrever os testes**

Acrescente ao fim de `test.js`:

```js
const fs = require('node:fs/promises');
const os = require('node:os');
const path = require('node:path');

test('tamanhoPasta soma arquivos e ignora juncoes', async () => {
  const dir = await fs.mkdtemp(path.join(os.tmpdir(), 'tela-'));
  const fora = await fs.mkdtemp(path.join(os.tmpdir(), 'tela-'));
  await fs.mkdir(path.join(dir, 'sub'));
  await fs.writeFile(path.join(dir, 'a.bin'), Buffer.alloc(100));
  await fs.writeFile(path.join(dir, 'sub', 'b.bin'), Buffer.alloc(50));
  await fs.writeFile(path.join(fora, 'c.bin'), Buffer.alloc(9000));
  await fs.symlink(fora, path.join(dir, 'juncao'), 'junction');
  assert.deepEqual(await coleta.tamanhoPasta(dir), { bytes: 150, parcial: false });
});

test('tamanhoPasta marca como parcial o que nao consegue ler', async () => {
  assert.deepEqual(await coleta.tamanhoPasta('C:/nao/existe'), { bytes: 0, parcial: true });
});

test('criarPastas calcula uma vez e usa o cache', async () => {
  let chamadas = 0;
  const pastas = coleta.criarPastas(async () => {
    chamadas += 1;
    return { pastas: [{ caminho: 'C:/x', bytes: 1 }], parcial: false };
  });
  assert.equal(pastas.ler().estado, 'calculando');
  assert.equal(pastas.ler().estado, 'calculando');
  await new Promise((r) => setImmediate(r));
  assert.equal(pastas.ler().estado, 'pronto');
  assert.equal(chamadas, 1);
});

test('iniciarColeta entrega uma leitura com os campos esperados', async () => {
  const c = coleta.iniciarColeta();
  const l = c.leitura();
  c.parar();
  assert.equal(typeof l.nome, 'string');
  assert.ok(l.ram.total > 0);
  assert.ok(Array.isArray(l.cpu.historico));
});
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `node --test test.js`
Expected: FAIL com `coleta.tamanhoPasta is not a function`.

- [ ] **Step 3: Implementar**

No topo de `coleta.js`, depois de `'use strict';`, adicione:

```js
const os = require('node:os');
const fs = require('node:fs/promises');
const path = require('node:path');
const readline = require('node:readline');
const { spawn, execFile } = require('node:child_process');
const { promisify } = require('node:util');

const exec = promisify(execFile);
const DISCOS = [
  { letra: 'C', nome: 'NVMe' },
  { letra: 'D', nome: 'SSD USB' },
  { letra: 'E', nome: 'HD' },
];
const CHAVE_HWINFO = 'HKCU\\SOFTWARE\\HWiNFO64\\VSB';
const CAMPOS_GPU = 'utilization.gpu,temperature.gpu,power.draw,memory.used,memory.total';
const HISTORICO = 30;
const CACHE_PASTAS_MS = 10 * 60 * 1000;
```

Antes de `module.exports`, adicione:

```js
async function lerDiscos() {
  return Promise.all(DISCOS.map(async (d) => {
    try {
      const s = await fs.statfs(`${d.letra}:/`);
      return { ...d, total: s.blocks * s.bsize, livre: s.bavail * s.bsize };
    } catch {
      return { ...d, total: null, livre: null };
    }
  }));
}

async function tamanhoPasta(dir) {
  let bytes = 0;
  let parcial = false;
  let itens;
  try {
    itens = await fs.readdir(dir, { withFileTypes: true });
  } catch {
    return { bytes, parcial: true };
  }
  for (const item of itens) {
    if (item.isSymbolicLink()) continue;
    const caminho = path.join(dir, item.name);
    if (item.isDirectory()) {
      const sub = await tamanhoPasta(caminho);
      bytes += sub.bytes;
      parcial ||= sub.parcial;
    } else {
      try {
        bytes += (await fs.lstat(caminho)).size;
      } catch {
        parcial = true;
      }
    }
  }
  return { bytes, parcial };
}

async function subpastas(raiz) {
  const itens = await fs.readdir(raiz, { withFileTypes: true }).catch(() => []);
  return itens.filter((i) => i.isDirectory() && !i.isSymbolicLink()).map((i) => path.join(raiz, i.name));
}

async function calcularMaioresPastas() {
  const usuarios = path.join('C:/', 'Users');
  const alvos = [
    ...(await subpastas('C:/')).filter((p) => p !== usuarios),
    ...(await subpastas(os.homedir())),
  ];
  const pastas = [];
  let parcial = false;
  for (const caminho of alvos) {
    const r = await tamanhoPasta(caminho);
    pastas.push({ caminho, bytes: r.bytes });
    parcial ||= r.parcial;
  }
  pastas.sort((a, b) => b.bytes - a.bytes);
  return { pastas: pastas.slice(0, 10), parcial };
}

function criarPastas(calcular = calcularMaioresPastas) {
  let cache = null;
  let quando = 0;
  let emCurso = null;
  return {
    ler() {
      if (cache && Date.now() - quando < CACHE_PASTAS_MS) return { estado: 'pronto', ...cache };
      emCurso ??= calcular()
        .then((r) => { cache = r; quando = Date.now(); })
        .catch(() => {})
        .finally(() => { emCurso = null; });
      return { estado: 'calculando' };
    },
  };
}

function iniciarColeta() {
  const estado = {
    nome: os.hostname(),
    cpu: { uso: 0, modelo: os.cpus()[0].model.trim(), historico: [] },
    ram: { total: os.totalmem(), usada: os.totalmem() - os.freemem() },
    gpu: null,
    hwinfo: null,
    discos: [],
  };
  let ativo = true;
  let tempos = os.cpus().map((c) => c.times);
  let processoGpu = null;
  let reinicioGpu = null;
  let proximoHwinfo = null;

  function medir() {
    const agora = os.cpus().map((c) => c.times);
    estado.cpu.uso = usoCpu(tempos, agora);
    tempos = agora;
    estado.cpu.historico = [...estado.cpu.historico, estado.cpu.uso].slice(-HISTORICO);
    estado.ram.usada = os.totalmem() - os.freemem();
  }

  async function atualizarDiscos() {
    estado.discos = await lerDiscos();
  }

  function iniciarGpu() {
    if (!ativo) return;
    const args = [`--query-gpu=${CAMPOS_GPU}`, '--format=csv,noheader,nounits', '-l', '2'];
    processoGpu = spawn('nvidia-smi', args, { windowsHide: true, stdio: ['ignore', 'pipe', 'ignore'] });
    readline.createInterface({ input: processoGpu.stdout }).on('line', (linha) => {
      estado.gpu = parseGpu(linha);
    });
    processoGpu.on('error', () => {});
    processoGpu.on('close', () => {
      estado.gpu = null;
      if (ativo) reinicioGpu = setTimeout(iniciarGpu, 5000);
    });
  }

  async function lerHwinfo() {
    let espera = 30000;
    try {
      const { stdout } = await exec('reg', ['query', CHAVE_HWINFO], { windowsHide: true });
      estado.hwinfo = parseHwinfo(stdout);
      espera = 5000;
    } catch {
      estado.hwinfo = null;
    }
    if (ativo) proximoHwinfo = setTimeout(lerHwinfo, espera);
  }

  const pulsos = [setInterval(medir, 2000), setInterval(atualizarDiscos, 30000)];
  atualizarDiscos();
  iniciarGpu();
  lerHwinfo();

  return {
    leitura: () => estado,
    parar() {
      ativo = false;
      pulsos.forEach(clearInterval);
      clearTimeout(reinicioGpu);
      clearTimeout(proximoHwinfo);
      processoGpu?.kill();
    },
  };
}
```

Troque `module.exports` por:

```js
module.exports = {
  usoCpu, parseGpu, parseHwinfo, lerDiscos, tamanhoPasta, criarPastas, iniciarColeta,
};
```

- [ ] **Step 4: Rodar e ver passar**

Run: `node --test test.js`
Expected: 11 testes passando.

- [ ] **Step 5: Conferir dados reais**

Run, a partir da raiz do projeto:

```bash
node -e "const k=require('./coleta').iniciarColeta();setTimeout(()=>{console.log(JSON.stringify(k.leitura(),null,1));k.parar()},5000)"
```

Expected: `cpu.uso` maior que zero, `gpu.vramTotal` igual a `12227`, três discos com valores e `hwinfo: null`. O processo encerra sozinho.

- [ ] **Step 6: Commit**

```bash
git add coleta.js test.js
git commit -m "feat(coleta): leitura continua de metricas e pastas"
```

---

### Task 3: Ações de inicialização

**Files:**
- Create: `acoes.js`
- Modify: `test.js`

- [ ] **Step 1: Escrever os testes**

Acrescente ao fim de `test.js`:

```js
const acoes = require('./acoes');

const APROVADOS = [
  'HKEY_CURRENT_USER\\Software\\Microsoft\\Windows\\CurrentVersion\\Explorer\\StartupApproved\\Run',
  '    Docker Desktop    REG_BINARY    030000000000000000000000',
  '    Discord    REG_BINARY    020000000000000000000000',
  '    LGHUB    REG_BINARY    03000000BF2478B7B15CDD01',
  '',
].join('\r\n');

test('parseBinarios le os valores reais do registro', () => {
  const m = acoes.parseBinarios(APROVADOS);
  assert.equal(m.get('Docker Desktop'), '030000000000000000000000');
  assert.equal(m.get('LGHUB'), '03000000BF2478B7B15CDD01');
});

test('itemLigado segue o bit do primeiro byte e assume ligado sem valor', () => {
  assert.equal(acoes.itemLigado('030000000000000000000000'), false);
  assert.equal(acoes.itemLigado('070000000000000000000000'), false);
  assert.equal(acoes.itemLigado('020000000000000000000000'), true);
  assert.equal(acoes.itemLigado('060000000000000000000000'), true);
  assert.equal(acoes.itemLigado(undefined), true);
});

test('definirInicializacao rejeita item fora da lista', async () => {
  await assert.rejects(acoes.definirInicializacao('SecurityHealth', false), { status: 400 });
  await assert.rejects(acoes.definirInicializacao('Riot Vanguard', false), { status: 400 });
});

test('definirInicializacao exige booleano', async () => {
  await assert.rejects(acoes.definirInicializacao('discord', 'sim'), { status: 400 });
});
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `node --test test.js`
Expected: FAIL com `Cannot find module './acoes'`.

- [ ] **Step 3: Implementar**

Crie `acoes.js`:

```js
'use strict';
const os = require('node:os');
const fs = require('node:fs/promises');
const path = require('node:path');
const { execFile } = require('node:child_process');
const { promisify } = require('node:util');

const run = promisify(execFile);
const exec = (cmd, args) => run(cmd, args, { windowsHide: true });

const BASE = 'HKCU\\Software\\Microsoft\\Windows\\CurrentVersion';
const APROVADOS = `${BASE}\\Explorer\\StartupApproved`;
const PASTA_INICIALIZAR = path.join(os.homedir(), 'AppData/Roaming/Microsoft/Windows/Start Menu/Programs/Startup');

// Lista fechada. Riot Vanguard e SecurityHealth ficam de fora de proposito.
const ITENS = [
  { id: 'ollama', nome: 'Ollama', chave: 'StartupFolder', valor: 'Ollama.lnk' },
  { id: 'openclaw', nome: 'OpenClaw Gateway', chave: 'StartupFolder', valor: 'OpenClaw Gateway.cmd' },
  { id: 'steam', nome: 'Steam', chave: 'Run', valor: 'Steam' },
  { id: 'discord', nome: 'Discord', chave: 'Run', valor: 'Discord' },
  { id: 'riot', nome: 'Riot Client', chave: 'Run', valor: 'RiotClient' },
  { id: 'docker', nome: 'Docker Desktop', chave: 'Run', valor: 'Docker Desktop' },
  { id: 'proton', nome: 'Proton VPN', chave: 'Run', valor: 'Proton VPN' },
  { id: 'lghub', nome: 'Logitech G HUB', chave: 'Run', valor: 'LGHUB' },
];

function erro400(mensagem) {
  return Object.assign(new Error(mensagem), { status: 400 });
}

function parseValores(saida) {
  const valores = new Map();
  for (const linha of saida.split(/\r?\n/)) {
    const m = linha.match(/^\s{4}(.+?)\s{4}(REG_\w+)\s{4}(.*)$/);
    if (m) valores.set(m[1], m[3].trim());
  }
  return valores;
}

const parseBinarios = parseValores;

function itemLigado(hex) {
  return !hex || (parseInt(hex.slice(0, 2), 16) & 1) === 0;
}

async function consultar(chave) {
  try {
    return parseValores((await exec('reg', ['query', chave])).stdout);
  } catch {
    return new Map();
  }
}

async function lerInicializacao() {
  const [aprovadosRun, aprovadosPasta, run, pasta] = await Promise.all([
    consultar(`${APROVADOS}\\Run`),
    consultar(`${APROVADOS}\\StartupFolder`),
    consultar(`${BASE}\\Run`),
    fs.readdir(PASTA_INICIALIZAR).catch(() => []),
  ]);
  return ITENS
    .filter((i) => (i.chave === 'Run' ? run.has(i.valor) : pasta.includes(i.valor)))
    .map((i) => ({
      id: i.id,
      nome: i.nome,
      ligado: itemLigado((i.chave === 'Run' ? aprovadosRun : aprovadosPasta).get(i.valor)),
    }));
}

async function definirInicializacao(id, ligado) {
  const item = ITENS.find((i) => i.id === id);
  if (!item) throw erro400('Item desconhecido');
  if (typeof ligado !== 'boolean') throw erro400('Estado inválido');
  const dados = `${ligado ? '02' : '03'}${'0'.repeat(22)}`;
  await exec('reg', ['add', `${APROVADOS}\\${item.chave}`, '/v', item.valor, '/t', 'REG_BINARY', '/d', dados, '/f']);
}

module.exports = { parseBinarios, itemLigado, lerInicializacao, definirInicializacao };
```

- [ ] **Step 4: Rodar e ver passar**

Run: `node --test test.js`
Expected: 15 testes passando.

- [ ] **Step 5: Conferir leitura real**

Run, a partir da raiz do projeto:

```bash
node -e "require('./acoes').lerInicializacao().then(console.log)"
```

Expected: Ollama, Discord e Proton VPN com `ligado: true`; Docker Desktop, Steam, Riot Client, Logitech G HUB e OpenClaw Gateway com `ligado: false`. Não execute `definirInicializacao` de verdade aqui.

- [ ] **Step 6: Commit**

```bash
git add acoes.js test.js
git commit -m "feat(acoes): ler e alterar itens de inicializacao"
```

---

### Task 4: Plano de energia

**Files:**
- Modify: `acoes.js`
- Modify: `test.js`

- [ ] **Step 1: Escrever os testes**

Acrescente ao fim de `test.js`:

```js
const PLANOS = [
  'Existing Power Schemes (* Active)',
  '-----------------------------------',
  'Power Scheme GUID: 381b4222-f694-41f0-9685-ff5bb260df2e  (Balanced) *',
  'Power Scheme GUID: 8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c  (High performance)',
  'Power Scheme GUID: a1841308-3541-4fab-bc81-f71556f20b4a  (Power saver)',
].join('\r\n');

test('parsePlanos identifica disponiveis e ativo pelo GUID', () => {
  assert.deepEqual(acoes.parsePlanos(PLANOS), {
    ativo: 'equilibrado',
    disponiveis: ['equilibrado', 'desempenho'],
  });
});

test('parsePlanos oculta Alto desempenho quando nao existe', () => {
  const r = acoes.parsePlanos('Power Scheme GUID: 381b4222-f694-41f0-9685-ff5bb260df2e  (Balanceado) *');
  assert.deepEqual(r, { ativo: 'equilibrado', disponiveis: ['equilibrado'] });
});

test('parsePlanos com outro plano ativo devolve ativo null', () => {
  const r = acoes.parsePlanos('Power Scheme GUID: a1841308-3541-4fab-bc81-f71556f20b4a  (Power saver) *');
  assert.equal(r.ativo, null);
});

test('definirEnergia rejeita plano fora da lista', async () => {
  await assert.rejects(acoes.definirEnergia('turbo'), { status: 400 });
});
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `node --test test.js`
Expected: FAIL com `acoes.parsePlanos is not a function`.

- [ ] **Step 3: Implementar**

Em `acoes.js`, antes de `module.exports`, adicione:

```js
const PLANOS = {
  equilibrado: '381b4222-f694-41f0-9685-ff5bb260df2e',
  desempenho: '8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c',
};

function parsePlanos(saida) {
  const linhas = saida.split(/\r?\n/);
  const disponiveis = Object.keys(PLANOS).filter((p) => linhas.some((l) => l.includes(PLANOS[p])));
  const ativo = disponiveis.find((p) => linhas.some((l) => l.includes(PLANOS[p]) && l.trim().endsWith('*'))) ?? null;
  return { ativo, disponiveis };
}

async function lerEnergia() {
  return parsePlanos((await exec('powercfg', ['/list'])).stdout);
}

async function definirEnergia(plano) {
  if (!Object.hasOwn(PLANOS, plano)) throw erro400('Plano desconhecido');
  await exec('powercfg', ['/setactive', PLANOS[plano]]);
}
```

Troque `module.exports` por:

```js
module.exports = {
  parseBinarios, itemLigado, lerInicializacao, definirInicializacao,
  parsePlanos, lerEnergia, definirEnergia,
};
```

- [ ] **Step 4: Rodar e ver passar**

Run: `node --test test.js`
Expected: 19 testes passando.

- [ ] **Step 5: Commit**

```bash
git add acoes.js test.js
git commit -m "feat(acoes): alternar plano de energia"
```

---

### Task 5: Limpeza de temporários e log

**Files:**
- Modify: `acoes.js`
- Modify: `test.js`

- [ ] **Step 1: Escrever os testes**

Acrescente ao fim de `test.js`:

```js
test('listarAntigos pega so arquivos antigos e nao entra em juncoes', async () => {
  const dir = await fs.mkdtemp(path.join(os.tmpdir(), 'tela-'));
  const fora = await fs.mkdtemp(path.join(os.tmpdir(), 'tela-'));
  const velho = new Date(Date.now() - 8 * 86400000);
  await fs.mkdir(path.join(dir, 'sub'));
  await fs.writeFile(path.join(dir, 'novo.tmp'), 'x');
  await fs.writeFile(path.join(dir, 'sub', 'velho.tmp'), 'abc');
  await fs.utimes(path.join(dir, 'sub', 'velho.tmp'), velho, velho);
  await fs.writeFile(path.join(fora, 'protegido.tmp'), 'zzz');
  await fs.utimes(path.join(fora, 'protegido.tmp'), velho, velho);
  await fs.symlink(fora, path.join(dir, 'juncao'), 'junction');

  const lista = await acoes.listarAntigos(dir);
  assert.deepEqual(lista.map((a) => path.basename(a.caminho)), ['velho.tmp']);
  assert.equal(lista[0].bytes, 3);
});

test('limparTemp apaga antigos, preserva novos e o que esta fora', async () => {
  const dir = await fs.mkdtemp(path.join(os.tmpdir(), 'tela-'));
  const velho = new Date(Date.now() - 8 * 86400000);
  await fs.writeFile(path.join(dir, 'novo.tmp'), 'x');
  await fs.writeFile(path.join(dir, 'velho.tmp'), 'abc');
  await fs.utimes(path.join(dir, 'velho.tmp'), velho, velho);

  assert.deepEqual(await acoes.limparTemp(dir), { apagados: 1, bytes: 3, ignorados: 0 });
  assert.deepEqual(await fs.readdir(dir), ['novo.tmp']);
});

test('registrar grava uma linha por acao', async () => {
  const dir = await fs.mkdtemp(path.join(os.tmpdir(), 'tela-'));
  await acoes.registrar(dir, 'energia', 'desempenho', 'ok');
  const texto = await fs.readFile(path.join(dir, 'acoes.log'), 'utf8');
  assert.match(texto, /^\d{4}-\d{2}-\d{2}T.* energia desempenho ok\n$/);
});
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `node --test test.js`
Expected: FAIL com `acoes.listarAntigos is not a function`.

- [ ] **Step 3: Implementar**

Em `acoes.js`, antes de `module.exports`, adicione:

```js
const IDADE_TEMP_MS = 7 * 86400000;

async function listarAntigos(dir = os.tmpdir()) {
  const limite = Date.now() - IDADE_TEMP_MS;
  const achados = [];
  const itens = await fs.readdir(dir, { withFileTypes: true }).catch(() => []);
  for (const item of itens) {
    if (item.isSymbolicLink()) continue;
    const caminho = path.join(dir, item.name);
    if (item.isDirectory()) {
      achados.push(...await listarAntigos(caminho));
      continue;
    }
    try {
      const s = await fs.lstat(caminho);
      if (s.mtimeMs < limite) achados.push({ caminho, bytes: s.size });
    } catch {
      // Arquivo sumiu ou sem acesso: fica fora da limpeza.
    }
  }
  return achados;
}

async function previaTemp(dir) {
  const lista = await listarAntigos(dir);
  return { arquivos: lista.length, bytes: lista.reduce((t, a) => t + a.bytes, 0) };
}

async function limparTemp(dir) {
  let apagados = 0;
  let bytes = 0;
  let ignorados = 0;
  for (const arquivo of await listarAntigos(dir)) {
    try {
      await fs.unlink(arquivo.caminho);
      apagados += 1;
      bytes += arquivo.bytes;
    } catch {
      ignorados += 1;
    }
  }
  return { apagados, bytes, ignorados };
}

async function registrar(dir, acao, parametro, resultado) {
  await fs.mkdir(dir, { recursive: true });
  await fs.appendFile(path.join(dir, 'acoes.log'), `${new Date().toISOString()} ${acao} ${parametro} ${resultado}\n`);
}
```

Troque `module.exports` por:

```js
module.exports = {
  parseBinarios, itemLigado, lerInicializacao, definirInicializacao,
  parsePlanos, lerEnergia, definirEnergia,
  listarAntigos, previaTemp, limparTemp, registrar,
};
```

- [ ] **Step 4: Rodar e ver passar**

Run: `node --test test.js`
Expected: 22 testes passando.

- [ ] **Step 5: Commit**

```bash
git add acoes.js test.js
git commit -m "feat(acoes): limpar temporarios antigos e registrar acoes"
```

---

### Task 6: Sugestões

**Files:**
- Modify: `acoes.js`
- Modify: `test.js`

- [ ] **Step 1: Escrever os testes**

Acrescente ao fim de `test.js`:

```js
const GIB = 1024 ** 3;
const BASE_SUG = {
  discos: [{ letra: 'C', nome: 'NVMe', total: 476 * GIB, livre: 82 * GIB }],
  inicializacao: [{ id: 'ollama', nome: 'Ollama', ligado: true }, { id: 'docker', nome: 'Docker Desktop', ligado: false }],
  energia: { ativo: 'equilibrado', disponiveis: ['equilibrado', 'desempenho'] },
  temp: { arquivos: 16183, bytes: 0.5 * GIB },
};

test('sugestoes refletem o estado real do PC', () => {
  const ids = acoes.sugestoes(BASE_SUG).map((s) => s.id);
  assert.ok(ids.includes('disco-C'));
  assert.ok(ids.includes('inicio-ollama'));
  assert.ok(!ids.includes('inicio-docker'));
  assert.ok(ids.includes('energia'));
  assert.ok(ids.includes('temp'));
  assert.ok(ids.includes('ram'));
});

test('sugestoes somem quando nao ha o que fazer', () => {
  const ids = acoes.sugestoes({
    discos: [{ letra: 'C', nome: 'NVMe', total: 100, livre: 50 }, { letra: 'E', nome: 'HD', total: null, livre: null }],
    inicializacao: [{ id: 'ollama', nome: 'Ollama', ligado: false }],
    energia: { ativo: 'desempenho', disponiveis: ['equilibrado', 'desempenho'] },
    temp: { arquivos: 1, bytes: 10 },
  }).map((s) => s.id);
  assert.deepEqual(ids, ['ram', 'usb', 'hd']);
});

test('estado junta acoes e sugestoes sem depender de discos validos', async () => {
  const e = await acoes.estado([], os.tmpdir());
  assert.ok(Array.isArray(e.inicializacao));
  assert.ok(Array.isArray(e.sugestoes));
  assert.equal(typeof e.temp.bytes, 'number');
});
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `node --test test.js`
Expected: FAIL com `acoes.sugestoes is not a function`.

- [ ] **Step 3: Implementar**

Em `acoes.js`, antes de `module.exports`, adicione:

```js
const MINIMO_TEMP = 256 * 1024 ** 2;
const ITENS_PESADOS = ['ollama', 'docker'];

function formatarGb(bytes) {
  return `${(bytes / 1024 ** 3).toFixed(1).replace('.', ',')} GB`;
}

function sugestoes({ discos, inicializacao, energia, temp }) {
  const lista = [];
  for (const d of discos) {
    if (d.total && d.livre / d.total < 0.2) {
      lista.push({
        id: `disco-${d.letra}`,
        titulo: `${d.letra}: com pouco espaço`,
        texto: `${formatarGb(d.livre)} livres. Veja as maiores pastas em Armazenamento.`,
      });
    }
  }
  for (const i of inicializacao) {
    if (i.ligado && ITENS_PESADOS.includes(i.id)) {
      lista.push({
        id: `inicio-${i.id}`,
        titulo: `${i.nome} sobe com o Windows`,
        texto: 'Desligue na inicialização se não usa sempre.',
        controle: { tipo: 'inicio', id: i.id },
      });
    }
  }
  if (energia.ativo === 'equilibrado' && energia.disponiveis.includes('desempenho')) {
    lista.push({
      id: 'energia',
      titulo: 'Plano de energia Equilibrado',
      texto: 'Alto desempenho pode ajudar pouco e aumenta o consumo parado.',
      controle: { tipo: 'energia' },
    });
  }
  if (temp.bytes >= MINIMO_TEMP) {
    lista.push({
      id: 'temp',
      titulo: 'Temporários antigos',
      texto: `${formatarGb(temp.bytes)} com mais de 7 dias.`,
      controle: { tipo: 'temp' },
    });
  }
  lista.push(
    { id: 'ram', titulo: 'Memória a 2667 MHz', texto: 'Se o seu kit suportar mais, ative XMP ou DOCP na BIOS.' },
    { id: 'usb', titulo: 'D: é um SSD SATA por USB', texto: 'Evite instalar jogos nele.' },
    { id: 'hd', titulo: 'E: é um disco rígido', texto: 'Use só para arquivos.' },
  );
  return lista;
}

async function estado(discos, dirTemp) {
  const [inicializacao, energia, temp] = await Promise.all([
    lerInicializacao(), lerEnergia(), previaTemp(dirTemp),
  ]);
  return {
    inicializacao, energia, temp,
    sugestoes: sugestoes({ discos, inicializacao, energia, temp }),
  };
}
```

Troque `module.exports` por:

```js
module.exports = {
  parseBinarios, itemLigado, lerInicializacao, definirInicializacao,
  parsePlanos, lerEnergia, definirEnergia,
  listarAntigos, previaTemp, limparTemp, registrar,
  sugestoes, estado,
};
```

- [ ] **Step 4: Rodar e ver passar**

Run: `node --test test.js`
Expected: 25 testes passando.

- [ ] **Step 5: Commit**

```bash
git add acoes.js test.js
git commit -m "feat(acoes): sugestoes a partir do estado real"
```

---

### Task 7: Servidor e autenticação

**Files:**
- Create: `server.js`
- Create: `public/index.html` (provisório, substituído na Task 9)
- Modify: `test.js`

- [ ] **Step 1: Escrever os testes**

Acrescente ao fim de `test.js`:

```js
const { criarServidor, carregarConfig } = require('./server');

async function subir(sobrescrever = {}) {
  const dados = await fs.mkdtemp(path.join(os.tmpdir(), 'tela-'));
  const config = carregarConfig(dados);
  const chamadas = [];
  const servidor = criarServidor({
    config,
    dados,
    coleta: { leitura: () => ({ nome: 'TESTE', discos: [] }) },
    pastas: { ler: () => ({ estado: 'calculando' }) },
    acoes: {
      estado: async () => ({ inicializacao: [], sugestoes: [] }),
      definirInicializacao: async (id, ligado) => { chamadas.push([id, ligado]); },
      definirEnergia: async (plano) => { chamadas.push([plano]); },
      previaTemp: async () => ({ arquivos: 2, bytes: 5 }),
      limparTemp: async () => ({ apagados: 2, bytes: 5, ignorados: 0 }),
      registrar: async () => {},
      ...sobrescrever,
    },
  });
  await new Promise((r) => servidor.listen(0, '127.0.0.1', r));
  const base = `http://127.0.0.1:${servidor.address().port}`;
  const cookie = `sessao=${config.segredo}`;
  const post = (rota, corpo, extra = {}) => fetch(base + rota, {
    method: 'POST',
    headers: { 'content-type': 'application/json', origin: base, ...extra },
    body: JSON.stringify(corpo),
  });
  return { servidor, base, config, cookie, post, chamadas };
}

test('carregarConfig gera o codigo uma vez e mantem depois', async () => {
  const dados = await fs.mkdtemp(path.join(os.tmpdir(), 'tela-'));
  const a = carregarConfig(dados);
  assert.match(a.codigo, /^[A-Z2-9]{8}$/);
  assert.deepEqual(carregarConfig(dados), a);
});

test('rota protegida sem cookie devolve 401', async (t) => {
  const s = await subir();
  t.after(() => s.servidor.close());
  assert.equal((await fetch(`${s.base}/api/tweaks`)).status, 401);
});

test('pagina abre sem cookie para mostrar o login', async (t) => {
  const s = await subir();
  t.after(() => s.servidor.close());
  assert.equal((await fetch(`${s.base}/`)).status, 200);
});

test('login certo define cookie HttpOnly e Strict', async (t) => {
  const s = await subir();
  t.after(() => s.servidor.close());
  const r = await s.post('/api/login', { codigo: s.config.codigo });
  assert.equal(r.status, 200);
  const c = r.headers.getSetCookie()[0];
  assert.match(c, /HttpOnly/);
  assert.match(c, /SameSite=Strict/);
  assert.match(c, new RegExp(`sessao=${s.config.segredo}`));
});

test('cinco erros bloqueiam ate o codigo certo', async (t) => {
  const s = await subir();
  t.after(() => s.servidor.close());
  for (let i = 0; i < 5; i += 1) {
    assert.equal((await s.post('/api/login', { codigo: 'ERRADO' })).status, 401);
  }
  assert.equal((await s.post('/api/login', { codigo: s.config.codigo })).status, 429);
});

test('POST com Origin diferente e negado', async (t) => {
  const s = await subir();
  t.after(() => s.servidor.close());
  const r = await s.post('/api/login', { codigo: s.config.codigo }, { origin: 'http://evil.test' });
  assert.equal(r.status, 403);
});

test('corpo acima de 4 KB e negado', async (t) => {
  const s = await subir();
  t.after(() => s.servidor.close());
  assert.equal((await s.post('/api/login', { codigo: 'x'.repeat(5000) })).status, 413);
});

test('caminho fora da lista de arquivos nao e servido', async (t) => {
  const s = await subir();
  t.after(() => s.servidor.close());
  assert.equal((await fetch(`${s.base}/server.js`)).status, 404);
  assert.equal((await fetch(`${s.base}/..%2fdados%2fconfig.json`)).status, 404);
});
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `node --test test.js`
Expected: FAIL com `Cannot find module './server'`.

- [ ] **Step 3: Implementar**

Crie `public/index.html` provisório:

```html
<!doctype html><title>Z3R0</title>
```

Crie `server.js`:

```js
'use strict';
const http = require('node:http');
const fs = require('node:fs');
const path = require('node:path');
const crypto = require('node:crypto');

const PORTA = 8420;
const LIMITE_CORPO = 4096;
const ALFABETO = 'ABCDEFGHJKMNPQRSTUVWXYZ23456789';
const ARQUIVOS = {
  '/': ['index.html', 'text/html; charset=utf-8'],
  '/manifest.json': ['manifest.json', 'application/manifest+json'],
  '/icone.png': ['icone.png', 'image/png'],
};

function carregarConfig(dados) {
  const arquivo = path.join(dados, 'config.json');
  try {
    return JSON.parse(fs.readFileSync(arquivo, 'utf8'));
  } catch {
    const codigo = Array.from({ length: 8 }, () => ALFABETO[crypto.randomInt(ALFABETO.length)]).join('');
    const config = { codigo, segredo: crypto.randomBytes(32).toString('hex') };
    fs.mkdirSync(dados, { recursive: true });
    fs.writeFileSync(arquivo, JSON.stringify(config));
    return config;
  }
}

function erroHttp(status, mensagem) {
  return Object.assign(new Error(mensagem), { status });
}

function iguais(a, b) {
  const ha = crypto.createHash('sha256').update(String(a)).digest();
  const hb = crypto.createHash('sha256').update(String(b)).digest();
  return crypto.timingSafeEqual(ha, hb);
}

function json(res, status, corpo, cabecalhos = {}) {
  res.writeHead(status, { 'content-type': 'application/json', ...cabecalhos });
  res.end(JSON.stringify(corpo));
}

async function lerJson(req) {
  // Sair do laco no meio destruiria o socket antes da resposta 413.
  let texto = '';
  let grande = false;
  for await (const pedaco of req) {
    if (texto.length + pedaco.length > LIMITE_CORPO) grande = true;
    else texto += pedaco;
  }
  if (grande) throw erroHttp(413, 'Corpo grande demais');
  try {
    return JSON.parse(texto || '{}');
  } catch {
    throw erroHttp(400, 'JSON inválido');
  }
}

function criarServidor({ config, coleta, pastas, acoes, dados }) {
  const trava = { erros: 0, ate: 0 };

  function autenticado(req) {
    const m = /(?:^|;\s*)sessao=([^;]+)/.exec(req.headers.cookie ?? '');
    return Boolean(m) && iguais(m[1], config.segredo);
  }

  function validarOrigem(req) {
    let host = null;
    try {
      host = new URL(req.headers.origin).host;
    } catch {
      // Sem Origin valido: cai na negacao abaixo.
    }
    if (host !== req.headers.host) throw erroHttp(403, 'Origem negada');
  }

  async function login(req, res) {
    const { codigo } = await lerJson(req);
    if (Date.now() < trava.ate) throw erroHttp(429, 'Muitas tentativas. Aguarde 1 minuto.');
    if (!iguais(codigo, config.codigo)) {
      trava.erros += 1;
      if (trava.erros >= 5) {
        trava.erros = 0;
        trava.ate = Date.now() + 60000;
      }
      throw erroHttp(401, 'Código incorreto');
    }
    trava.erros = 0;
    json(res, 200, { ok: true }, {
      'set-cookie': `sessao=${config.segredo}; HttpOnly; SameSite=Strict; Path=/; Max-Age=31536000`,
    });
  }

  function stream(req, res) {
    res.writeHead(200, {
      'content-type': 'text/event-stream',
      'cache-control': 'no-store',
      connection: 'keep-alive',
    });
    const enviar = () => res.write(`data: ${JSON.stringify(coleta.leitura())}\n\n`);
    enviar();
    const pulso = setInterval(enviar, 2000);
    req.on('close', () => clearInterval(pulso));
  }

  const tweaks = () => acoes.estado(coleta.leitura().discos ?? []);

  async function executar(res, nome, parametro, acao) {
    try {
      await acao();
      await acoes.registrar(dados, nome, parametro, 'ok');
    } catch (e) {
      await acoes.registrar(dados, nome, parametro, `falhou: ${e.message}`);
      throw e;
    }
    json(res, 200, await tweaks());
  }

  async function api(req, res, rota) {
    if (req.method === 'GET' && rota === '/api/stream') return stream(req, res);
    if (req.method === 'GET' && rota === '/api/tweaks') return json(res, 200, await tweaks());
    if (req.method === 'GET' && rota === '/api/temp') return json(res, 200, await acoes.previaTemp());
    if (req.method === 'GET' && rota === '/api/storage/folders') return json(res, 200, pastas.ler());
    if (req.method === 'POST' && rota === '/api/startup') {
      const { id, enabled } = await lerJson(req);
      return executar(res, 'inicializacao', `${id}=${enabled}`, () => acoes.definirInicializacao(id, enabled));
    }
    if (req.method === 'POST' && rota === '/api/power') {
      const { plan } = await lerJson(req);
      return executar(res, 'energia', plan, () => acoes.definirEnergia(plan));
    }
    if (req.method === 'POST' && rota === '/api/temp/clean') {
      const resultado = await acoes.limparTemp();
      const texto = `${resultado.apagados} apagados, ${resultado.ignorados} ignorados`;
      await acoes.registrar(dados, 'temporarios', 'limpar', texto);
      return json(res, 200, { resultado, tweaks: await tweaks() });
    }
    throw erroHttp(404, 'Rota não encontrada');
  }

  return http.createServer(async (req, res) => {
    try {
      const rota = new URL(req.url, 'http://local').pathname;
      if (req.method === 'POST') validarOrigem(req);
      if (req.method === 'POST' && rota === '/api/login') return await login(req, res);
      if (rota.startsWith('/api/')) {
        if (!autenticado(req)) throw erroHttp(401, 'Não autenticado');
        return await api(req, res, rota);
      }
      const arquivo = ARQUIVOS[rota];
      if (!arquivo || req.method !== 'GET') throw erroHttp(404, 'Não encontrado');
      res.writeHead(200, { 'content-type': arquivo[1] });
      fs.createReadStream(path.join(__dirname, 'public', arquivo[0]))
        .on('error', () => res.destroy())
        .pipe(res);
    } catch (e) {
      if (res.headersSent) return res.end();
      const status = e.status ?? 500;
      json(res, status, { erro: status === 500 ? 'Erro interno' : e.message });
    }
  });
}

if (require.main === module) {
  const { iniciarColeta, criarPastas } = require('./coleta');
  const dados = path.join(__dirname, 'dados');
  const config = carregarConfig(dados);
  const servidor = criarServidor({
    config, dados, coleta: iniciarColeta(), pastas: criarPastas(), acoes: require('./acoes'),
  });
  servidor.listen(PORTA, '0.0.0.0', () => {
    console.log(`Painel em http://localhost:${PORTA} | código: ${config.codigo}`);
  });
}

module.exports = { criarServidor, carregarConfig };
```

- [ ] **Step 4: Rodar e ver passar**

Run: `node --test test.js`
Expected: 33 testes passando.

- [ ] **Step 5: Commit**

```bash
git add server.js public/index.html test.js
git commit -m "feat(server): http, login e protecao das rotas"
```

---

### Task 8: Testar as rotas de ação

**Files:**
- Modify: `test.js`

- [ ] **Step 1: Escrever os testes**

Acrescente ao fim de `test.js`:

```js
async function logado(sobrescrever) {
  const s = await subir(sobrescrever);
  const autorizar = (rota, corpo) => s.post(rota, corpo, { cookie: s.cookie });
  return { ...s, autorizar };
}

test('POST de acao sem cookie nao executa nada', async (t) => {
  const s = await subir();
  t.after(() => s.servidor.close());
  assert.equal((await s.post('/api/startup', { id: 'discord', enabled: false })).status, 401);
  assert.deepEqual(s.chamadas, []);
});

test('startup executa, registra e devolve o estado novo', async (t) => {
  const registros = [];
  const s = await logado({ registrar: async (...a) => { registros.push(a.slice(1)); } });
  t.after(() => s.servidor.close());
  const r = await s.autorizar('/api/startup', { id: 'discord', enabled: false });
  assert.equal(r.status, 200);
  assert.deepEqual(await r.json(), { inicializacao: [], sugestoes: [] });
  assert.deepEqual(s.chamadas, [['discord', false]]);
  assert.deepEqual(registros, [['inicializacao', 'discord=false', 'ok']]);
});

test('falha da acao vira mensagem e fica no log', async (t) => {
  const registros = [];
  const s = await logado({
    definirEnergia: async () => { throw Object.assign(new Error('Plano desconhecido'), { status: 400 }); },
    registrar: async (...a) => { registros.push(a.slice(1)); },
  });
  t.after(() => s.servidor.close());
  const r = await s.autorizar('/api/power', { plan: 'turbo' });
  assert.equal(r.status, 400);
  assert.equal((await r.json()).erro, 'Plano desconhecido');
  assert.equal(registros[0][2], 'falhou: Plano desconhecido');
});

test('erro inesperado nao vaza detalhes', async (t) => {
  const s = await logado({ definirEnergia: async () => { throw new Error('C:/segredo/caminho'); } });
  t.after(() => s.servidor.close());
  const r = await s.autorizar('/api/power', { plan: 'desempenho' });
  assert.equal(r.status, 500);
  assert.equal((await r.json()).erro, 'Erro interno');
});

test('limpeza devolve resultado e estado', async (t) => {
  const s = await logado();
  t.after(() => s.servidor.close());
  const corpo = await (await s.autorizar('/api/temp/clean', {})).json();
  assert.deepEqual(corpo.resultado, { apagados: 2, bytes: 5, ignorados: 0 });
  assert.deepEqual(corpo.tweaks, { inicializacao: [], sugestoes: [] });
});

test('stream envia a leitura como evento', async (t) => {
  const s = await subir();
  t.after(() => s.servidor.close());
  const controle = new AbortController();
  const r = await fetch(`${s.base}/api/stream`, { headers: { cookie: s.cookie }, signal: controle.signal });
  const { value } = await r.body.getReader().read();
  controle.abort();
  assert.match(new TextDecoder().decode(value), /^data: \{"nome":"TESTE"/);
});
```

- [ ] **Step 2: Rodar**

Run: `node --test test.js`
Expected: 39 testes passando. As rotas já existem desde a Task 7. Se algum falhar, corrija `server.js` e rode de novo.

- [ ] **Step 3: Commit**

```bash
git add test.js server.js
git commit -m "test(server): cobrir rotas de acao e stream"
```

---

### Task 9: Interface

**Files:**
- Modify: `public/index.html`
- Create: `public/manifest.json`
- Create: `public/icone.png`

- [ ] **Step 1: Gerar o ícone**

Crie `gerar-icone.js` temporário na raiz do projeto:

```js
const zlib = require('node:zlib');
const fs = require('node:fs');
const T = 180;
const barras = [[46, 104, 26], [77, 72, 26], [108, 40, 26]];
const px = Buffer.alloc((T * 4 + 1) * T);
for (let y = 0; y < T; y += 1) {
  const l = y * (T * 4 + 1);
  for (let x = 0; x < T; x += 1) {
    const branco = barras.some(([bx, by, w]) => x >= bx && x < bx + w && y >= by && y < 140);
    const c = branco ? [255, 255, 255] : [10, 132 - Math.round(y / 6), 255];
    px.set([...c, 255], l + 1 + x * 4);
  }
}
const crcT = Array.from({ length: 256 }, (_, n) => {
  let c = n;
  for (let k = 0; k < 8; k += 1) c = c & 1 ? 0xedb88320 ^ (c >>> 1) : c >>> 1;
  return c >>> 0;
});
const crc = (b) => { let c = -1; for (const v of b) c = crcT[(c ^ v) & 255] ^ (c >>> 8); return (c ^ -1) >>> 0; };
const bloco = (tipo, dados) => {
  const t = Buffer.concat([Buffer.from(tipo), dados]);
  const n = Buffer.alloc(4); n.writeUInt32BE(dados.length);
  const f = Buffer.alloc(4); f.writeUInt32BE(crc(t));
  return Buffer.concat([n, t, f]);
};
const ihdr = Buffer.alloc(13);
ihdr.writeUInt32BE(T, 0); ihdr.writeUInt32BE(T, 4); ihdr[8] = 8; ihdr[9] = 6;
fs.writeFileSync('public/icone.png', Buffer.concat([
  Buffer.from([137, 80, 78, 71, 13, 10, 26, 10]),
  bloco('IHDR', ihdr), bloco('IDAT', zlib.deflateSync(px)), bloco('IEND', Buffer.alloc(0)),
]));
```

Run: `node gerar-icone.js`
Expected: `public/icone.png` criado, com ícone azul e três barras brancas. Abra o PNG para conferir e apague `gerar-icone.js`.

- [ ] **Step 2: Criar o manifest**

Crie `public/manifest.json`:

```json
{
  "name": "Painel do PC",
  "short_name": "Painel",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#000000",
  "theme_color": "#000000",
  "icons": [{ "src": "/icone.png", "sizes": "180x180", "type": "image/png" }]
}
```

- [ ] **Step 3: Escrever a interface**

Substitua `public/index.html` por:

```html
<!doctype html>
<html lang="pt-BR">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
<meta name="apple-mobile-web-app-title" content="Painel">
<link rel="manifest" href="/manifest.json">
<link rel="apple-touch-icon" href="/icone.png">
<title>Painel do PC</title>
<style>
  :root {
    color-scheme: dark;
    --l2: rgba(235,235,245,.6); --sep: rgba(84,84,88,.6); --g2: #1c1c1e; --g3: #2c2c2e;
    --blue: #0a84ff; --green: #30d158; --orange: #ff9f0a; --red: #ff453a; --indigo: #5e5cf6; --gray: #8e8e93;
  }
  * { box-sizing: border-box; min-width: 0; }
  html, body { margin: 0; height: 100%; background: #000; color: #fff; overflow: hidden; }
  body { font: 17px/1.25 -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; font-variant-numeric: tabular-nums; -webkit-font-smoothing: antialiased; -webkit-user-select: none; user-select: none; }
  button { font: inherit; color: inherit; border: 0; background: none; cursor: pointer; }
  [hidden] { display: none !important; }

  #app { display: grid; grid-template-columns: 300px 1fr; height: 100%; }
  aside { background: var(--g2); padding: calc(env(safe-area-inset-top) + 28px) 10px 10px; border-right: 1px solid rgba(84,84,88,.35); overflow: auto; }
  aside h1 { font-size: 30px; margin: 0 0 14px 10px; letter-spacing: -.01em; }
  nav button { display: flex; align-items: center; gap: 12px; width: 100%; min-height: 48px; padding: 7px 10px; border-radius: 10px; text-align: left; }
  nav button[aria-current=page] { background: var(--blue); }
  .ic { flex: none; width: 30px; height: 30px; border-radius: 7px; display: grid; place-items: center; }
  .ic svg { width: 19px; height: 19px; }
  .badge { margin-left: auto; background: var(--red); font-size: 13px; font-weight: 600; border-radius: 10px; min-width: 22px; text-align: center; padding: 1px 6px; }

  main { padding: calc(env(safe-area-inset-top) + 28px) 24px 24px; overflow: auto; -webkit-overflow-scrolling: touch; }
  main h2 { font-size: 30px; margin: 0 0 16px; letter-spacing: -.01em; }
  section > * + * { margin-top: 14px; }
  .st { display: grid; grid-template-columns: repeat(3, 1fr); gap: 12px; }
  .st > div { background: var(--g2); border-radius: 14px; padding: 13px 14px; }
  .st small { display: block; font-size: 14px; font-weight: 600; }
  .st b { display: block; font-size: 34px; letter-spacing: -.02em; line-height: 1.15; }
  .st > div > span { font-size: 13px; color: var(--l2); }
  .gp { background: var(--g2); border-radius: 14px; overflow: hidden; }
  .tit { font-size: 13px; color: var(--l2); text-transform: uppercase; letter-spacing: .02em; margin: 18px 0 -8px 16px; }
  .rowl { display: flex; align-items: center; justify-content: space-between; gap: 14px; padding: 10px 16px; min-height: 54px; position: relative; }
  .rowl + .rowl::before { content: ""; position: absolute; top: 0; left: 16px; right: 0; height: 1px; background: var(--sep); }
  .rowl small { display: block; color: var(--l2); font-size: 14px; margin-top: 1px; }
  .chart { position: relative; height: 130px; margin-top: 8px; }
  .chart svg { position: absolute; inset: 0; width: 100%; height: 100%; }
  .chart i { position: absolute; left: 0; right: 0; height: 1px; background: var(--sep); opacity: .55; }
  .pad { padding: 12px 16px; }
  .hd { font-size: 14px; font-weight: 600; color: var(--l2); }
  .cap { height: 8px; border-radius: 8px; background: rgba(120,120,128,.3); overflow: hidden; margin-top: 6px; }
  .cap i { display: block; height: 100%; border-radius: 8px; background: var(--blue); }
  .cap i.warn { background: var(--orange); }
  .dk { display: flex; justify-content: space-between; gap: 12px; font-size: 15px; }
  .dk span:last-child { color: var(--l2); white-space: nowrap; }
  .mt { margin-top: 12px; }

  .sw { flex: none; width: 51px; height: 31px; border-radius: 31px; background: rgba(120,120,128,.36); position: relative; transition: background .25s; }
  .sw::after { content: ""; position: absolute; top: 2px; left: 2px; width: 27px; height: 27px; border-radius: 50%; background: #fff; box-shadow: 0 2px 6px rgba(0,0,0,.35); transition: transform .25s cubic-bezier(.3,.7,.3,1); }
  .sw[aria-checked=true] { background: var(--green); }
  .sw[aria-checked=true]::after { transform: translateX(20px); }
  .seg { flex: none; display: flex; background: rgba(118,118,128,.24); border-radius: 9px; padding: 2px; }
  .seg button { font-size: 14px; font-weight: 500; padding: 8px 14px; min-height: 40px; border-radius: 7px; }
  .seg button[aria-pressed=true] { background: #636366; box-shadow: 0 1px 3px rgba(0,0,0,.3); }
  .pill { flex: none; font-size: 15px; font-weight: 600; color: var(--blue); background: rgba(10,132,255,.18); padding: 9px 18px; min-height: 40px; border-radius: 30px; }
  button:disabled { opacity: .5; }

  #offline, #aviso { position: fixed; left: 50%; transform: translateX(-50%); z-index: 5; border-radius: 30px; padding: 9px 20px; font-size: 15px; font-weight: 600; box-shadow: 0 6px 24px rgba(0,0,0,.5); }
  #offline { top: calc(env(safe-area-inset-top) + 10px); background: var(--orange); color: #000; }
  #aviso { bottom: 24px; background: var(--g3); max-width: 80%; text-align: center; }
  #aviso.erro { background: var(--red); }

  #login { position: fixed; inset: 0; background: #000; display: grid; place-items: center; z-index: 10; }
  #login form { width: 340px; text-align: center; }
  #login h1 { font-size: 30px; margin: 0 0 6px; }
  #login p { color: var(--l2); margin: 0 0 18px; }
  #login input { width: 100%; font: inherit; font-size: 22px; letter-spacing: .14em; text-align: center; text-transform: uppercase; color: #fff; background: var(--g2); border: 0; border-radius: 12px; padding: 13px; outline: none; -webkit-user-select: text; user-select: text; }
  #login .pill { width: 100%; margin-top: 12px; color: #fff; background: var(--blue); }
  #login .err { color: var(--red); min-height: 22px; margin-top: 10px; font-size: 15px; }

  dialog { background: var(--g3); color: #fff; border: 0; border-radius: 16px; padding: 22px; width: 340px; text-align: center; }
  dialog::backdrop { background: rgba(0,0,0,.55); }
  dialog h3 { margin: 0 0 6px; font-size: 19px; }
  dialog p { margin: 0 0 16px; color: var(--l2); font-size: 15px; }
  dialog div { display: flex; gap: 10px; }
  dialog button { flex: 1; min-height: 44px; border-radius: 12px; background: rgba(120,120,128,.3); font-weight: 600; }
  dialog .perigo { background: var(--red); }

  @media (max-width: 820px) { #app { grid-template-columns: 220px 1fr; } .seg button { padding: 8px 9px; } }
  @media (prefers-reduced-motion: reduce) { .sw, .sw::after { transition: none; } }
</style>
</head>
<body>

<div id="app">
  <aside>
    <h1 id="nome">PC</h1>
    <nav id="nav"></nav>
  </aside>
  <main>
    <section id="geral">
      <h2>Visão geral</h2>
      <div class="st">
        <div><small style="color:var(--blue)">CPU</small><b id="cpuUso">—</b><span id="cpuSub">—</span></div>
        <div><small style="color:var(--green)">GPU</small><b id="gpuUso">—</b><span id="gpuSub">—</span></div>
        <div><small style="color:var(--orange)">RAM</small><b id="ramUso">—</b><span id="ramSub">—</span></div>
      </div>
      <div class="gp pad">
        <div class="hd">CPU · último minuto</div>
        <div class="chart">
          <i style="top:0"></i><i style="top:50%"></i><i style="top:100%"></i>
          <svg viewBox="0 0 100 100" preserveAspectRatio="none" aria-hidden="true">
            <defs><linearGradient id="ar" x1="0" y1="0" x2="0" y2="1"><stop offset="0" stop-color="#0a84ff" stop-opacity=".45"/><stop offset="1" stop-color="#0a84ff" stop-opacity="0"/></linearGradient></defs>
            <polygon id="area" fill="url(#ar)"></polygon>
            <polyline id="linha" fill="none" stroke="#0a84ff" stroke-width="2" vector-effect="non-scaling-stroke" stroke-linejoin="round"></polyline>
          </svg>
        </div>
      </div>
      <div class="tit">Sugestões</div>
      <div class="gp" id="sugResumo"></div>
    </section>

    <section id="armazenamento" hidden>
      <h2>Armazenamento</h2>
      <div class="gp pad" id="discos"></div>
      <div class="tit">Maiores pastas</div>
      <div class="gp" id="pastas"></div>
    </section>

    <section id="inicializacao" hidden>
      <h2>Inicialização</h2>
      <div class="gp" id="itens"></div>
    </section>

    <section id="energia" hidden>
      <h2>Energia</h2>
      <div class="gp" id="energiaBox"></div>
    </section>

    <section id="sugestoes" hidden>
      <h2>Sugestões</h2>
      <div class="gp" id="sugTodas"></div>
    </section>
  </main>
</div>

<div id="offline" hidden>Sem conexão</div>
<div id="aviso" role="status" hidden></div>

<div id="login" hidden>
  <form id="formLogin">
    <h1>Painel do PC</h1>
    <p>Digite o código mostrado na instalação.</p>
    <input id="codigo" autocomplete="off" autocapitalize="characters" spellcheck="false" maxlength="8" aria-label="Código de acesso">
    <button class="pill" type="submit">Entrar</button>
    <div class="err" id="loginErro"></div>
  </form>
</div>

<dialog id="confirmar">
  <h3>Limpar temporários?</h3>
  <p id="confirmarTexto"></p>
  <div><button id="cancelar">Cancelar</button><button id="executar" class="perigo">Limpar</button></div>
</dialog>

<script>
'use strict';
const $ = (id) => document.getElementById(id);
const esc = (t) => String(t).replace(/[&<>"']/g, (c) => `&#${c.charCodeAt(0)};`);
const gb = (b, c = 1) => (b == null ? '—' : (b / 1024 ** 3).toFixed(c).replace('.', ','));
const SECOES = [
  ['geral', 'Visão geral', 'var(--blue)', '<rect x="3" y="12" width="4" height="9" rx="1"/><rect x="10" y="7" width="4" height="14" rx="1"/><rect x="17" y="3" width="4" height="18" rx="1"/>'],
  ['armazenamento', 'Armazenamento', 'var(--gray)', '<rect x="3" y="5" width="18" height="6" rx="2"/><rect x="3" y="13" width="18" height="6" rx="2"/>'],
  ['inicializacao', 'Inicialização', 'var(--indigo)', '<path d="M11 3h2v9h-2z"/><path d="M6.3 6.3a8 8 0 1 0 11.4 0l-1.4 1.4a6 6 0 1 1-8.6 0z"/>'],
  ['energia', 'Energia', 'var(--green)', '<path d="M13 2 5 14h6l-1 8 8-12h-6z"/>'],
  ['sugestoes', 'Sugestões', 'var(--orange)', '<path d="M12 2a7 7 0 0 0-4 12.7V17h8v-2.3A7 7 0 0 0 12 2zM9 19h6v2H9z"/>'],
];
let tweaks = null;
let leitura = null;
let fonte = null;
let timerPastas = null;

$('nav').innerHTML = SECOES.map(([id, nome, cor, icone]) =>
  `<button data-sec="${id}"><span class="ic" style="background:${cor}"><svg viewBox="0 0 24 24" fill="#fff" aria-hidden="true">${icone}</svg></span><span>${nome}</span>${id === 'sugestoes' ? '<span class="badge" id="badge" hidden></span>' : ''}</button>`).join('');

function abrir(id) {
  SECOES.forEach(([s]) => { $(s).hidden = s !== id; });
  document.querySelectorAll('#nav button').forEach((b) => {
    if (b.dataset.sec === id) b.setAttribute('aria-current', 'page');
    else b.removeAttribute('aria-current');
  });
  clearTimeout(timerPastas);
  if (id === 'armazenamento') carregarPastas();
}
$('nav').addEventListener('click', (e) => {
  const b = e.target.closest('button');
  if (b) abrir(b.dataset.sec);
});

function avisar(texto, erro = false) {
  const a = $('aviso');
  a.textContent = texto;
  a.className = erro ? 'erro' : '';
  a.hidden = false;
  clearTimeout(avisar.t);
  avisar.t = setTimeout(() => { a.hidden = true; }, 4000);
}

async function api(rota, corpo) {
  const opcoes = corpo === undefined ? {} : {
    method: 'POST', headers: { 'content-type': 'application/json' }, body: JSON.stringify(corpo),
  };
  const r = await fetch(rota, opcoes);
  if (r.status === 401 && rota !== '/api/login') { mostrarLogin(); throw new Error('Sessão expirada'); }
  const dados = await r.json();
  if (!r.ok) throw new Error(dados.erro ?? 'Falha na ação');
  return dados;
}

function mostrarLogin() {
  fonte?.close();
  fonte = null;
  $('login').hidden = false;
}

$('formLogin').addEventListener('submit', async (e) => {
  e.preventDefault();
  $('loginErro').textContent = '';
  try {
    await api('/api/login', { codigo: $('codigo').value.trim().toUpperCase() });
    $('login').hidden = true;
    iniciar();
  } catch (erro) {
    $('loginErro').textContent = erro.message;
  }
});

function desenharLeitura(l) {
  leitura = l;
  $('nome').textContent = l.nome;
  $('cpuUso').textContent = `${l.cpu.uso}%`;
  const temp = l.hwinfo?.cpuTemp;
  const ghz = l.hwinfo?.cpuClock;
  $('cpuSub').textContent = [temp == null ? '—' : `${Math.round(temp)}°C`, ghz == null ? null : `${(ghz / 1000).toFixed(1).replace('.', ',')} GHz`].filter(Boolean).join(' · ');
  $('gpuUso').textContent = l.gpu?.uso == null ? '—' : `${l.gpu.uso}%`;
  $('gpuSub').textContent = l.gpu ? `${l.gpu.temp ?? '—'}°C · ${l.gpu.watts == null ? '—' : Math.round(l.gpu.watts)} W · VRAM ${gb(l.gpu.vramUsada * 1024 ** 2)} de ${gb(l.gpu.vramTotal * 1024 ** 2, 0)} GB` : '—';
  $('ramUso').textContent = gb(l.ram.usada);
  $('ramSub').textContent = `de ${gb(l.ram.total, 0)} GB`;

  const h = l.cpu.historico;
  const passos = Math.max(h.length - 1, 1);
  const pontos = h.map((v, i) => `${(i / passos * 100).toFixed(1)},${100 - v}`).join(' ');
  $('linha').setAttribute('points', pontos);
  $('area').setAttribute('points', h.length ? `0,100 ${pontos} 100,100` : '');

  $('discos').innerHTML = l.discos.map((d, i) => {
    if (d.total == null) return `<div class="dk${i ? ' mt' : ''}"><span>${esc(d.letra)}: ${esc(d.nome)}</span><span>Indisponível</span></div>`;
    const usado = 100 * (1 - d.livre / d.total);
    return `<div class="${i ? 'mt' : ''}"><div class="dk"><span>${esc(d.letra)}: ${esc(d.nome)}</span><span>${gb(d.livre, 0)} GB livres de ${gb(d.total, 0)}</span></div><div class="cap"><i class="${usado > 80 ? 'warn' : ''}" style="width:${usado.toFixed(1)}%"></i></div></div>`;
  }).join('');
}

function controleInicio(i) {
  return `<button class="sw" role="switch" aria-checked="${i.ligado}" aria-label="${esc(i.nome)}" data-inicio="${esc(i.id)}"></button>`;
}
function controleEnergia() {
  const e = tweaks.energia;
  const nomes = { equilibrado: 'Equilibrado', desempenho: 'Alto desempenho' };
  return `<div class="seg">${e.disponiveis.map((p) => `<button aria-pressed="${e.ativo === p}" data-plano="${p}">${nomes[p]}</button>`).join('')}</div>`;
}
function linhaSugestao(s) {
  let controle = '';
  if (s.controle?.tipo === 'inicio') {
    const item = tweaks.inicializacao.find((i) => i.id === s.controle.id);
    if (item) controle = controleInicio(item);
  } else if (s.controle?.tipo === 'energia') {
    controle = controleEnergia();
  } else if (s.controle?.tipo === 'temp') {
    controle = '<button class="pill" data-limpar>Limpar</button>';
  }
  return `<div class="rowl"><div>${esc(s.titulo)}<small>${esc(s.texto)}</small></div>${controle}</div>`;
}

function desenharTweaks(t) {
  tweaks = t;
  const comAcao = t.sugestoes.filter((s) => s.controle);
  $('sugResumo').innerHTML = comAcao.length
    ? comAcao.map(linhaSugestao).join('')
    : '<div class="rowl"><div>Nenhuma ação pendente</div></div>';
  $('sugTodas').innerHTML = t.sugestoes.map(linhaSugestao).join('');
  $('itens').innerHTML = t.inicializacao.map((i) => `<div class="rowl"><div>${esc(i.nome)}</div>${controleInicio(i)}</div>`).join('');
  $('energiaBox').innerHTML = `<div class="rowl"><div>Plano de energia<small>Alto desempenho consome mais com o PC parado.</small></div>${controleEnergia()}</div>`;
  $('badge').textContent = comAcao.length;
  $('badge').hidden = comAcao.length === 0;
}

async function acao(botao, rota, corpo) {
  botao.disabled = true;
  try {
    desenharTweaks(await api(rota, corpo));
  } catch (erro) {
    avisar(erro.message, true);
    desenharTweaks(await api('/api/tweaks'));
  }
}

async function pedirLimpeza() {
  const p = await api('/api/temp');
  $('confirmarTexto').textContent = `${p.arquivos} arquivos com mais de 7 dias, somando ${gb(p.bytes)} GB. Isso não pode ser desfeito.`;
  $('confirmar').showModal();
}

$('cancelar').addEventListener('click', () => $('confirmar').close());
$('executar').addEventListener('click', async () => {
  $('confirmar').close();
  try {
    const { resultado, tweaks: t } = await api('/api/temp/clean', {});
    desenharTweaks(t);
    avisar(`${gb(resultado.bytes)} GB liberados, ${resultado.ignorados} arquivos em uso ignorados.`);
  } catch (erro) {
    avisar(erro.message, true);
  }
});

document.querySelector('main').addEventListener('click', (e) => {
  const b = e.target.closest('button');
  if (!b) return;
  if (b.dataset.inicio) acao(b, '/api/startup', { id: b.dataset.inicio, enabled: b.getAttribute('aria-checked') !== 'true' });
  else if (b.dataset.plano) acao(b, '/api/power', { plan: b.dataset.plano });
  else if ('limpar' in b.dataset) pedirLimpeza().catch((erro) => avisar(erro.message, true));
});

async function carregarPastas() {
  const box = $('pastas');
  try {
    const r = await api('/api/storage/folders');
    if (r.estado !== 'pronto') {
      box.innerHTML = '<div class="rowl"><div>Calculando…</div></div>';
      timerPastas = setTimeout(carregarPastas, 3000);
      return;
    }
    box.innerHTML = r.pastas.map((p) => `<div class="rowl"><div>${esc(p.caminho)}</div><div>${gb(p.bytes)} GB</div></div>`).join('')
      + (r.parcial ? '<div class="rowl"><small>Algumas pastas sem acesso ficaram de fora.</small></div>' : '');
  } catch (erro) {
    box.innerHTML = `<div class="rowl"><div>${esc(erro.message)}</div></div>`;
  }
}

function conectar() {
  fonte?.close();
  fonte = new EventSource('/api/stream');
  fonte.onmessage = (e) => {
    $('offline').hidden = true;
    desenharLeitura(JSON.parse(e.data));
  };
  fonte.onerror = () => {
    $('offline').hidden = false;
    if (fonte?.readyState === EventSource.CLOSED) setTimeout(conectar, 3000);
  };
}

async function iniciar() {
  try {
    desenharTweaks(await api('/api/tweaks'));
    conectar();
  } catch (erro) {
    if (erro.message === 'Sessão expirada') return;
    $('offline').hidden = false;
    setTimeout(iniciar, 3000);
  }
}

abrir('geral');
iniciar();
setInterval(() => { if (leitura && fonte && $('login').hidden) api('/api/tweaks').then(desenharTweaks).catch(() => {}); }, 60000);
</script>
</body>
</html>
```

- [ ] **Step 4: Testar no PC**

Run: `node server.js` em segundo plano.
Abra `http://localhost:8420` no Edge headless ou no navegador do PC, com janela 1024×768.
Expected:
- Tela de login primeiro. O código aparece no terminal.
- Depois de entrar, a barra lateral mostra `Z3R0` e números reais de CPU, GPU e RAM.
- Temperatura da CPU mostra "—".
- Armazenamento mostra os três discos e, depois de alguns minutos, as maiores pastas.
- Inicialização mostra Docker desligado e Discord ligado.

Pare o servidor. **Não acione nenhum controle ainda:** isso é verificado na Task 11.

- [ ] **Step 5: Commit**

```bash
git add public
git commit -m "feat(ui): painel no estilo do iPadOS"
```

---

### Task 10: Instalação

**Files:**
- Create: `instalar.ps1`

- [ ] **Step 1: Escrever o instalador**

Crie `instalar.ps1`:

```powershell
#Requires -RunAsAdministrator
$ErrorActionPreference = 'Stop'
$pasta = $PSScriptRoot
$node = (Get-Command node).Source
$nome = 'Terceira Tela iPad'

Get-NetFirewallRule -DisplayName $nome -ErrorAction SilentlyContinue | Remove-NetFirewallRule
New-NetFirewallRule -DisplayName $nome -Direction Inbound -Action Allow -Protocol TCP -LocalPort 8420 `
  -RemoteAddress LocalSubnet -Program $node -Profile Private,Public | Out-Null

$usuario = "$env:USERDOMAIN\$env:USERNAME"
$acao = New-ScheduledTaskAction -Execute 'conhost.exe' -Argument "--headless `"$node`" server.js" -WorkingDirectory $pasta
$gatilho = New-ScheduledTaskTrigger -AtLogOn -User $usuario
$opcoes = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries `
  -ExecutionTimeLimit ([TimeSpan]::Zero) -RestartCount 3 -RestartInterval (New-TimeSpan -Minutes 1)
$principal = New-ScheduledTaskPrincipal -UserId $usuario -LogonType Interactive -RunLevel Limited
Register-ScheduledTask -TaskName $nome -Action $acao -Trigger $gatilho -Settings $opcoes -Principal $principal -Force | Out-Null

Start-ScheduledTask -TaskName $nome
Start-Sleep -Seconds 3
$config = Get-Content (Join-Path $pasta 'dados\config.json') | ConvertFrom-Json
Write-Host "Painel: http://$($env:COMPUTERNAME.ToLower()).local:8420 ou http://192.168.1.2:8420"
Write-Host "Codigo: $($config.codigo)"
```

- [ ] **Step 2: Conferir sintaxe sem instalar**

Run, direto no PowerShell, a partir da raiz do projeto:

```powershell
$e=$null; [void][System.Management.Automation.Language.Parser]::ParseFile("$PWD\instalar.ps1",[ref]$null,[ref]$e); $e.Count
```

Expected: `0`.

- [ ] **Step 3: Commit**

```bash
git add instalar.ps1
git commit -m "feat(instalar): firewall e tarefa no login"
```

---

### Task 11: Verificação real

Esta tarefa depende do usuário. Não instale nem aja no sistema sem a confirmação dele em cada bloco.

**Files:**
- Modify: `docs/superpowers/specs/2026-11-04-terceira-tela-ipad-design.md` (apenas se algo mudar)

- [ ] **Step 1: Instalar**

Pedir ao usuário para abrir um PowerShell como administrador e rodar `.\instalar.ps1` na pasta do projeto. Se a política de execução bloquear, ele pode usar `powershell -ExecutionPolicy Bypass -File .\instalar.ps1`.
Expected: endereço e código impressos.

- [ ] **Step 2: Conectar o iPad**

No Safari do iPad, abrir `http://z3r0.local:8420`. Se não resolver, usar `http://192.168.1.2:8420` e anotar para a reserva no roteador. Adicionar à Tela de Início, abrir pelo ícone e digitar o código.
Expected: painel em tela cheia, sem barras do Safari.

- [ ] **Step 3: Testar cada ação e desfazer**

- Desligar Discord pelo iPad, conferir no Gerenciador de Tarefas e religar.
- Trocar para Alto desempenho, conferir com `powercfg /getactivescheme` e voltar para Equilibrado.
- Confirmar a limpeza de temporários só se o usuário concordar. Conferir o total antes.
- Ler `dados/acoes.log` e conferir uma linha por ação.

Expected: estado do iPad igual ao estado real do Windows em todas as verificações.

- [ ] **Step 4: Testar falhas**

- Parar a tarefa agendada: o iPad mostra "Sem conexão".
- Iniciar a tarefa: o iPad volta sozinho.
- Reiniciar o PC: o serviço sobe e o iPad reconecta.

- [ ] **Step 5: Medir consumo**

Run: `Get-Process node | Select-Object Id,CPU,WorkingSet64` duas vezes, com 60 s de intervalo.
Expected: variação de `CPU` abaixo de 0,6 s nesse intervalo, equivalente a menos de 1% de um núcleo. A meta da especificação é consumo abaixo de 1% de CPU em média.

- [ ] **Step 6: Observar o iPad**

Deixar o painel aberto por algumas horas com Bloqueio Automático em "Nunca".
Expected: sem recarregamentos nem travamentos.

- [ ] **Step 7: Registrar no README e publicar**

Crie `README.md` curto com o que é o projeto, requisitos e como instalar. Depois, com aprovação do usuário:

```bash
git add README.md
git commit -m "docs: instrucoes de instalacao"
git push
```

---

## Etapas posteriores (planos separados)

1. HWiNFO: instalar, ativar "Reporting to Gadget", trocar a amostra sintética de `test.js` pela saída real e testar com o Vanguard.
2. Monitor estendido com app pronto.
