# Guia Treino — Resumo do Projeto

## Visão Geral

PWA (Progressive Web App) de treino pessoal para Roberto Garcia. Hospedado em `https://robgarcia82.github.io/personaltrainer/`. Arquivo único: `index.html` (~740 linhas) + `sw.js` (service worker). Nenhum build step — React 18 via CDN UMD, tudo inline.

**Stack:** React 18 UMD, `h = React.createElement`, `useState/useEffect/useRef`, localStorage para persistência, Web Audio API para alarmes, PWA Service Worker (cache-first, versão atual: `guia-treino-v19`).

---

## Estrutura de Abas

O app tem 3 abas fixas no rodapé:
- **Treino** — fluxo principal de execução
- **Dieta** — plano alimentar semanal
- **Coach** — periodização e regras de progressão

---

## Aba Treino — Fluxo de Navegação

### Tela 1: Home (SessionHomeView)
Painel de uma tela, de cima pra baixo:

1. **Seletor de programa** — Split, Full Body e Grupos (a lista sai de `PROGRAMS`).
2. **Faixa da semana** — segunda a domingo, com estado por dia: treino concluído,
   hoje, ainda não chegou, sem treino. Sai do log real, filtrada pelo programa ativo.
3. **Carrossel de treinos** — todos os treinos do programa, deslizando na horizontal.
   O recomendado abre a fila com o rótulo de status (Para hoje / Pendente / Descanso /
   Próximo treino) e os demais são rotulados pela posição no ciclo.
4. **Cards de progresso** — últimos 28 dias (contagem + grade dia a dia), volume
   semanal (Σ carga × reps, janelas móveis de 7 dias), PSE médio do mês e maior evolução.

Os treinos de cada programa vivem em `SESSIONS`; para o Split são Superiores 1
(Peito + Tríceps), Superiores 2 (Costas + Bíceps), Superiores 3 (Ombros),
Inferiores (Pernas + Core) e Bônus (Pull-ups, fora do ciclo).

**A recomendação é por ciclo, não por calendário:** depois do primeiro treino o app
deriva o próximo item de `TRAINING_CYCLE` a partir do último concluído. O dia da
semana só decide a recomendação no primeiro uso, quando ainda não há histórico.

### Tela 2: Grid de Exercícios (ExerciseGridView)
Grid 2 colunas com cards de exercício (imagem + nome + músculo). Header sticky com filtros de grupo muscular (Peito, Costas, Ombro, Pernas, Braços, Core). Sem campo de busca.

### Tela 3: Detalhe/Execução do Exercício (TreinoView — estado selEx)

Header sticky com: botão voltar, contador "N / Total", nome do exercício, badge de equipamento.

**Área de controle (topo):**
- Seletor DESCANSO: botões 45s / 60s / 90s / 120s (só configura, não inicia o timer)
- Botão principal: "Iniciar Série" — inicia countdown 3-2-1 e depois inicia a série
- Quando allSeriesDone (séries completadas >= planejadas): vira "Próximo Exercício" (verde) ou "Treino Concluído" (roxo)

**Conteúdo (scrollável):**
1. Última série — card com carga (kg), duração, PSE colorido + data. Se primeiro treino, placeholder.
2. Sugestão — aparece após 1ª série: "Reduza a carga" (PSE>=9) / "Pode aumentar" (PSE<=5) / "Mantenha o foco" (demais)
3. Meta de hoje — chips de bloco (B1/B2/B3/Deload) + prescrição do bloco selecionado (ex: "4x8-10 | PSE 7-8")
4. Foco da série — card verde com dica de execução específica
5. Como executar — passo a passo numerado
6. Imagem demonstrativa (180px, carregada de CDN externa)
7. Histórico de séries — grid 4 colunas com últimas 4 sessões (PSE colorido, duração, carga, data)

**Botão fixo "Próximo exercício"** — aparece a partir da 3ª série completa (seriesCt >= 3), fixo acima da navbar.

**Overlays (por ordem de zIndex):**
- Countdown (z:500) — 3-2-1 com cor da sessão, botão Cancelar
- Timer de descanso (z:200) — fullscreen escuro, cronômetro regressivo grande, botões play/pause/reset/fechar. Fecha automaticamente ao chegar em 0 (vibração + alarme 880Hz triplo beep)
- Barra de série em andamento (z:200) — strip na parte inferior com cronômetro crescente + botão "Finalizar Série"
- Tela de Carga (z:550) — stepper +/-0.5kg, atalhos rápidos 5–50kg em grid 5x2, pré-preenche com última carga usada
- Modal PSE (z:600) — escala 1–10 com cores (verde para vermelho), mostra carga selecionada em leitura

**Salvar série:** registra em localStorage["workout_log"] com {id, ts, date, session, exerciseId, exerciseName, duration, pse, carga}. Após salvar: inicia timer de descanso automaticamente, incrementa seriesCt.

---

## Exercícios (26 no total)

Cada exercício tem: id, n (nome), m (músculo), s (sessão), d (dia), f (grupo/filtro), eq (equipamento), st[] (passos de execução), t (foco/dica do coach), e (erros comuns), b1/b2/b3/dl (prescrição por bloco).

**Equipamentos:** mc1000 (polia/multifuncional), halter, sandbag, corporal

**Sessões e exercícios:**
- Superiores 1 (6 ex): Supino Halteres, Supino Inclinado, Voador Polia, Dips, Tríceps Pushdown, Tríceps Francês
- Superiores 2 (6 ex): Lat Pulldown, Cable Row, Remada Halter, Straight Arm Pulldown, Rosca Alternada, Cable Curl
- Superiores 3 (6 ex): Desenvolvimento Halteres, Arnold Press, Elevação Lateral Polia, Face Pull, Elevação Frontal Polia, Crucifixo Invertido Polia
- Inferiores (6 ex): Agachamento Búlgaro, Terra Sumo, Stiff, Glute Bridge (sandbag 20kg), Panturrilha, Hanging Leg Raise
- Bônus (2 ex): Pull-Up Pronada, Chin-Up Supinada

---

## Periodização (12 semanas)

| Bloco            | Semanas | Séries | Reps            | PSE |
|------------------|---------|--------|-----------------|-----|
| Bloco 1 Volume   | 1–4     | 3      | 10-12 / 12-15   | 6-7 |
| Bloco 2 Intensid.| 5–8     | 3-4    | 8-10 / 10-12    | 7-8 |
| Bloco 3 Carga    | 9–11    | 4      | 6-8 / 10-12     | 8-9 |
| Deload           | 12      | 2      | 10-12 / 12-15   | 5-6 |

---

## Aba Dieta

- Meta: 2.200 kcal, 154g proteína, 258g carb, 62g gordura (déficit ~350 kcal, TDEE 2.565)
- Nota sobre Mounjaro: priorizar proteína mesmo sem fome, 2,5L+ água
- Plano por dia da semana (accordion): 5 refeições/dia com hora, itens, macros
- Regras do plano (6 regras)
- Suplementos: Whey 30g, Creatina 5g, Vitamina D3, Ômega 3

---

## Aba Coach

- Agenda semanal (lista com dias treino vs descanso)
- Periodização 12 semanas (accordion com séries/reps/PSE/foco por bloco)
- Regras de progressão (5 regras)
- Descanso entre séries por tipo
- Progressão bônus para Pull-Ups (negativas → assistido → completo → com peso)

---

## Estado Principal (TreinoView)

```
af         — filtro de grupo muscular ativo
sb         — bloco selecionado (ex: "1b2" = exercício id 1, bloco b2)
selEx      — exercício selecionado (objeto completo)
st         — timer de descanso ativo (overlay visível)
ts         — duração configurada do descanso (padrão: 90s)
tr         — timer rodando (true) vs pausado (false)
tl         — tempo restante no timer
selS       — sessão selecionada (string, ex: "Superiores 1")
cntd       — countdown (3, 2, 1, null quando inativo)
exRun      — série em andamento
exTime     — tempo da série em andamento em segundos (crescente)
showPse    — modal PSE visível
lastDur    — duração da série que acabou de finalizar
carga      — carga selecionada em kg (incremento 0.5)
showCarga  — tela de seleção de carga visível
seriesCt   — contador de séries completadas neste exercício (reseta ao trocar de exercício)
```

**Variáveis computadas:**
```
blocoKey      — chave do bloco selecionado (b1/b2/b3/dl) ou null
plannedSeries — número de séries planejadas no bloco (parseado do campo ex[blocoKey])
allSeriesDone — seriesCt >= plannedSeries && plannedSeries > 0
nextEx        — próximo exercício na sessão (ou null se for o último)
exLog         — últimas 4 séries deste exercício (localStorage, mais recente primeiro)
```

---

## Constantes Globais

```
SC        — cores por sessão: { "Superiores 1": {t:"#ef4444", bg:"#2a0808"}, ... }
FC        — cores por grupo muscular
EB        — estilos por equipamento: { mc1000, halter, sandbag, corporal }
BC        — cores dos 4 blocos: ["#1e40af","#ef4444","#7c3aed","#16a34a"]
BL        — ["Bloco 1","Bloco 2","Bloco 3","Deload"]
BK        — ["b1","b2","b3","dl"]
fmtT(s)   — formata segundos em "M:SS"
BASE      — URL base para imagens (GitHub CDN free-exercise-db)
EX_IMGS   — mapa { id: URL } para imagem de cada exercício
DIET      — objeto com kcal, macros, plano semanal, regras, suplementos
COACH     — objeto com agenda, blocos, progressão, descanso, bônus
SESSIONS  — array com as 5 sessões (s, dia, sub, img, cor)
EX        — array com todos os 26 exercícios
```

---

## Fluxo de uma Série Completa

1. Usuário abre exercício, seleciona bloco (Bloco 1/2/3/Deload) e tempo de descanso
2. Clica "Iniciar Série" → countdown 3-2-1 (vibra a cada tick) → série inicia
3. Cronômetro crescente aparece na barra inferior
4. Clica "Finalizar Série" → tela de Carga aparece (pré-preenchida com última carga)
5. Ajusta carga e clica "Continuar" → modal PSE aparece (1–10)
6. Seleciona PSE → série salva no localStorage → timer de descanso inicia fullscreen
7. Timer chega a 0 → alarme toca (880Hz, 3 beeps) + vibra → overlay fecha automaticamente
8. seriesCt incrementa; se atingiu planejado, botão principal vira "Próximo Exercício"
9. Após 3ª série, botão fixo "Próximo exercício" aparece na parte inferior

---

## Arquivos do Projeto

| Arquivo              | Descrição                                      |
|----------------------|------------------------------------------------|
| index.html           | App completo (~740 linhas), tudo inline        |
| sw.js                | Service Worker, cache guia-treino-v17          |
| manifest.json        | PWA manifest                                   |
| icons/icon-192.png   | Ícone PWA                                      |
| icons/icon-512.png   | Ícone PWA                                      |

**Deploy:** git push origin main → GitHub Pages em https://robgarcia82.github.io/personaltrainer/

**localStorage:** chave "workout_log", array de entradas de série salvas.
