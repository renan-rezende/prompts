Você é um engenheiro de software sênior especializado em visualização 3D industrial e gêmeos digitais. Crie do zero um **gêmeo digital interativo dos 12 discos de pelotamento de uma usina de pelotização de minério de ferro**, rodando no navegador.

Antes de codar, pense profundamente e apresente brevemente o plano de arquitetura (estrutura de pastas, componentes, modelo de dados, fluxo de estado). Depois implemente TUDO sem parar para perguntar — assuma decisões razoáveis e documente-as no README.

## Stack obrigatória
- Vite + React 18 + TypeScript (modo strict, sem `any`)
- Three.js via @react-three/fiber
- @react-three/drei (CameraControls, Html, etc.)
- Zustand para estado global
- Gráfico de granulometria: barras em SVG/CSS puro (sem lib pesada)
- SEM backend nesta fase: dados vêm de um simulador interno, mas atrás de uma interface `DataSource` para futura integração com dados reais (PI Web API / OPC UA / WebSocket)

## Contexto de domínio (leia com atenção)
- A usina tem **12 discos de pelotamento**: pratos giratórios inclinados (~7,5 m de diâmetro) que transformam minério de ferro úmido em pelotas verdes.
- Variáveis manipuláveis por disco: **velocidade de rotação (rpm)** e **inclinação (graus)**. Elas mudam o tempo de residência do material e, portanto, o tamanho das pelotas.
- **Alimentação (t/h)**: taxa de material chegando ao disco.
- **Granulometria** medida em 7 faixas (mm): 0–6, 6–8, 8–9, 9–12, 12–16, 16–19, >19. Cada faixa é um percentual; a soma é 100%.
- **PV (Process Variable)** = percentual da faixa 9–12 ÷ percentual da faixa 12–16. Clamp entre 0 e 2.
- **SP (Set Point)** = valor-alvo de PV definido pelo operador. Varia de 0 a 2.
- **SCAP (Sistema de Controle Avançado de Processo)**: controle automático que ajusta velocidade e inclinação para levar o PV até o SP. Cada disco tem flag SCAP habilitado/desabilitado:
  - SCAP habilitado (AUTO): o sistema ajusta velocidade/inclinação sozinho; o operador só define o SP.
  - SCAP desabilitado (MANUAL): operador ajusta velocidade e inclinação diretamente. Na planta real, desliga-se o SCAP quando a alimentação está muito ruim ou muda drasticamente.

## Requisitos funcionais

### 1. Cena geral (overview)
- 12 discos em 3D, duas fileiras de 6 (layout em arquivo de constantes), em um galpão industrial estilizado (chão, iluminação industrial, névoa leve).
- Cada disco: modelo procedural em Three.js (cilindro inclinado com borda), com rotação visual animada proporcional à rpm real.
- Badge flutuante sempre visível acima de cada disco (drei `Html`): "Disco 01"…"Disco 12", status SCAP (AUTO verde / MANUAL laranja) e mini-indicador do desvio |PV−SP|.
- CameraControls com limites (sem atravessar o chão).

### 2. Seleção e câmera
- Clique no disco ou badge → animação suave de câmera (~1,2 s, com easing) que voa até o disco e o isola (demais discos com opacidade reduzida).
- Hover: highlight sutil (emissive) + cursor pointer.
- Tecla ESC ou botão "← Visão geral" retorna à câmera inicial com animação.

### 3. Painel do disco selecionado (HUD overlay em HTML/CSS, fora do canvas)
- Cabeçalho: "Disco NN" + toggle interativo SCAP (AUTO/MANUAL).
- Se SCAP habilitado: exibir **PV** (calculado, leitura) e **SP** (input numérico 0–2, passo 0,01), com indicador de desvio colorido: |PV−SP| ≤ 0,05 verde; ≤ 0,15 amarelo; > 0,15 vermelho.
- Se SCAP desabilitado: ocultar SP e exibir **sliders de velocidade e inclinação** para controle manual (manter PV como leitura, pois a granulometria continua sendo medida).
- Sempre exibir: velocidade (rpm, 1 casa decimal), inclinação (graus, 1 casa), alimentação (t/h, inteiro).
- **Gráfico de barras da granulometria** com as 7 faixas (soma 100%), destacando visualmente as faixas 9–12 e 12–16 (as que formam o PV).

### 4. Simulador de processo (mock realista)
- Loop de simulação (tick de ~1 s) por disco:
  - Alimentação: random walk suave em torno de um valor base (ex.: 100 ± 15 t/h), com eventos ocasionais de "material ruim" (queda/pico) que perturbam a granulometria.
  - Granulometria: modelo simplificado que responde às variáveis — dentro dos limites, mais velocidade/inclinação desloca a distribuição para pelotas maiores até um ponto ótimo. Documente as suposições em comentários. Sempre normalizar para 100%.
  - PV derivado: PV = faixa(9–12)/faixa(12–16), clamp 0–2.
  - SCAP AUTO: controlador PI discreto ajusta velocidade e inclinação gradualmente (com limites e taxa máxima de variação — atuadores reais não saltam) para levar PV → SP.
  - SCAP MANUAL: velocidade/inclinação ficam onde o operador deixou; o PV deriva conforme alimentação/granulometria variam.
- Limites plausíveis num único `config.ts` (comentar que devem ser calibrados com dados reais da planta): velocidade 4–9 rpm, inclinação 44–50°, alimentação 60–140 t/h.

### 5. Modelo de dados e abstração
```ts
interface GranulometriaPct { f0_6: number; f6_8: number; f8_9: number; f9_12: number; f12_16: number; f16_19: number; f19p: number } // soma 100
interface DiscState {
  id: number;              // 1..12
  scapEnabled: boolean;
  sp: number;              // 0..2
  pv: number;              // 0..2 (derivado)
  speedRpm: number;
  inclinationDeg: number;
  feedTph: number;
  granulometry: GranulometriaPct;
  updatedAt: number;
}
interface DataSource {
  subscribe(cb: (discs: DiscState[]) => void): () => void;
  setScap(id: number, enabled: boolean): void;
  setSp(id: number, sp: number): void;
  setManual(id: number, speedRpm?: number, inclinationDeg?: number): void;
}
```
- Implementar `SimulatedDataSource` agora. A UI conversa APENAS com `DataSource`, para no futuro trocar por PI Web API / OPC UA sem alterar componentes.

### 6. Visual e UX
- Tema dark industrial (fundo grafite, acentos ciano/âmbar), tipografia legível à distância (uso em TV de sala de controle, 1080p/4K).
- Unidades sempre visíveis; textos da interface em pt-BR.
- Alvo de 60 fps.

### 7. Qualidade e entrega
- Componentes pequenos, estado centralizado no Zustand, estrutura organizada.
- README.md: como rodar (`npm i && npm run dev`), arquitetura, modelo de simulação e onde plugar dados reais.
- Ao final: rode o projeto, verifique erros de console/TypeScript e corrija tudo antes de concluir.

## Critérios de aceite
1. Abro o app e vejo os 12 discos girando com badges de status.
2. Clico no Disco 07 → câmera anima e isola o disco; painel mostra SCAP, PV/SP (se AUTO), velocidade, inclinação, alimentação e o gráfico das 7 faixas.
3. Mudo o SP → em ~10–30 s de simulação o PV converge suavemente para o SP, com velocidade/inclinação se movendo de forma gradual.
4. Desligo o SCAP → sliders manuais aparecem; mexo na velocidade → granulometria e PV respondem.
5. ESC retorna à visão geral com animação.
