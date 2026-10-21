# Inspire-se v1.8 — Android nativo (Kotlin + Room)

App de apoio a momentos de ansiedade: respiração guiada (SOS), registro de humor,
dicas, **chat híbrido** e **lembrete diário**. Banco local (Room/SQLite), sem login.

## Novidades da v1.1

| Função | Como funciona | Precisa de internet? |
|---|---|---|
| Aba **Conversa** (modo guiado) | Árvore de respostas com botões, sem digitar. Pode registrar o humor no mesmo banco do Histórico | Não |
| Aba **Conversa** (modo IA, opcional) | Caixa de texto + IA via seu proxy. Só liga com internet, com aceite do usuário e com o proxy configurado | Sim |
| **Lembrete diário** (aba Dicas) | Notificação local uma vez por dia, no horário escolhido. Tocar nela abre a respiração guiada | Não |

## Estrutura

```
app/src/main/java/com/example/inspirese/
  MainActivity.kt          menu inferior com as abas e abertura pela notificação
  HomeFragment.kt          saudação + registro de humor (5 estados)
  ChatFragment.kt          conversa: offline por padrão, IA opcional (menu ⋮)
  TipsFragment.kt          dicas, contatos de confiança (Ligar, SMS, WhatsApp) e lembrete
  HistoryFragment.kt       histórico e resumo de 7 dias
  SosActivity.kt           respiração guiada 4-7-8
  WelcomeActivity.kt       tela de Boas-vindas
  EmergencyContactsActivity.kt, TipDetailActivity.kt
  chat/                    CrisisDetector, OfflineResponder, AiClient, AiConfig
  data/                    Room: mood_entries, chat_messages, emergency_contacts, activity_entries
  reminder/                ReminderTime (pura, testada), ReminderScheduler, ReminderWorker, Notifier
  sos/BreathCycle.kt       fases da respiração (pura, testada)
  util/                    Prefs, Dialer, Sms, WhatsApp, PhoneValidation
proxy/                     servidor intermediário da IA (Flask)
tests/                     testes da lógica pura e do proxy
```

## Banco de dados

`AppDatabase` está na **versão 4**. Cada migração só cria tabela nova:

- `mood_entries(id, mood, timestamp)` — versão 1. Humor: `otimo`, `bem`, `normal`, `triste`, `muito_mal`.
- `chat_messages(id, role, text, timestamp)` — versão 2 (`MIGRATION_1_2`).
- `emergency_contacts(id, position, name, phone)` — versão 3 (`MIGRATION_2_3`), com índice único em `position` (1 a 3).
- `activity_entries(id, type, label, timestamp, durationSec)` — versão 4 (`MIGRATION_3_4`), alimenta o Histórico.

**O histórico salvo não é apagado nas migrações.** Não use `fallbackToDestructiveMigration`.
O chat grava o humor do dia na mesma tabela `mood_entries` da tela inicial.

## Configurar a IA online (opcional)

O app funciona sem isso (modo offline). A IA só liga depois que um servidor é configurado no app.

1. `cd proxy && pip install -r requirements.txt`
2. Defina a chave **no computador**: `export ANTHROPIC_API_KEY="..."` (PowerShell: `$env:ANTHROPIC_API_KEY="..."`). A chave nunca vai para o app.
3. `flask --app server run --host 0.0.0.0 --port 8000`
4. No app: aba **Conversa** › ⋮ › **Configurar servidor da IA**, e informe o endereço:
   - Emulador: `http://10.0.2.2:8000` (é assim que o emulador enxerga o PC).
   - Celular de verdade: o app recusa HTTP comum fora do emulador (ver
     `res/xml/network_security_config.xml`, que libera HTTP só para `10.0.2.2` e `localhost`).
     Use um endereço **HTTPS**: um túnel para o seu computador ou o proxy publicado num servidor.
5. Depois, ⋮ › **Ligar IA online**. Na primeira vez, o app pede o aceite do envio de dados.

Modelo padrão: `claude-sonnet-5-5`. Para trocar, defina a variável `AI_MODEL` antes de iniciar o proxy.

Por que um proxy: a chave da API não pode ficar dentro do APK, qualquer um consegue ler.
Antes de publicar de verdade, o proxy precisa de limite de uso por usuário e autenticação.

## Segurança e privacidade 

- Texto livre com sinais de crise **não** é enviado à IA: o app responde com CVV 188 / SAMU 192.
  O detector é uma busca por trechos: pode deixar passar frases que não previmos e pode
  disparar por engano. Ele é uma rede de segurança, não um classificador.
- Com a IA ligada, as últimas mensagens da conversa saem do aparelho. Dados de saúde são
  dados pessoais sensíveis pela LGPD: confirme com o professor/responsável o que é exigido.
- O app não substitui acompanhamento profissional (aviso na tela de Dicas e no aceite da IA).

## Lembrete diário: limites

- Usa WorkManager: funciona offline e sobrevive a reiniciar o celular, mas **não é relógio exato**;
  o Android pode adiantar ou atrasar alguns minutos para poupar bateria.
- Android 13+ pede permissão de notificações; sem ela o lembrete não liga.

## Testes

- Lógica pura (respiração, crise, chat offline, telefone, horário do lembrete). Cada arquivo
  em `tests/logica/` tem o seu `main`, então rode um por vez, compilando junto com os fontes
  que ele usa. Exemplo com o detector de crise e o lembrete:
  ```
  S=app/src/main/java/com/example/inspirese
  kotlinc $S/chat/CrisisDetector.kt $S/reminder/ReminderTime.kt tests/logica/CrisisAndReminderTest.kt -include-runtime -d t.jar && java -jar t.jar
  ```
  `BreathCycleTest` usa `sos/BreathCycle.kt`; `OfflineResponderTest` usa `chat/OfflineResponder.kt` e
  `chat/CrisisDetector.kt`; `PhoneValidationTest` usa `util/PhoneValidation.kt`.
- Proxy (com a API simulada, 31 verificações): `pip install flask requests && python3 -I tests/test_proxy.py`
- Os testes do Android (JUnit em `app/src/test/`, com `./gradlew test`) ainda não existem: os acima rodam fora do Android

## Som da tela SOS (opcional)

Coloque `calm_loop.mp3` em `app/src/main/res/raw/`. Sem o arquivo, o app funciona, só sem som.

## v1.2 — Contatos de confiança

3 contatos (nome + telefone) cadastrados na aba Dicas. Cada um tem dois botões:

- **Ligar** — abre o discador com o número já preenchido (`ACTION_DIAL`, sem permissão).
- **SMS** — abre o app de mensagens com o texto "Oi, pode me ligar? Preciso conversar." já
  preenchido; o usuário revê e confirma antes de enviar (`ACTION_SENDTO`, sem permissão).

Nenhum dos dois liga ou envia sozinho — é sempre o usuário quem confirma no app padrão do
celular. Dá para trocar por envio silencioso com `SmsManager.sendTextMessage()`, mas isso pede
a permissão perigosa `SEND_SMS` e manda a mensagem sem o usuário ver antes; mantive o padrão
mais seguro, igual ao `Dialer` que já existia.


## v1.3 — novo visual (tema escuro) + técnica 4-7-8




### Humor: 3 estados → 5

`MoodEntry.mood` passa a guardar `"otimo" | "bem" | "normal" | "triste" | "muito_mal"`
(antes era `"bem" | "ansiedade" | "panico"`). **Não precisou de migração do Room** — a coluna
já era TEXT livre, só mudam os valores aceitos. HistoryFragment/HistoryAdapter foram
atualizados para os 5. O chat guiado (ChatTree.kt) não mudou nada: ele só usa o enum
`ChatAction`, e é o `ChatFragment` quem traduz cada ação para um dos 5 valores novos
(`LOG_BEM→"bem"`, `LOG_ANSIEDADE→"triste"`, `LOG_PANICO→"muito_mal"`).

### SOS: a lógica de fase virou uma classe testável

Extraí a matemática da respiração para `sos/BreathCycle.kt` — uma função pura
`stateAt(elapsedMs)` que devolve fase, segundos restantes e escala do círculo. A
`SosActivity` ficou só "lê o relógio a cada 50ms e desenha o que o BreathCycle manda".
24 testes cobrem as transições exatas entre as 3 fases (`tests/logica/BreathCycleTest.kt`).
Um bug pequeno do ciclo antigo foi corrigido de quebra: o contador mostrava "5,4,3,2,1" num
intervalo de 4 segundos (um número a mais); agora mostra exatamente "4,3,2,1".

### SOS agora é uma aba, não mais um botão flutuante

`MainActivity` trata o item "SOS" do menu como um caso especial: abre a `SosActivity` por
cima da tela atual e devolve `false` no listener, então o menu continua marcando a aba de
conteúdo que já estava selecionada (o SOS nunca "fica marcado" depois de fechar). Os
ladrilhos de "Acesso rápido" da Home chamam `(activity as? MainActivity)?.selectTab(id)`
para trocar de aba por código.



## v1.4 — correção de contraste (bugs reais, achados por auditoria de cor)

Você reportou "não consigo ver as letras direito" e "não consigo acessar o chat pra
conversar". Encontrei e corrigi, com números — calculei a razão de contraste (fórmula do
WCAG) de todo par texto/fundo do app:

| Onde | Antes | Depois |
|---|---|---|
| **Botões de resposta do chat guiado** (`item_chat_option.xml`) | texto quase preto sobre fundo roxo-escuro: **1.14:1** | branco sobre roxo cheio: **4.86:1** |
| Fase/contador por cima do círculo de respiração | **3.00:1** (no limite mínimo, e o círculo ainda anima) | **4.38:1** |
| Botões de ação primária em geral (Salvar, Ligar, Estou melhor, Enviar) | **3.33:1** (abaixo do recomendado p/ texto normal) | **4.86:1** |

(Referência: WCAG pede ≥4.5:1 para texto normal e ≥3:1 para texto grande/negrito.)


 c
## v1.5 — chat virou conversa livre (sem botões)

Você pediu pra tirar os botões de resposta rápida ("Estou bem", "Pânico" etc.) e deixar só a
conversa de texto, como na referência. Feito — e isso mudou a arquitetura do chat, não só a
tela:

- **Removido:** `chat/ChatTree.kt` (a árvore de nós/botões), `item_chat_option.xml`, o estilo
  `ChatOptionButton`, e as funções `chatNode`/`setChatNode` do `Prefs.kt` (só existiam pra
  lembrar "em que nó da árvore" a conversa estava).
- **Novo:** `chat/OfflineResponder.kt` — não é IA, é um conjunto de respostas fixas escolhidas
  por palavra-chave (ansiedade, pânico, tristeza, "quero respirar", humor positivo, saudação
  pura, e uma resposta genérica de fallback). É quem responde por padrão, sem precisar de
  internet nem de proxy configurado — mantém a regra original do projeto ("funciona 100%
  offline") mesmo com o chat agora sendo livre.
- O `CrisisDetector` continua rodando **antes** de tudo: texto de crise nunca chega no
  `OfflineResponder` nem na IA, vai direto pra resposta fixa com CVV/SAMU.
- A IA online (opcional, com proxy configurado) continua existindo do mesmo jeito de antes —
  agora ela só substitui o `OfflineResponder`, não substitui mais um "modo", já que não tem
  mais modo nenhum de botão pra trocar.
- **Efeito colateral que vale saber:** antes, tocar em "Estou bem" no chat também registrava
  o humor no Histórico. Sem botões, essa ponte sumiu — o Histórico só é alimentado pela Home
  agora. Se quiser esse registro de volta a partir da conversa livre, dá pra fazer o
  `OfflineResponder` sugerir "quer registrar esse humor?" como próximo passo, mas prefiro não
  inferir e salvar sozinho sem confirmação sua.
- 12 testes novos no `OfflineResponderTest.kt`, cobrindo cada categoria de palavra-chave e o
  caso de borda que quase me pegou: "Bom dia, tudo bem?" não pode ser lido como "humor
  positivo" só porque tem "bem" dentro — tratei saudação pura como categoria própria.

## v1.6 — visual no nível da referência + IA mais inteligente

Você mandou a imagem de referência (telas: Início, SOS, Conversa, Dicas, Dica, Histórico) e
pediu pra levar o app pra esse nível de acabamento, além de deixar a IA do chat mais
inteligente. As pendências que ficaram em aberto desde a v1.3 ("telas de Dicas e Histórico
ainda usam a estrutura antiga", "Conversa só herdou as cores novas") são resolvidas aqui.

### Telas novas / redesenhadas

- **Boas-vindas** (`WelcomeActivity`, nova): primeira tela ao abrir o app — fundo em degradê
  escuro, brilho atrás do logo, nome do app e "Começar". Só uma porta de entrada, sem lógica
  nem estado — por isso não tem teste próprio. `MainActivity` deixou de ser a tela de
  LAUNCHER (virou `exported="false"`, só é aberta de dentro do app).
- **SOS** (`activity_sos.xml` + `SosActivity.kt`, reescritas): agora tem 3 estados — parado
  (card da técnica 4-7-8 + "Começar"), em andamento (anel de progresso
  `CircularProgressIndicator` mostrando quanto falta da fase atual, 4 pontos marcando o
  ciclo, "Pausar") e pausado ("Continuar"). O círculo que só mudava de escala (`bg_orb`) virou
  só o estado parado; o anel ativo é o que a referência mostra ("Expire / 8s" com a barra
  circular). "Pausar" de verdade pausa o cronômetro (não só a tela) — sair e voltar ao app no
  meio da respiração retoma de onde parou, sem contar o tempo em segundo plano.
- **Conversa** (`fragment_chat.xml` + `ChatFragment.kt`): cabeçalho enxuto (← Conversa · SOS ·
  ⋮) — "IA online" e "Limpar conversa" viraram itens do menu ⋮ em vez de ocupar uma chave e
  um botão fixos na tela. Balões do bot ganharam avatar.
- **Dicas** (`fragment_tips.xml` + `TipsFragment.kt`, `TipsAdapter` novo): abas **Todos /
  Saúde Mental / Bem-estar** filtram a lista; cada dica agora é uma linha com ícone + resumo
  + seta, e tocar abre uma **tela de detalhe** (`TipDetailActivity`, nova) com ícone grande,
  texto de abertura e passos numerados — exatamente o padrão "lista → detalhe" da referência.
  O conteúdo das 8 dicas (texto que já existia) foi reorganizado em `data/Tip.kt` com um passo
  a passo pra cada uma, sem inventar conteúdo novo.
- **Histórico** (`fragment_history.xml` + `HistoryFragment.kt` + `HistoryAdapter` novo): além
  dos cartões de resumo dos últimos 7 dias (que já existiam), a lista agora é uma **linha do
  tempo de verdade** — respiração, conversa e dica lida, agrupadas por "Hoje, `<data>`" /
  "Ontem, `<data>`" / data completa, cada linha com um ícone colorido por tipo (pulmão azul =
  respiração, balão verde = conversa, lâmpada âmbar = dica) e duração quando faz sentido
  ("14:32 · 4 min"). Tocar numa linha abre os detalhes num diálogo.
- Ícones vetoriais próprios (`ic_home`, `ic_sos`, `ic_chat`, `ic_tips`, `ic_history`,
  `ic_back`, `ic_more`, `ic_send`, `ic_chevron_right`, `ic_bot`, `ic_logo_heart`) substituem os
  ícones padrão do Android no menu inferior e os emojis nos ladrilhos da Home — mesma ideia
  visual da referência (traço fino, monocromático, tingido pela cor do contexto).

### Histórico agora é alimentado por 3 fontes, não só pelo humor

Nova tabela `activity_entries(id, type, label, timestamp, durationSec)` — Room sobe pra
**version = 4** (`MIGRATION_3_4`, só cria a tabela nova; nada é apagado). `SosActivity` grava
um registro ao sair (se a respiração ativa durou 5s ou mais), `TipDetailActivity` grava ao
abrir uma dica, e `ChatFragment` grava ao sair da aba Conversa (se pelo menos uma mensagem foi
enviada). `MoodEntry` continua existindo só para os cartões de resumo de humor da Home — as
duas tabelas não se misturam.

**Bug que encontrei e corrigi nessa auditoria:** o `ChatFragment` já tinha os campos
`sessionStart`/`sentAnyThisSession` e um comentário apontando pro `onDestroyView` como o lugar
que devia gravar a conversa no Histórico — mas o `onDestroyView` só limpava o binding, nunca
chamava o `activityDao`. Resultado: nenhuma conversa aparecia no Histórico, mesmo depois de
mandar várias mensagens. Implementei o `logSessionIfNeeded()` que faltava e marquei
`sentAnyThisSession = true` no envio da mensagem — agora a conversa é registrada como as
outras duas atividades.

### IA do chat: mais inteligente, não só "ligada"

- **Modelo padrão trocado** de `claude-haiku-4-5-20251001` para `claude-sonnet-5`
  (`proxy/server.py`, variável `AI_MODEL`) — Sonnet entende nuance e contexto bem melhor que
  Haiku, o que importa numa conversa de apoio emocional. Pra mais qualidade ainda (com mais
  custo/latência), dá pra trocar por `claude-opus-5-5` só mudando a variável de ambiente
  `AI_MODEL`, sem mexer em código.
- **Prompt de sistema reescrito** para parecer menos um roteiro genérico: agora pede
  explicitamente pra validar o que a pessoa sente antes de sugerir qualquer coisa, fazer no
  máximo uma pergunta por resposta (em vez de bombardear com perguntas), e mencionar o SOS ou
  as Dicas só quando fizer sentido, não em toda resposta. As regras de segurança (nunca dizer
  que é profissional de saúde, nunca diagnosticar, sempre indicar CVV/SAMU em crise, não pedir
  dados pessoais) continuam exatamente como antes.
- **Contexto do humor do dia** (opcional): se a pessoa registrou o humor na Home hoje, o
  `ChatFragment` manda um resuminho de uma linha (ex.: "Humor que a pessoa registrou hoje no
  app: triste.") junto com a mensagem — o proxy acrescenta isso ao prompt de sistema só como
  pano de fundo pro tom da resposta, nunca como instrução, e nunca é tratado como dado vindo
  da pessoa. Nenhum outro dado (nome, histórico completo, mensagens antigas além das 12 já
  enviadas) sai do aparelho — mesma regra de privacidade de antes.




### O que NÃO veio nesta leva

- A tela de **Contatos de confiança** (`EmergencyContactsActivity`) manteve o visual mais
  simples (cabeçalho com texto "Fechar" em vez do padrão ← + título das outras telas) — não é
  uma das telas da referência, então não priorizei redesenhá-la agora. Funciona normalmente.
- O anel de progresso da respiração ainda usa o componente padrão do Material Design (cor e
  espessura customizadas); a referência parece ter um gradiente sutil no traço — dá pra




##

Sobre o `applicationId`: isso é o identificador único do app pro Android (tipo um CPF do
app), não só um texto, porque "Inspire-se" é um app diferente do
"Respira" na visão do sistema agora. Na prática isso quer dizer que instalar este APK não vai
*atualizar* uma instalação antiga do Respira no celular de ninguém — vai instalar como um app
separado, do lado do antigo. Se vocês tinham o Respira instalado em algum aparelho de teste e
querem só um dos dois, desinstalem o antigo manualmente. Pareceu a escolha certa pra uma
troca de identidade "de verdade" (é literalmente um rebranding, não só um ajuste de texto),
mas é uma decisão que dá pra reverter (mantendo `com.example.respira`) se preferirem que uma
instalação nova simplesmente substitua a antiga.

### Logo nova

A logo veio como ícone adaptativo (`ic_launcher_foreground.xml` + `ic_launcher_background.xml`,
usados tanto no ícone do launcher quanto na tela de Boas-vindas) — recriei como vetor,
porque um ícone adaptativo de verdade (que o Android recorta em círculo, quadrado arredondado
etc., dependendo do celular) precisa ser vetor, não uma imagem fixa colada. É a mesma
composição da arte que você mandou: duas pétalas azuis nas laterais (gradiente `#2979FF` →
`#4FC3F7`) e uma pétala roxa maior no centro, por cima (gradiente `#7C4DFF` → `#B388FF`), com
um ponto branco perto do topo da pétala central. Esse mesmo drawable é reusado na tela de
Boas-vindas, então a logo do ícone e a logo dentro do app são sempre a mesma arte — muda a
logo num lugar só (`ic_launcher_foreground.xml`) se precisar ajustar de novo.

### O `.rar` estava certo: o erro foi na minha extração

Na primeira leitura do `.rar` (`5036d9fd-Inspire-se_android.rar`) eu usei ferramentas de extração
que não leem bem este formato (RAR5), como `unar` e `7z`. Vários arquivos saíram **vazios**
(0 bytes). Eu atribuí isso ao Windows sem verificar, e estava errado. Extraí de novo com `unrar`,
e todos os arquivos vieram com o tamanho certo (`unrar t` deu "All OK"). O `.rar` estava íntegro.



## v1.8 — IA do chat melhor, WhatsApp nos contatos, Acesso rápido removido

Pedidos desta versão: melhorar a IA integrada do chat, remover o "Acesso rápido" da Home,
pôr um botão de WhatsApp nos 3 contatos de emergência e gerar o APK.

### Acesso rápido removido



### WhatsApp nos contatos

Cada um dos 3 contatos (aba Dicas) agora tem **Ligar · SMS · WhatsApp**. O botão WhatsApp é
outlined, com texto e contorno verdes `#25D366`. Sobre o fundo do cartão (`#1E1B2E`) o contraste
é 8,5:1, acima dos 4,5:1 do WCAG. O rótulo "🟢 WhatsApp" cabe em uma linha e encolhe sozinho em
celulares pequenos. O botão abre `api.whatsapp.com/send` com a mensagem
"Oi, pode me ligar? Preciso conversar." e, se o número tiver 10 ou 11 dígitos sem `+`, põe o DDI 55.

### IA do chat

Mudanças no servidor (`proxy/server.py`):

- **Modelo padrão** `claude-sonnet-5-5` (antes `claude-sonnet-5`). Troque com a variável `AI_MODEL`.
- **Prompt reescrito**, com estas regras: acolher antes de aconselhar; retomar o que a pessoa já
  contou (continuidade); uma ideia e no máximo uma pergunta por resposta; texto corrido de até
  4 frases, sem listas (é uma tela de celular); uma técnica simples por vez (respiração lenta ou
  5 sentidos); citar só as abas que o app tem (SOS e Dicas); lembrar UBS e CAPS pelo SUS quando a
  angústia é longa; nunca se apresentar como psicólogo ou médico, nunca diagnosticar, nunca
  indicar remédio, nunca pedir dados pessoais; em crise, CVV 188 e SAMU 192.
- **Limite de resposta** em `MAX_TOKENS = 500` (o teste agora bate com o código).
- **Log de erro** do servidor: status da IA e um trecho do erro. Nunca a chave nem a conversa.

Mudanças no app:

- **Configurar o servidor pelo app.** Antes, nada gravava o endereço do proxy, e o item
  "Ligar IA online" só aparece com um servidor configurado. Resultado: a IA nunca ligava.
  Agora: ⋮ › **Configurar servidor da IA**. A chave da API continua só no servidor.

Não testei a IA ao vivo: aqui não há acesso à API da Anthropic. O teste real é rodar o proxy com
a sua chave e conversar pelo emulador ou por um endereço HTTPS.

### Logo na tela de Boas-vindas

O logo ficou em um disco claro (`drawable/bg_welcome_logo_disc.xml`), não no roxo da marca. As
pétalas são azuis e roxas, então no roxo elas perdiam contraste. A tagline voltou a ser
"Um lugar para ser ouvido." e o nome vem de `@string/app_name`.

### Verificação

- **Recursos:** 64 arquivos XML bem formados. Todas as referências `@color/`, `@string/`,
  `@style/`, `@drawable/`, `@layout/`, `@id/` estão definidas. Nenhum resto de Acesso rápido.
- **Lógica pura:** os quatro conjuntos (`BreathCycleTest`, `OfflineResponderTest`,
  `PhoneValidationTest`, `CrisisAndReminderTest`) compilam com o `kotlinc` oficial e passam.
- **Proxy:** `python3 -I tests/test_proxy.py`, 31 verificações, todas passando.
- **Não compilei o app Android aqui.** `ChatFragment.kt`, `HomeFragment.kt` e `MainActivity.kt`
  precisam do SDK do Android, que não está neste ambiente. O que falta é o build no Android Studio.

### Gerar o APK

No Android Studio: **Build › Build Bundle(s) / APK(s) › Build APK(s)**. O arquivo sai em
`app/build/outputs/apk/debug/app-debug.apk`. Para um APK assinado de release:
**Build › Generate Signed Bundle / APK**.
