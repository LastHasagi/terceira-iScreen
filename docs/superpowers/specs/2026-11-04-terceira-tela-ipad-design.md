# Terceira tela no iPad: design

Data: 2026-11-04
Estado: aprovado na conversa, aguardando revisão desta especificação

## Objetivo

Usar um iPad 6ª geração (iPadOS 17.7, tela 4:3, 2 GB de RAM) como painel fixo do PC Z3R0, mostrando a saúde da máquina, sugerindo melhorias e aplicando um conjunto fechado de ajustes.

O painel é uma página web servida pelo próprio PC. O uso do iPad como monitor estendido do Windows fica para uma etapa posterior, com app pronto e sem código deste projeto.

## Decisões

| Tema | Decisão | Motivo |
|---|---|---|
| Forma | Página salva na Tela de Início do iPad | App nativo exigiria Mac com Xcode |
| Servidor | Node 24, sem dependências | Já instalado; Python exigiria `psutil` e .NET não está instalado |
| Privilégio | Usuário comum, sem administrador | Todas as ações da v1 dispensam elevação |
| Layout | Estilo do app Ajustes: barra lateral e listas agrupadas | Escolha do usuário (opção B dos mockups) |
| Tweaks | Sugere e aplica pelo iPad, com confirmação | Escolha do usuário (opção C) |
| Temperatura da CPU | Opcional, via HWiNFO | O Windows não expõe esse dado neste PC e o HWiNFO pode conflitar com o Vanguard |

## Fora da v1

Encerrar processos, mexer em Riot Vanguard e SecurityHealth, notificações, modo claro, HTTPS e instalar HWiNFO ou monitor estendido. As duas últimas instalações são etapas posteriores ao painel.

## Arquivos

| Arquivo | Responsabilidade |
|---|---|
| `server.js` | HTTP, autenticação, rotas e stream de dados |
| `coleta.js` | Leitura e interpretação de métricas, sem efeitos colaterais |
| `acoes.js` | Lista fechada de ações, com leitura de estado e execução |
| `public/index.html` | Interface completa |
| `public/manifest.json` e ícone | Instalação na Tela de Início |
| `instalar.ps1` | Regra de firewall e tarefa agendada, rodado uma vez como administrador |
| `test.js` | Testes com `node --test` |
| `dados/` | Código de acesso e log de ações, fora do controle de versão |

## Coleta

| Dado | Fonte | Frequência |
|---|---|---|
| Uso de CPU | `os.cpus()`, diferença entre leituras | 2 s |
| RAM | `os.totalmem()` e `os.freemem()` | 2 s |
| Discos C:, D: e E: | `fs.statfs` | 30 s |
| GPU, VRAM, temperatura e potência | Um processo contínuo `nvidia-smi --query-gpu=... --format=csv,noheader,nounits -l 2` | 2 s |
| Temperatura da CPU, frequência e ventoinhas | `reg query HKCU\SOFTWARE\HWiNFO64\VSB` | 5 s; 30 s se a chave não existir |
| Maiores pastas | Varredura do primeiro nível de `C:\` e do perfil do usuário | Sob demanda |

Regras:

- O `nvidia-smi` reinicia sozinho se morrer, com espera de 5 s entre tentativas.
- A frequência da CPU vem apenas do HWiNFO, porque `os.cpus()` não garante valor atual no Windows.
- Os valores do HWiNFO são localizados pelo rótulo; o sensor principal esperado neste Ryzen é `CPU (Tctl/Tdie)`.
- A varredura de pastas roda uma de cada vez, ignora pastas sem acesso, marca o resultado como parcial nesse caso e guarda o resultado por 10 minutos.
- O servidor guarda as últimas 30 leituras de CPU, para o gráfico do último minuto sobreviver a recarregar a página.

## Ações

Toda ação recebe um identificador validado contra a lista abaixo. O servidor usa `execFile` com argumentos fixos, sem interpretar texto vindo do iPad.

### Inicialização

Itens permitidos: Ollama, OpenClaw Gateway, Steam, Discord, RiotClient, Docker Desktop, Proton VPN e LGHUB.

O estado é lido e gravado em `HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\StartupApproved`, nas subchaves `Run` e `StartupFolder`, o mesmo mecanismo do Gerenciador de Tarefas. O bit menos significativo do primeiro byte indica desligado (`03`) ou ligado (`02`), e valor ausente significa ligado. Listar `Win32_StartupCommand` não serve, porque inclui itens desativados.

Um item só aparece se existir de fato em `HKCU\...\CurrentVersion\Run` ou na pasta Inicializar do usuário.

Estado em 2026-11-04: ligados Ollama, Discord e Proton VPN; desligados Docker Desktop, Steam, RiotClient, LGHUB e OpenClaw Gateway.

### Plano de energia

Alterna entre Equilibrado e Alto desempenho com `powercfg /setactive`. O estado vem de `powercfg /getactivescheme`. Se Alto desempenho não existir, a opção fica oculta.

### Limpeza de temporários

- Alcança somente arquivos em `%TEMP%` com mais de 7 dias.
- Não segue links simbólicos nem junções.
- Mostra quantidade e tamanho antes de confirmar. O servidor recalcula a lista ao executar.
- Arquivos em uso são ignorados e informados no resultado.
- Não pode ser desfeita, por isso exige confirmação.

### Regras comuns

- Depois de agir, o servidor relê o estado real e devolve esse valor.
- A interface mostra o estado devolvido pelo servidor. Em caso de falha, o controle volta ao estado real e exibe o erro.
- Cada tentativa é registrada em `dados/acoes.log` com data, ação, parâmetro e resultado.

## Sugestões

Regras simples no servidor, cada uma com texto curto e ação quando houver:

| Condição | Sugestão | Ação |
|---|---|---|
| Disco com menos de 20% livre | Aponta o disco e leva a Armazenamento | Nenhuma |
| Docker ou Ollama ligados na inicialização | Sugere desligar | Interruptor |
| Plano Equilibrado | Informa a opção e o ganho provavelmente pequeno | Controle segmentado |
| Temporários com mais de 7 dias acima de 256 MB | Informa o total | Limpar |
| RAM a 2667 MHz | Se o kit suportar mais, ativar XMP/DOCP na BIOS | Nenhuma |
| Unidade ligada por USB | Evitar instalar jogos no D: | Nenhuma |
| Disco rígido | Usar o E: só para arquivos | Nenhuma |

A velocidade nominal da RAM é desconhecida, então a sugestão precisa ser condicional.

## Interface

- **Barra lateral:** nome do PC vindo de `os.hostname()`, com Visão geral, Armazenamento, Inicialização, Energia e Sugestões.
- **Visão geral:** blocos de CPU, GPU e RAM, gráfico da CPU no último minuto e sugestões com ação.
- **Armazenamento:** barras por disco e maiores pastas.
- **Inicialização:** os 8 itens com interruptor.
- **Energia:** controle segmentado.
- **Sugestões:** lista completa, com badges na barra lateral.

Linguagem visual: fonte do sistema via `-apple-system`, cores do sistema em modo escuro, listas agrupadas, interruptores e controles segmentados em CSS. Ícones próprios em SVG, sem SF Symbols. Otimizado para 1024×768 em paisagem, utilizável em retrato, com alvos de toque de pelo menos 44 pt e respeito a `prefers-reduced-motion`. Atualizações usam só CSS e SVG, sem canvas nem bibliotecas.

## API

| Rota | Função |
|---|---|
| `POST /api/login` | Valida o código e define o cookie |
| `GET /api/stream` | Server-Sent Events com uma leitura a cada 2 s |
| `GET /api/tweaks` | Estado das ações e sugestões |
| `POST /api/startup` | `{id, enabled}` |
| `POST /api/power` | `{plan}` |
| `GET /api/temp` | Prévia da limpeza |
| `POST /api/temp/clean` | Executa a limpeza |
| `GET /api/storage/folders` | Maiores pastas |

## Segurança

- Porta 8420, com regra de firewall restrita à sub-rede local. A rede Ethernet está no perfil Público, então a regra precisa cobrir esse perfil.
- Código de 8 caracteres gerado na primeira execução e mostrado no terminal do PC, digitado uma vez no iPad.
- Cookie `HttpOnly`, `SameSite=Strict`, válido por 1 ano. Rotas sem cookie válido retornam 401.
- Cinco códigos errados bloqueiam novas tentativas por 1 minuto.
- Requisições POST exigem `Origin` igual ao `Host`.
- Corpo limitado a 4 KB.
- Limite aceito: HTTP sem criptografia. Alguém já conectado à rede de casa poderia capturar o código ou o cookie.

## Falhas

| Situação | Comportamento |
|---|---|
| PC desligado ou em suspensão | Faixa "Sem conexão" no iPad e reconexão automática |
| `nvidia-smi` indisponível | Campos de GPU com "—" e reinício automático |
| HWiNFO ausente ou bloqueado | Temperatura da CPU com "—", sem aviso de erro |
| Ação falhou | Mensagem na tela e controle no estado real |
| Disco sem leitura | Disco marcado como indisponível, restante normal |

## Instalação

1. Rodar `instalar.ps1` como administrador uma vez, para criar a regra de firewall e a tarefa agendada no login, com o usuário comum e sem janela visível.
2. No iPad, abrir `http://<PC>:8420` no Safari e adicionar à Tela de Início.
3. Abrir pelo ícone e digitar o código. A página instalada tem cookies próprios, separados do Safari.
4. Ajustar o Bloqueio Automático para "Nunca" e manter o iPad na tomada.

Durante a implementação, verificar se o iPad resolve `z3r0.local`. Se não resolver, usar `192.168.1.2` e reservar esse IP no roteador.

## Testes

Automáticos, com `node --test`:

- Interpretação da saída do `nvidia-smi`, incluindo `[N/A]`.
- Interpretação de `reg query` para HWiNFO e StartupApproved.
- Cálculo de uso de CPU.
- Rejeição de identificador fora da lista.
- Seleção de temporários por idade, sem seguir links.
- Regras de sugestão.
- Rotas sem cookie, com `Origin` errado e bloqueio por tentativas.

Manuais, no PC e no iPad:

- Consumo do servidor abaixo de 1% de CPU em média.
- Cada ação conferida no Gerenciador de Tarefas ou em `powercfg`, e desfeita em seguida.
- Reinício do PC, reconexão do iPad e subida automática do serviço.
- Uso contínuo no iPad por algumas horas, observando memória e fluidez.

## Etapas posteriores

1. **HWiNFO:** instalar, ativar "Reporting to Gadget", manter só os sensores necessários e testar junto do Vanguard. Confirmar se o limite de 12 horas da versão gratuita afeta o registro.
2. **Monitor estendido:** testar um app pronto, como Spacedesk, sem código neste projeto.
