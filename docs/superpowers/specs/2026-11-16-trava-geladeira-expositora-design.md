# Trava para geladeira expositora — spec inicial

**Data:** 2026-11-16
**Status:** rascunho inicial (decisões marcadas como *proposta* ainda não foram validadas)
**Referência visual:** [diagrama.jpg](../../../diagrama.jpg)

## 1. Intenção

Trava eletromecânica, projetada do zero, para geladeira expositora de porta de vidro. A porta abre de duas formas:

- **pelo app** (celular), com autorização;
- **por chave física**, sem depender da eletrônica.

O sistema funciona só com bateria, sem ligação à tomada da geladeira.

**Por quê:** permitir **autoatendimento**. O cliente abre a geladeira sozinho, sem precisar de alguém da loja, e o dono controla quem abriu e quando.

**Fases de uso:**
- **Fase 1 — só o dono:** o dono abre pelo app (reposição, testes, operação) e pela chave.
- **Fase 2 — clientes:** clientes abrem pelo próprio celular, com permissão concedida pelo dono.

A fase 1 precisa nascer preparada para a fase 2 (ver §6.4): o protocolo de autorização não pode ter que ser refeito quando os clientes entrarem.

**Contexto de quem constrói:**
- É a **primeira vez** construindo algo assim (eletrônica, firmware e app). O projeto também é um aprendizado e quer dar certo nessa tentativa.
- **Orçamento apertado:** a construção vai **por partes** (§12), cada etapa barata, testável sozinha e útil para a próxima. Só se compra a peça da etapa seguinte depois que a anterior funcionar.
- Consequência: em cada decisão, preferir peças **baratas, comuns no Brasil e com muito tutorial disponível**, mesmo que não sejam as ideais para produção.

## 2. Objetivos

- Manter a porta travada por padrão e destravar sob comando do app.
- Abrir sempre pela chave mecânica, inclusive com a bateria descarregada.
- Fazer a bateria durar meses entre recargas ou trocas (meta: **≥ 6 meses**, *proposta*).
- Avisar no app quando a bateria estiver baixa, com folga para agir antes de acabar.
- Instalar sem furar a câmara refrigerada nem comprometer a vedação da porta.

## 3. Fora de escopo (por enquanto)

- Alimentação pela tomada ou pela geladeira.
- Controle remoto pela internet, de longe (ver §6.2).
- Pagamento ou venda integrada. O autoatendimento é o objetivo, mas a forma como o cliente paga ainda não foi definida (ver §10).
- Identificar o que o cliente retirou (câmera, balança, RFID).
- Monitoramento de temperatura.

## 4. Antiobjetivos (o que tornaria o projeto um fracasso)

- Porta que só abre pelo app: a geladeira não pode ficar trancada com a bateria morta ou com o ESP32 travado.
- Trava que gasta energia para se manter travada (por exemplo, um solenoide energizado o tempo todo).
- Bateria que acaba sem aviso prévio.
- Qualquer pessoa com o app instalado conseguir abrir sem autorização.
- Instalação que estrague a borracha de vedação e cause condensação ou perda de frio.
- Comprar o kit inteiro de uma vez e descobrir no fim que algo não funciona. Cada etapa precisa ser validada na bancada antes da próxima compra.
- Etapa grande demais, que fica meses sem nada funcionando. O ideal é cada etapa terminar com algo que se veja funcionando.

## 5. Arquitetura física (conforme o diagrama)

| Parte | Onde fica | Função |
|---|---|---|
| Caixa de controle | Topo do gabinete | ESP32, bateria, circuito de acionamento |
| Trava eletrônica | Gabinete, ao lado da porta | Recebe o pino e prende a porta; tem o cilindro da chave na frente |
| Batente com pino | Moldura da porta | Pino que encaixa na trava ao fechar |
| Cabo | Entre a caixa e a trava | Acionamento e sensores |

## 6. Decisões técnicas (*propostas*)

### 6.1 Mecanismo da trava
- **Lingueta de engate automático (fail-secure)**: ao fechar, o pino empurra a lingueta e a porta trava sozinha, sem gastar energia.
- **Destravamento elétrico por pulso**: solenoide ou motor que só consome energia durante a abertura (cerca de 0,5 a 2 s), e não para manter a posição.
- **Chave mecânica** atuando direto na lingueta, em paralelo ao atuador elétrico.
- **Sensores**: um para porta aberta/fechada (reed switch + ímã no batente) e outro para lingueta travada (microchave), para confirmar no app que a porta trancou de verdade.

### 6.2 Comunicação
- **Bluetooth Low Energy (BLE)**: o celular precisa estar perto da geladeira.
- Motivo: com Wi-Fi ligado o tempo todo, a bateria do ESP32 dura dias. Com BLE e anúncios espaçados, dura meses.
- Alternativa, caso seja preciso abrir de longe: um gateway na loja (ESP32 na tomada ou o próprio celular) que repassa os comandos. Isso fica fora do escopo da v1.

### 6.3 Energia
- ESP32 em **deep sleep / light sleep com BLE**, acordando por anúncio BLE ou por interrupção do sensor da porta.
- Bateria de lítio (por exemplo, 18650 ou pack 2S), com regulador de baixo consumo em repouso.
- O pulso de abertura vem de um capacitor ou direto da bateria, com MOSFET. O regulador do ESP32 fica isolado do pico de corrente.
- Medição de tensão da bateria, com o nível enviado ao app.
- Orçamento de energia: calcular na fase de plano (consumo em repouso + aberturas por dia × energia por abertura).

### 6.4 Segurança do acesso
- Pareamento inicial feito pelo dono, que fica como administrador.
- Comandos de abertura autenticados (desafio/resposta), para impedir que um comando gravado seja reenviado depois.
- **Papéis:** `dono` (abre, gerencia acessos, vê histórico, configura) e `cliente` (só abre, dentro do que foi permitido).
- **Permissão que a trava confere sozinha, sem internet** (*proposta*): o servidor do dono emite para o app do cliente um **token assinado** (quem é, qual geladeira, validade). A trava guarda só a chave pública e confere a assinatura via BLE. Assim a trava continua sem Wi-Fi e com a bateria preservada.
- **Revogar um cliente:** tokens de validade curta (por exemplo, horas), renovados pelo app quando ele tem internet. Revogar um cliente é simplesmente parar de emitir tokens para ele.
- Fase 1: o dono pode usar o mesmo mecanismo com um token de longa duração. Isso evita ter dois protocolos.
- Registro local das aberturas (quem, quando, se pelo app ou pela chave). O histórico é enviado pelo app na próxima conexão, por qualquer celular que se conectar, e chega ao servidor.

### 6.6 Servidor (fase 2)
- Necessário para cadastrar clientes, emitir tokens e centralizar o histórico de várias geladeiras.
- Na fase 1 pode ser mínimo, ou o próprio app do dono pode assinar os tokens. A decisão fica para o plano.

### 6.5 App
- Plataforma e tecnologia: **em aberto**.
- Funções mínimas: parear, destravar, ver estado (porta aberta/fechada, travada), ver nível da bateria, ver histórico de aberturas, gerenciar quem pode abrir.

## 7. Cenários que o projeto precisa cobrir

| Cenário | Comportamento esperado |
|---|---|
| Bateria acaba | A porta continua travada e abre pela chave. O app já tinha avisado da bateria baixa antes. |
| Porta fechada sem engatar | O app mostra "fechada, não travada". |
| Porta aberta por muito tempo | Alerta (local e/ou no app). Tempo a definir. |
| Abertura pela chave | Registrada no histórico como "manual". |
| Celular sem permissão tenta abrir | Comando recusado. |
| ESP32 trava ou reinicia | A trava continua fechada (fail-secure) e o watchdog reinicia o ESP32. |
| Condensação ou frio na trava | Componentes e caixa adequados para ambiente úmido e frio. |

## 8. Restrições

- Bateria como única fonte de energia.
- Montagem externa, sem furar a câmara refrigerada.
- Projeto do zero: hardware, firmware e app próprios.

## 9. Abordagens consideradas

| Abordagem | Prós | Contras | Situação |
|---|---|---|---|
| A. BLE + trava com engate automático e pulso | Bateria dura meses; simples | Precisa estar perto da geladeira | **Proposta** |
| B. Wi-Fi direto no ESP32 | Abre de qualquer lugar | Bateria dura dias; inviável sem tomada | Descartada (contra a restrição de bateria) |
| C. BLE + gateway na loja | Abre de longe e mantém a bateria | Mais uma peça, que precisa de tomada | Possível evolução da v1 |

## 10. Perguntas em aberto

1. Na fase 2, como o cliente ganha permissão para abrir? Cadastro simples, pagamento antes de abrir, cobrança depois?
2. Precisa abrir de longe, ou só com o celular perto da geladeira? (No autoatendimento o cliente está na frente da geladeira, então BLE atende.)
3. Quantas aberturas por dia, em média?
4. Bateria recarregável (com entrada USB) ou de troca?
5. Plataforma do app: Android, iOS ou ambos?
6. Modelo e dimensões da geladeira-alvo (espessura da moldura e espaço no topo).
7. ~~Quantidade~~ → começa com **um protótipo**, construído por partes.
8. Que ferramentas você já tem (ferro de solda, multímetro, protoboard)? Isso muda o custo da etapa 0.

## 11. Deixado para quem for construir

- Escolha da peça do atuador e da lingueta, a partir das medidas da geladeira.
- Esquemático, PCB e caixa (impressa em 3D ou comprada).
- Orçamento de energia detalhado e escolha da bateria.
- Protocolo BLE (serviços e características) e formato do histórico.

## 12. Construção por partes (*proposta*)

Cada etapa termina com algo funcionando e só exige comprar o que ela usa. Os valores são **estimativas aproximadas** e precisam ser conferidas no mercado.

| # | Etapa | O que se aprende | Compra nova (aprox.) | Pronto quando… |
|---|---|---|---|---|
| 0 | **Primeiros passos com o ESP32** | Programar o ESP32, acender um LED, BLE básico | ESP32 DevKit, cabo USB, protoboard, jumpers (~R$ 50–80) | O celular, usando um app genérico de BLE (por exemplo, nRF Connect), liga e desliga um LED |
| 1 | **Trava na bancada** | Acionar carga com MOSFET, pulso de abertura | Fechadura solenoide 12 V tipo lingueta, MOSFET/módulo, fonte 12 V (~R$ 60–100) | O comando BLE do app genérico destrava por cerca de 1 s e a lingueta volta sozinha |
| 2 | **Sensores** | Ler entradas e interrupções | Reed switch + ímã, microchave (~R$ 15–30) | O app genérico mostra "porta aberta/fechada" e "travada" |
| 3 | **Bateria e economia de energia** | Sono profundo, medir consumo | Bateria(s) 18650, carregador/proteção, conversor para 12 V (~R$ 50–90) | Funciona só na bateria, com consumo medido e estimativa de duração calculada |
| 4 | **Instalação na geladeira + chave** | Montagem mecânica, caixa | Caixa plástica, suportes, solução da chave mecânica (~R$ 40–100) | Instalada, abre pelo app genérico e pela chave, mesmo sem bateria |
| 5 | **App do dono** | Desenvolvimento de app com BLE | — (software) | O app próprio substitui o genérico: abrir, estado, bateria |
| 6 | **Autorização segura** | Assinatura digital, tokens | — (software) | Só o celular autorizado abre; um comando gravado e reenviado é recusado |
| 7 | **Fase 2: clientes** | Servidor, cadastro, histórico | Hospedagem (pode começar no plano gratuito) | Um cliente cadastrado abre pelo próprio celular; o dono vê no histórico e consegue revogar |

Observações:
- Até a etapa 4 **não é preciso fazer app próprio**: o app genérico de BLE serve para testar. Isso adia a parte de software e mantém o foco no hardware.
- A **fonte 12 V da etapa 1** é só para a bancada. A bateria entra na etapa 3, depois que o resto já funciona.
- Mantém-se a protoboard até a etapa 4. Placa própria (PCB) só se o protótipo for aprovado.
