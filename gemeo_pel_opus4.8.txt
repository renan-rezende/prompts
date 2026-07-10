# Projeto: Gêmeo Digital Interativo — Discos de Pelotamento
 
## Papel
Você é um engenheiro sênior de software industrial, especialista em visualização 3D
web (React Three Fiber) e em sistemas de controle avançado de processo. Construa uma
aplicação completa, funcional e bem arquitetada.
 
## Contexto de domínio
Usina de pelotização de minério de ferro (Samarco). São 12 discos de pelotamento que
formam pelotas cruas. Cada disco possui:
 
- Velocidade de rotação (rpm)
- Inclinação (graus)
- Alimentação (t/h de material chegando ao disco)
- Distribuição granulométrica das pelotas, medida em 7 faixas (mm):
  0–6, 6–8, 8–9, 9–12, 12–16, 16–19, >19  (percentuais que somam 100%)
 
Controle:
- SCAP (Sistema de Controle Avançado de Processo) atua automaticamente sobre
  velocidade e inclinação de cada disco.
- O operador pode habilitar ou desabilitar o SCAP por disco.
- Quando o SCAP está DESABILITADO, velocidade e inclinação são definidas manualmente
  pelo operador. Isso ocorre tipicamente quando a alimentação está ruim ou sofre
  mudança brusca de característica.
- PV (Process Variable) = %faixa(9–12) / %faixa(12–16). Faixa válida: 0.0 a 2.0.
- SP (Setpoint) = alvo que o SCAP persegue para o PV. Faixa válida: 0.0 a 2.0.
- PV e SP só são exibidos quando o SCAP está habilitado.
 
## Requisitos funcionais
 
### Cena 3D
1. Visão geral: os 12 discos dispostos em layout de planta (ex.: 2 fileiras de 6),
   cada um como um disco inclinado que gira em torno do próprio eixo, com velocidade
   de rotação visualmente proporcional ao valor real de rpm e inclinação real
   refletida na malha.
2. Cada disco exibe um indicador visual de estado:
   - SCAP habilitado → contorno/emissive verde
   - SCAP desabilitado (modo manual) → contorno/emissive âmbar
   - Desvio |PV − SP| acima de tolerância → pulsação vermelha
3. Hover: highlight + tooltip mínimo (ID do disco, estado do SCAP).
4. Click em um disco: transição suave de câmera (easing, ~1.2s) que enquadra e isola
   o disco selecionado; os demais discos ficam atenuados (opacidade reduzida ou
   desfocados).
5. Tecla ESC ou botão "Voltar" retorna à visão geral com transição reversa.
6. Sem selecionar disco, câmera em órbita livre com limites (OrbitControls travado
   em polar angle e distância).
 
### Painel de dados (HUD em HTML, não em texto 3D)
Quando um disco está selecionado, exibir painel lateral com:
- Identificação do disco
- Badge do estado do SCAP (Habilitado / Manual)
- PV e SP (somente se SCAP habilitado), com barra visual mostrando PV contra SP
  e a zona de tolerância
- Velocidade (rpm) e inclinação (°), com indicação clara de quem está comandando
  (SCAP ou Operador)
- Alimentação (t/h)
- Histograma da distribuição granulométrica nas 7 faixas, com as faixas 9–12 e
  12–16 destacadas (são as que compõem o PV)
- Gráfico de tendência (últimos ~5 min) de PV e SP sobrepostos
- Toggle para habilitar/desabilitar o SCAP
- Quando o SCAP está desabilitado: sliders para o operador ajustar velocidade e
  inclinação manualmente, com limites de faixa operacional
 
### Barra superior
- Status geral: quantos discos em automático, quantos em manual, quantos em desvio
- Alimentação total da linha
- Timestamp da última atualização
 
## Camada de dados
- Backend FastAPI expondo:
  - `GET /api/discs` → snapshot dos 12 discos
  - `WS /ws/telemetry` → push de telemetria a cada 1s
  - `POST /api/discs/{id}/scap` → habilita/desabilita SCAP
  - `POST /api/discs/{id}/setpoints` → velocidade e inclinação (só aceito em modo manual)
- Simulador de processo (para o POC) que:
  - Gera alimentação com ruído e ocasionais degraus (mudança brusca de material)
  - Modela a granulometria como função de velocidade, inclinação e alimentação
    (não precisa ser fisicamente rigoroso, mas precisa ser coerente: mais velocidade
     e mais inclinação → pelota menor; mais alimentação → deslocamento da distribuição)
  - Recalcula PV a partir das faixas 9–12 e 12–16
  - Quando o SCAP está habilitado, roda um controlador PI simples que ajusta
    velocidade e inclinação para levar PV até o SP, com taxa de variação limitada
    (rate limit) e saturação nos limites operacionais
  - Quando o SCAP está desabilitado, apenas propaga os comandos manuais
- Abstração de fonte de dados (`DataSource` protocol / interface) com implementação
  `SimulatedDataSource` e um stub `OpcUaDataSource` — para que a troca do simulador
  por dados reais de planta não exija nenhuma alteração no frontend.
 
## Stack obrigatória
Frontend:
- React 18 + TypeScript + Vite
- @react-three/fiber, @react-three/drei (CameraControls, OrbitControls, Html, Outlines)
- Zustand para estado
- Recharts para histograma e tendência
- Tailwind CSS + shadcn/ui para HUD
 
Backend:
- Python 3.11+, FastAPI, Pydantic v2, uvicorn
- WebSocket nativo do FastAPI
- Sem banco de dados no POC (buffer em memória para a tendência)
 
## Requisitos não funcionais
- 60 fps na visão geral com os 12 discos girando. Use instancing quando fizer sentido,
  reuse geometrias e materiais, e faça as atualizações de rotação dentro de `useFrame`
  mutando refs — nunca via `setState`.
- Telemetria em Zustand com selectors granulares para não re-renderizar a cena inteira
  a cada tick do WebSocket.
- Tipagem estrita. `strict: true` no tsconfig. Sem `any`.
- Reconexão automática do WebSocket com backoff.
- Estados de erro e de desconexão visíveis na UI (nunca mostrar dado velho como se
  fosse ao vivo — marcar como stale).
- Tema escuro (sala de controle).
- Comentários e nomes de variáveis em português onde forem termos de domínio
  (disco, pelota, alimentação, inclinação, faixa granulométrica).
 
## Entregáveis
1. Estrutura de diretórios completa.
2. Todos os arquivos de código, completos e executáveis — sem `// TODO` e sem trechos
   omitidos.
3. `requirements.txt` / `pyproject.toml` e `package.json`.
4. `README.md` com instruções de execução de backend e frontend.
5. Uma seção final explicando: (a) quais decisões de modelagem do processo você tomou
   e por quê; (b) exatamente onde e como plugar a fonte de dados real (OPC UA ou PI
   Web API) no lugar do simulador.
 
## Método
Antes de escrever qualquer código, apresente:
1. O diagrama da arquitetura (texto/ASCII).
2. Os schemas Pydantic e os tipos TypeScript correspondentes.
3. A lógica do simulador e do controlador PI, descrita em pseudocódigo.
4. A estratégia de performance da cena 3D.
 
Só então implemente. Se qualquer requisito estiver ambíguo, declare a suposição
explicitamente e siga em frente — não faça perguntas.
