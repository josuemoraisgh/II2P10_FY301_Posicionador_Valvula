# Instrumentação Industrial II
## Aula Prática 10: Posicionador de Válvula FY301 — Configuração, Auto Setup e Calibração

Esta prática apresenta a base teórica e a execução, no simulador **LasecSimul**, do comissionamento de uma **válvula de controle** com atuador pneumático de mola e diafragma, comandada pelo posicionador inteligente **Smar FY301 (HART)**.

O material segue a ordem de trabalho de uma planta real, com foco em:

- válvula de controle: corpo, característica inerente, Kv, atuador, ar para abrir ou para fechar e posição de falha;
- faixa da mola (**bench set**), atrito, banda morta e histerese;
- o posicionador: por que usar, princípio do FY301 (piezo, carretel e sensor Hall) e o servo PI;
- o laço de 4–20 mA de um **atuador**: o FY301 **lê** o sinal que o controlador **escreve**, é alimentado pelo laço e responde HART por tensão;
- configuração: TYPE, AIR_T, CHAR, MODE, TIME, KP, TR, limites, split range e TSO;
- **Auto Setup**, ajuste de posição inferior e superior (LOPOS/UPPOS) e trim da corrente de entrada;
- teste de curso em cinco pontos, banda morta, resposta ao degrau, posição de falha e registro de calibração.

---

## Índice

1. [Introdução](#1-introdução)
2. [A válvula de controle](#2-a-válvula-de-controle)
3. [O posicionador FY301](#3-o-posicionador-fy301)
4. [O FY301 no laço de 4–20 mA](#4-o-fy301-no-laço-de-420-ma)
5. [A planta no LasecSimul](#5-a-planta-no-lasecsimul)
6. [Arquivos da prática](#6-arquivos-da-prática)
7. [Como executar](#7-como-executar)
8. [Conexão com o PACTware](#8-conexão-com-o-pactware)
9. [Configuração do FY301](#9-configuração-do-fy301)
10. [Calibração](#10-calibração)
11. [Bancada virtual no LasecSimul](#11-bancada-virtual-no-lasecsimul)
12. [Sugestão de atividade experimental](#12-sugestão-de-atividade-experimental)
13. [Conclusão](#13-conclusão)
14. [Referências](#14-referências)
15. [Apêndice: lista de imagens](#15-apêndice-lista-de-imagens)

---

## 1. Introdução

A **válvula de controle** é o elemento final de quase todas as malhas de processo: é ela que muda a vazão de vapor, de água, de gás ou de produto para que a variável controlada siga o setpoint. Uma malha pode ter o melhor transmissor e o controlador mais bem sintonizado; se a válvula não for exatamente para onde o controlador mandou, a malha oscila ou fica lenta.

O **posicionador** é o instrumento montado na válvula que garante essa posição. Ele recebe o sinal de 4–20 mA do controlador, mede a posição da haste e ajusta a pressão de ar no atuador até a válvula chegar à posição pedida, vencendo atrito, variação de pressão do processo e desgaste.

Nesta prática, o posicionador é o **FY301** da Smar, com comunicação **HART**.

**Figura 1. Válvula de controle com o posicionador FY301**

![Figura 1](assets/fig01_valvula_fy301.jpg)

> **Imagem a inserir:** foto de uma válvula de controle globo com atuador pneumático de diafragma e o FY301 montado no castelo (suporte). Indique com setas:
> - o corpo da válvula e o sentido da vazão (seta fundida no corpo);
> - o atuador (caixa do diafragma) e a haste;
> - o FY301, o ímã preso à haste e o indicador de curso (régua);
> - o conjunto filtro-regulador de ar e os tubos de suprimento e de saída (OUT1).

---

## 2. A válvula de controle

### 2.1 Corpo e característica inerente

Na válvula globo, um **obturador** se afasta da **sede** e abre a passagem. A relação entre o curso `l` (0 a 1) e a vazão relativa, com pressão diferencial constante, é a **característica inerente**:

| Característica | Equação | Uso típico |
|---|---|---|
| Linear | `f(l) = l` | nível, vazão com perda de carga pequena na linha |
| Igual porcentagem | `f(l) = (R^l − 1)/(R − 1)` (R = rangeabilidade, ex.: 50) | maioria das malhas de vazão e pressão; compensa a perda de carga da linha |
| Abertura rápida | `f(l) = √l` (aproximação) | liga-desliga, alívio |

A vazão de líquido é calculada pelo **coeficiente de vazão Kv** (m³/h de água com 1 bar de queda):

$$
Q = K_{vs}\, f(l)\,\sqrt{\frac{\Delta P}{G}}
$$

- `Kvs`: Kv com a válvula toda aberta;
- `ΔP`: pressão diferencial na válvula (bar);
- `G`: densidade relativa do líquido (água = 1).

Na instalação, a pressão diferencial muda com a vazão. A curva real (vazão × curso) é a **característica instalada**, e é ela que a malha "enxerga".

### 2.2 Atuador de mola e diafragma

A pressão de ar `P` no diafragma (área `A`) faz força contra a mola. No equilíbrio, o curso é proporcional à pressão, dentro da **faixa da mola**:

$$
l = \frac{P - P_{início}}{P_{fim} - P_{início}}
$$

- **Ar para abrir** (mola fecha): sem ar, a válvula **fecha**. É **falha fechada**.
- **Ar para fechar** (mola abre): sem ar, a válvula **abre**. É **falha aberta**.

A posição de falha é escolhida pela segurança do processo: a válvula de combustível de um queimador é falha fechada; a de água de resfriamento de um reator costuma ser falha aberta.

### 2.3 Faixa da mola (bench set)

O **bench set** é a faixa de pressão, medida na bancada, sem pressão de processo, em que a haste percorre o curso todo. Por exemplo, numa válvula ar para abrir com bench set de **0,2 a 1,0 bar**:

| Pressão no atuador (bar) | 0,2 | 0,4 | 0,6 | 0,8 | 1,0 |
|---|---:|---:|---:|---:|---:|
| Curso (%) | 0 | 25 | 50 | 75 | 100 |

O bench set é ajustado pela pré-carga da mola. Errado, a válvula não fecha estanque ou não abre totalmente.

### 2.4 Atrito, banda morta e histerese

A gaxeta (vedação da haste) gera **atrito**. A haste só se move quando a força supera o atrito. Sem posicionador, isso aparece direto no curso:

- **banda morta:** a menor variação do sinal que faz a haste se mover;
- **histerese:** subindo e descendo, a mesma pressão dá cursos diferentes;
- **agarra-solta (stick-slip):** a haste fica parada e depois salta.

Uma válvula com banda morta grande faz a malha oscilar em torno do setpoint (ciclo-limite). O posicionador mede a posição e ajusta a pressão até vencer o atrito: o curso fica certo, e o atrito passa a aparecer como **pressão a mais** no atuador para a mesma posição. Medida com o laço aberto (rampa de pressão, subindo e descendo), a diferença de pressão entre os dois sentidos é a base da **assinatura da válvula**.

**Figura 2. Características inerentes**

![Figura 2](assets/fig02_caracteristicas_inerentes.png)

> **Imagem a inserir:** gráfico com o curso (%, eixo horizontal) e a vazão relativa (%, eixo vertical) das características linear, igual porcentagem (R = 50) e abertura rápida, pelas equações da seção 2.1. Destaque o ponto de 50 % de curso em cada curva (50 %, 12,4 % e 70,7 %).

**Figura 3. Atuador de mola e diafragma**

![Figura 3](assets/fig03_atuador_mola_diafragma.png)

> **Imagem a inserir:** corte de um atuador pneumático de diafragma em duas versões, lado a lado:
> - **ar para abrir:** mola empurra a haste para fechar, ar entra embaixo do diafragma;
> - **ar para fechar:** mola empurra a haste para abrir, ar entra em cima.
>
> Marque a câmara de ar, o diafragma, a mola, a haste e a seta da força de cada um. Escreva embaixo "falha fechada" e "falha aberta".

---

## 3. O posicionador FY301

### 3.1 Por que usar um posicionador

- vence o atrito da gaxeta e a força do processo no obturador;
- deixa a válvula rápida: o carretel do posicionador enche e esvazia o atuador com vazão de ar muito maior que a de um conversor I/P;
- permite **caracterizar** a válvula (mudar a curva), fazer **split range** (duas válvulas no mesmo sinal), limitar o curso e garantir o fechamento estanque;
- informa a posição real e diagnósticos por HART.

### 3.2 Princípio do FY301

Pelo manual (seção 2):

- um **disco piezoelétrico** faz o papel da palheta de um bico-palheta: a tensão aplicada muda a pressão piloto;
- a pressão piloto age num diafragma que move a **válvula carretel**; o carretel liga a saída (OUT1) ao suprimento ou à exaustão;
- um **sensor de efeito Hall** mede, sem contato, o campo de um **ímã** preso à haste: é a posição real;
- a CPU compara o setpoint de posição com a posição medida e comanda o piezo com um controle **PI** (ganho KP e tempo integral TR).

### 3.3 Cadeia de sinal

```text
4-20 mA → split range → Auto/Man → curva (CHAR) → limites e TSO → TIME → setpoint de posição
ímã + sensor Hall → calibração (Auto Setup, LOPOS/UPPOS) → ação (TYPE) → posição
erro → PI (KP, TR) → piezo → carretel → OUT1 → atuador → haste
```

### 3.4 Especificações principais (manual, seção 6)

| Item | Valor |
|---|---|
| Sinal de entrada | 4–20 mA a dois fios, alimentado pelo laço |
| Impedância de entrada | cerca de 550 Ω |
| Corrente mínima | 3,8 mA |
| Suprimento de ar | 1,4 a 7 bar (20 a 100 psi), sem óleo, sujeira e água |
| Saída | 0 a 100 % do suprimento, ação simples ou dupla |
| Curso | linear 3 a 100 mm; rotativo 30 a 120° |
| Resolução, repetibilidade, histerese | 0,1 % do fundo de escala |
| Característica | linear, igual porcentagem, abertura rápida e tabela de 16 pontos |

**Figura 4. Esquema do transdutor pneumático do FY301**

![Figura 4](assets/fig04_transdutor_piezo_carretel.png)

> **Imagem a inserir:** redesenho da Figura 2.1 do manual (esquema do transdutor): suprimento, restrição, bico, disco piezoelétrico, câmara piloto, diafragmas do servo, carretel e as saídas OUT1 e OUT2. Indique o caminho do ar quando a válvula abre e quando fecha.

**Figura 5. Diagrama de blocos do FY301**

![Figura 5](assets/fig05_diagrama_blocos.png)

> **Imagem a inserir:** diagrama de blocos simplificado (baseado na Figura 2.2 do manual): entrada 4–20 mA com o A/D, CPU, modem HART, display, controle, piezo, carretel, sensor de posição Hall e a realimentação de posição. Destaque o laço interno de posição (servo PI).

---

## 4. O FY301 no laço de 4–20 mA

### 4.1 Quem escreve e quem lê o sinal

- O LD301 e o TT301 (aulas anteriores) **escrevem** o sinal: são transmissores, controlam a corrente do laço com a energia de uma fonte de 24 V.
- O FY301 **lê** o sinal: quem escreve a corrente é a **saída analógica do controlador** (CLP, SDCD ou controlador single-loop) ou, na bancada, um **calibrador em modo fonte de mA**.

O FY301 é alimentado pelo próprio sinal e se comporta como uma carga de cerca de **550 Ω**. A saída analógica tem de suportar a queda de tensão:

$$
V = R\,I = 550 \times 0{,}020 = 11\ \text{V}
$$

| Corrente (mA) | 4 | 8 | 12 | 16 | 20 |
|---|---:|---:|---:|---:|---:|
| Tensão nos bornes do FY301 (V) | 2,2 | 4,4 | 6,6 | 8,8 | 11,0 |

Em split range, dois posicionadores no mesmo sinal ficam em série: 1100 Ω e 22 V a 20 mA. Abaixo de **3,8 mA** o FY301 não funciona: o display apaga, a saída de ar esvazia e a válvula vai à posição de falha.

### 4.2 HART num atuador

O protocolo HART é o mesmo dos transmissores: os mesmos quadros, as mesmas frequências (1200 Hz e 2200 Hz), o mesmo modem e o mesmo PACTware. Muda só a forma elétrica da resposta:

- o transmissor controla a corrente, então responde variando a **corrente** (±0,5 mA);
- o posicionador não controla a corrente (quem controla é a saída analógica), então responde variando a **tensão** nos bornes (cerca de 0,5 Vpp).

O configurador mede tensão entre as garras, por isso ouve os dois. No laço de um posicionador **não há resistor de 250 Ω**: o configurador é ligado **em paralelo**, nos terminais COMM e − do FY301 ou em qualquer ponto da linha.

### 4.3 Bornes e ar

- **+ e −:** sinal de 4–20 mA (o FY301 é protegido contra polaridade invertida);
- **TEST:** para medir a corrente com o multímetro sem abrir o laço;
- **COMM:** para o configurador HART;
- **terra:** um interno e dois externos;
- **SUP:** suprimento de ar do filtro-regulador;
- **OUT1 / OUT2:** saídas para o atuador (OUT2 só em atuador de dupla ação).

**Figura 6. Ligação do FY301 no laço**

![Figura 6](assets/fig06_ligacao_fy301.png)

> **Imagem a inserir:** desenho baseado na Figura 1.10 do manual. Devem aparecer:
> - a saída analógica do controlador escrevendo 4–20 mA nos bornes + e − do FY301;
> - o multímetro em série (ou nos terminais TEST);
> - o configurador HART ligado em paralelo nos terminais COMM e −.
>
> Ao lado, a ligação de um transmissor (aula 06) com fonte de 24 V e resistor de 250 Ω, para comparar quem escreve e quem lê o sinal.

**Figura 7. Bornes do FY301**

![Figura 7](assets/fig07_bornes_fy301.jpg)

> **Imagem a inserir:** foto do FY301 com a tampa dos terminais aberta, com setas para + e −, TEST, COMM e os terminais de terra. Em outra foto, as conexões pneumáticas SUP, OUT1, OUT2 e os manômetros.

---

## 5. A planta no LasecSimul

O projeto [`posicionador_fy301.lsproj`](posicionador_fy301.lsproj) reproduz uma válvula globo com atuador de mola e diafragma, comandada pelo FY301:

- o slider **"Saída do controlador (mA)"** faz o papel da saída analógica do controlador (ou do calibrador em modo fonte), de 4 a 20 mA;
- um amperímetro mostra a corrente escrita;
- o **mesmo modem HART** das outras aulas está ligado **em paralelo** na linha (propriedade "Ligação na linha = Em paralelo");
- a válvula vem **incorporada ao projeto**, ligada ao FY301 por três sinais: o suprimento de ar (SUP), a pressão de saída (OUT1) e a posição da haste lida pelo ímã (POS).

É preciso o LasecSimul **0.0.85 ou mais novo**: o FY301 como posicionador alimentado pelo laço foi introduzido nessa versão.

**Figura 8. Projeto completo no LasecSimul**

![Figura 8](assets/fig08_projeto_completo.png)

> **Imagem a inserir:** captura do LasecSimul com `posicionador_fy301.lsproj` rodando (selo "Running"), depois do Auto Setup, com o controlador em 12 mA. Devem aparecer:
> - a válvula, com a pressão no atuador, o curso e a vazão;
> - o FY301 com o display em 50 %;
> - o amperímetro em 12,00 mA, a saída analógica do controlador, o slider e o modem HART em paralelo.
>
> Use o botão "ajustar tudo à tela".

### 5.1 O que a planta calcula

- **Pressão no atuador:** `dP/dt = (OUT1 − P)/τ` (volume do atuador e tubo).
- **Mola e diafragma:** curso pela faixa da mola, ar para abrir ou ar para fechar.
- **Haste:** banda morta do atrito da gaxeta e pequeno atraso mecânico.
- **Vazão:** `Q = Kvs·f(l)·√ΔP`, com a característica inerente escolhida.

### 5.2 O que o FY301 calcula

- lê a corrente nos bornes (carga de 550 Ω) e calcula o setpoint;
- aplica split range, modo, curva, limites, TSO e TIME;
- mede a posição pelo ímã, com a calibração do Auto Setup e os ajustes LOPOS/UPPOS;
- comanda OUT1 com o servo PI (KP e TR);
- sem sinal (abaixo de 3,8 mA) desliga: OUT1 esvazia.

### 5.3 Parâmetros da válvula (Propriedades exportadas)

| Parâmetro | Padrão |
|---|---|
| Pressão de suprimento de ar | 1,4 bar |
| Ação do atuador | Ar para abrir (falha fechada) |
| Faixa da mola (bench set) | 0,2 a 1,0 bar |
| Curso da válvula | 20 mm |
| Constante de tempo pneumática do atuador | 0,5 s |
| Atrito da gaxeta (banda morta) | 0,5 % do curso |
| Kvs | 10 m³/h |
| Característica inerente | Linear |
| Pressão diferencial na válvula | 1 bar |

### 5.4 Parâmetros do FY301 (Propriedades exportadas = ajuste local)

No aparelho real, estes parâmetros são ajustados pelo PACTware ou pela chave magnética (ajuste local, seção 4 do manual). No simulador, ficam nas **Propriedades exportadas** do FY301:

| Parâmetro | Padrão | Significado |
|---|---|---|
| TYPE | Lind | tipo de válvula e ação: Lind, Linr, Rotd, Rotr |
| AIR_T | AIR_OPEN | o ar aumenta (AIR_OPEN) ou diminui (AIR_CLOSED) a posição |
| CHAR | Lin | curva: linear, igual porcentagem 1:25/1:33/1:50, abertura rápida 1:25/1:33/1:50 |
| MODE e SP% | Auto | Auto segue os 4–20 mA; Man segue o SP% |
| TIME | 1 s | tempo do setpoint para ir de 0 a 100 % |
| KP e TR | 40 e 2 min/rep | ganho e tempo integral do servo |
| LOPOS e UPPOS | 0 e 100 % | ajuste fino das posições inferior e superior |
| TSO | 0 (desligado) | abaixo desse setpoint a válvula fecha estanque |
| Limites do setpoint | 0 e 100 % | curso mínimo e máximo |
| Split range | 4 e 20 mA | correntes do 0 % e do 100 % |
| SETUP | Parado | Executar inicia o Auto Setup |

### 5.5 Valores esperados

Com o Auto Setup feito, característica linear e ação direta:

| Corrente (mA) | 4 | 8 | 12 | 16 | 20 |
|---|---:|---:|---:|---:|---:|
| Curso (%) | 0 | 25 | 50 | 75 | 100 |
| Curso (mm) | 0 | 5 | 10 | 15 | 20 |
| Pressão no atuador (bar) | ≤ 0,2 | 0,4 | 0,6 | 0,8 | ≥ 1,0 |
| Vazão, válvula linear (m³/h) | 0 | 2,5 | 5,0 | 7,5 | 10,0 |
| Vazão, válvula igual % (m³/h) | 0 | 0,34 | 1,24 | 3,63 | 10,0 |

**Figura 9. Detalhe da planta**

![Figura 9](assets/fig09_planta_valvula.png)

> **Imagem a inserir:** captura ampliada só da válvula, com o controlador em 12 mA depois do Auto Setup. Devem aparecer:
> - o atuador, a mola, a haste com o ímã e o indicador de curso em 50 %;
> - o manômetro PI-101 em 0,6 bar e o filtro-regulador;
> - os displays de OUT1, pressão no atuador, curso (10 mm), abertura (50 %) e vazão (5 m³/h).

**Figura 10. Propriedades exportadas do FY301**

![Figura 10](assets/fig10_propriedades_fy301.png)

> **Imagem a inserir:** painel **Properties** com o FY301 selecionado, rolado até "Propriedades exportadas", mostrando TYPE, AIR_T, CHAR, MODE, SP%, TIME, KP, TR, LOPOS, UPPOS, TSO, os limites, o split range e SETUP, com a lista de CHAR aberta.

---

## 6. Arquivos da prática

- [`posicionador_fy301.lsproj`](posicionador_fy301.lsproj): projeto do LasecSimul com a válvula incorporada, o FY301, a saída analógica do controlador e o modem HART em paralelo;
- [`assets/`](assets): imagens deste roteiro.

---

## 7. Como executar

1. Instale ou atualize a extensão **LasecSimul** no VS Code (0.0.85 ou mais nova).
2. Abra esta pasta no VS Code e dê dois cliques em `posicionador_fy301.lsproj`.
3. Clique em **Run** (▶).
4. Selecione o slider **"Saída do controlador (mA)"** e mude o **Valor do sinal** no painel Properties.
5. Para os ajustes do posicionador, selecione o FY301 e use **Propriedades exportadas**.
6. Para mudar a válvula (suprimento, ação, faixa da mola, atrito, curva), selecione a válvula e use **Propriedades exportadas**.

Observações importantes:

- o FY301 sai de fábrica **sem Auto Setup**: ele mede a posição pela escala de fábrica do sensor (30 mm). Num curso de 20 mm, o display mostra 50 % com a válvula 75 % aberta. Faça o Auto Setup (seção 9.2) antes de qualquer teste;
- o Auto Setup leva cerca de 50 s no simulador (cerca de 4 minutos no aparelho real);
- não mexa na corrente durante o Auto Setup.

---

## 8. Conexão com o PACTware

1. Selecione o **Modem HART** e confira em Properties: **Ligação com o PC** = Serial (COM / CNC), **Porta serial** = `CNCB0` e **Ligação na linha** = **Em paralelo na linha**.
2. No **com0com**, confira o par `CNCB0` ↔ `COM20`.
3. No PACTware, em **Ver → Catálogo de instrumentos**, atualize o catálogo e adicione **HART Communication**.
4. Configure o driver HART na porta **COM20** e desmarque **Multimaster and Burst mode support**.
5. Clique com o botão direito no driver → **Adicionar instrumento** → **FY301** (Smar).
6. Com a simulação rodando e pelo menos 4 mA no laço, conecte o FY301 e abra **OnLine Parameterize**.

No simulador, o FY301 responde aos comandos HART Universais e de Prática Comum:

- identificação (Tag, Descriptor, Message, Date);
- a corrente lida e a porcentagem (Comando 2);
- as variáveis do dispositivo (Comando 33), entre elas a posição.

As telas de configuração própria da Smar (TYPE, CHAR, Auto Setup, trims) dependem dos comandos do fabricante. Esses comandos ainda serão capturados no FY301 do laboratório. Até lá, no simulador, faça esses ajustes pelas Propriedades exportadas (seção 5.4). No aparelho real, siga as mesmas telas descritas na seção 9.

**Figura 11. FY301 conectado no PACTware**

![Figura 11](assets/fig11_pactware_conectado.png)

> **Imagem a inserir:** tela do PACTware com:
> - a árvore do projeto (HART Communication em COM20 → FY301) com ícones verdes;
> - a tela **Identification** aberta: Manufacturer SMAR, Device Type FY301, Tag, Descriptor e Message.
>
> Se possível, tire do FY301 real do laboratório.

---

## 9. Configuração do FY301

Siga o **Procedimento para Calibração da Válvula** do manual (seção 4). Faça uma alteração por vez.

### 9.1 Tipo de válvula e efeito do ar

1. **TYPE**: escolha **Lind** (linear, ação direta): 4 mA = 0 %, 20 mA = 100 %. Linr inverte (4 mA = 100 %). Rotd e Rotr são para válvulas rotativas (esfera, borboleta).
2. **AIR_T**: pela regra do manual:
   - **ação direta:** AIR_OPEN se o ar abre a válvula, AIR_CLOSED se o ar fecha;
   - **ação reversa:** AIR_OPEN se o ar fecha a válvula, AIR_CLOSED se o ar abre.

Na planta deste projeto (ar para abrir, ação direta), o certo é **AIR_OPEN**. Com o efeito do ar trocado, o servo empurra para o lado errado (realimentação positiva) e a válvula vai até o fim do curso.

### 9.2 Auto Setup

Com o tipo configurado, execute o **SETUP**. O FY301 abre e fecha a válvula, descobre como o ímã foi montado, grava as posições de 0 e de 100 % e ajusta as referências internas. O manual descreve os passos de 10 % em 10 %; as mensagens possíveis são:

| Mensagem | Causa |
|---|---|
| HALL | cabo flat desligado ou sensor de posição com defeito |
| FAIL MOVE | carretel travado ou falta de ar |
| MGNT | ímã desacoplado da haste ou montado errado |

No simulador: **Propriedades exportadas → SETUP = Executar**. Espere cerca de 50 s: o FY301 esvazia o atuador (válvula fechada), depois manda ar total (válvula aberta) e grava as duas posições.

### 9.3 Ganho, tempo integral e TIME

- **KP:** maior KP, mais rápido e com mais sobressinal. Em válvulas lineares, 35 a 45; em rotativas, cerca de 8.
- **TR:** tempo integral (minutos por repetição). Linear: cerca de 2; rotativa: cerca de 8. A integral remove o pequeno desvio que o proporcional deixa.
- **TIME:** tempo do setpoint para ir de 0 a 100 % (1 a 60 s). Aumente para reduzir o sobressinal em válvulas rápidas.

### 9.4 Curva, modo, limites, split range e TSO

- **CHAR:** a curva do posicionador muda a característica efetiva. Uma válvula linear com CHAR = EP 1:50 se comporta como igual porcentagem: com 12 mA (50 %) o curso vai a **12,4 %**.
- **MODE:** em **Man**, a válvula segue o **SP%** e ignora a corrente; use para testes com o processo parado.
- **Limites do setpoint:** impedem que a válvula feche ou abra além de um valor.
- **Split range:** com 4–12 mA → 0–100 %, a válvula percorre o curso todo na metade inferior do sinal; outra válvula usa 12–20 mA.
- **TSO (Tight Shut Off):** abaixo desse setpoint o FY301 fecha a válvula com força total (estanqueidade na sede).

### 9.5 Identificação

Por HART (PACTware, tela Identification): **Tag** `FV-101`, **Descriptor** `VALV VAPOR` e **Message** com a data e o responsável.

**Figura 12. Telas de configuração do FY301 no PACTware**

![Figura 12](assets/fig12_dtm_configuracao.png)

> **Imagem a inserir:** montagem com as telas do DTM do FY301 (do aparelho real), cada uma antes de aplicar:
> - Configuração do Equipamento: tipo de válvula, curva de caracterização e display;
> - Configuração Avançada: ar para abrir/fechar, ação, limites de setpoint e split range;
> - Setup: Auto Setup, Posição Inferior/Superior, KP, TR e TIME.

**Figura 13. Display do FY301**

![Figura 13](assets/fig13_lcd_fy301.png)

> **Imagem a inserir:** duas capturas ampliadas do display do FY301 no simulador com o controlador em 12 mA, montadas lado a lado: a posição (50,0 %) e o setpoint de posição, que se alternam no display.

---

## 10. Calibração

### 10.1 O que se calibra

Num conjunto de válvula de controle há três partes a verificar:

- a **válvula e o atuador**: bench set, curso total, atrito e fechamento;
- o **posicionador**: a leitura da corrente de entrada e a relação corrente → posição;
- o **conjunto**: banda morta, resolução, velocidade, posição de falha.

Diferente de um transmissor, o "padrão" aqui é a **posição real da haste**, medida na régua do indicador de curso (ou com um relógio comparador) e a **corrente** aplicada por um calibrador.

### 10.2 Teste como encontrado

1. Salve a configuração do FY301 no PACTware.
2. Anote TYPE, AIR_T, CHAR, limites, split range, KP, TR e TIME.
3. Faça o teste de curso (seção 10.4) **sem** mudar nada.

### 10.3 Bench set do atuador

Feito com o posicionador desligado do atuador (ou na bancada), com um regulador de precisão e um manômetro:

1. Aplique pressão no atuador, subindo devagar, e anote a pressão em que a haste **começa** a se mover.
2. Continue até o fim do curso e anote a pressão.
3. Compare com a faixa da plaqueta (por exemplo, 0,2 a 1,0 bar).

| Curso (%) | 0 (início) | 25 | 50 | 75 | 100 (fim) |
|---|---:|---:|---:|---:|---:|
| Pressão esperada (bar) | 0,2 | 0,4 | 0,6 | 0,8 | 1,0 |
| Pressão medida ↑ (bar) | | | | | |
| Pressão medida ↓ (bar) | | | | | |

A diferença entre subida e descida, na mesma posição, mostra o **atrito** da gaxeta.

### 10.4 Teste de curso em cinco pontos

1. Ligue o **calibrador em modo fonte de mA** nos bornes + e − do FY301 (ou use a saída analógica forçada pelo sistema de controle) e o multímetro em série.
2. Aplique 4, 8, 12, 16 e 20 mA, subindo, e depois 20, 16, 12, 8 e 4 mA, descendo. Pare em cada ponto até a válvula estabilizar.
3. Anote o curso na régua e a posição lida pelo FY301 (display ou PACTware).

| Corrente (mA) | Curso esperado (%) | Curso medido ↑ (%) | Posição FY301 ↑ (%) | Curso medido ↓ (%) | Posição FY301 ↓ (%) |
|---:|---:|---:|---:|---:|---:|
| 4 | 0 | | | | |
| 8 | 25 | | | | |
| 12 | 50 | | | | |
| 16 | 75 | | | | |
| 20 | 100 | | | | |

Erro e histerese em % do curso:

$$
e(\%) = l_{medido} - l_{esperado}
\qquad
h(\%) = \left| l_{↑} - l_{↓} \right|
$$

Critério de exemplo: erro de **±1 %** do curso e histerese menor que 1 %.

**Figura 14. Calibrador em modo fonte de mA no FY301**

![Figura 14](assets/fig14_calibrador_ma.jpg)

> **Imagem a inserir:** foto da bancada com:
> - o calibrador de processo em modo **fonte de mA** (por exemplo, 12,00 mA) ligado aos bornes + e − do FY301;
> - o multímetro em série;
> - a régua do indicador de curso da válvula em 50 %.

### 10.5 Ajuste fino da posição (LOPOS e UPPOS)

Se, depois do Auto Setup, a válvula não fecha ou não abre exatamente:

1. Aplique **4 mA** e ajuste **LOPOS** até a válvula ficar na posição inferior desejada (por exemplo, encostando na sede sem esmagar).
2. Aplique **20 mA** e ajuste **UPPOS** até o curso total.
3. Repita os cinco pontos.

O manual recomenda variar o parâmetro aos poucos: se o controle saturar (a válvula não se move mais naquele sentido), volte no sentido contrário.

### 10.6 Trim da corrente de entrada

Se o FY301 lê uma corrente diferente da aplicada (veja o Comando 2 no PACTware ou a corrente na monitoração):

1. Aplique **4,000 mA** com o calibrador e faça o **Trim de 4 mA**: o FY301 passa a ler esse valor como 0 %.
2. Aplique **20,000 mA** e faça o **Trim de 20 mA**.

No posicionador, o trim de corrente ajusta a **leitura** da entrada; no transmissor (aula 07), ajustava a **saída**.

### 10.7 Banda morta e resolução

O teste de cinco pontos não mostra a banda morta. Para medi-la:

1. Com a válvula em 50 %, aplique degraus pequenos de corrente (por exemplo, 0,04 mA = 0,25 %) no mesmo sentido, até a haste se mover.
2. Repita no sentido contrário.

A **banda morta** é a menor variação que faz a válvula andar; a **resolução**, o menor movimento que ela faz. Válvulas com banda morta grande fazem a malha oscilar.

### 10.8 Resposta ao degrau

1. Aplique degraus de 10 % (por exemplo, de 40 a 50 % e de volta) e meça o tempo até a haste chegar a 63 % e a 86 % do degrau, e o sobressinal.
2. Ajuste KP, TR e TIME: menos sobressinal com KP menor ou TIME maior; mais rápido com KP maior.

O teste de resposta ao degrau é padronizado pela ISA-75.25.01.

### 10.9 Posição de falha e fechamento estanque

1. **Falta de sinal:** desligue a corrente (ou abaixe para menos de 3,8 mA). O FY301 apaga, OUT1 esvazia e a válvula vai para a posição de falha (fechada, se for ar para abrir).
2. **Falta de ar:** feche o suprimento. A válvula vai para a mesma posição de falha.
3. **TSO:** com TSO = 2 %, aplique um sinal de 1 % e confira que a válvula fecha totalmente.

### 10.10 Teste como deixado e registro

Repita o teste da seção 10.4 e preencha o registro com:

- identificação (tag, válvula, atuador, faixa da mola, FY301);
- padrões usados (calibrador, régua ou relógio comparador, manômetro) e validade;
- tabelas como encontrado e como deixado;
- ajustes feitos (Auto Setup, LOPOS/UPPOS, trims, KP/TR/TIME);
- resultado da posição de falha;
- responsável e data.

**Figura 15. Gráfico de curso como encontrado e como deixado**

![Figura 15](assets/fig15_grafico_curso.png)

> **Imagem a inserir:** gráfico com a corrente (4 a 20 mA) no eixo horizontal e o curso (%) no vertical, com a reta ideal, as séries subida e descida como encontrado (mostrando o erro da escala de fábrica, antes do Auto Setup) e como deixado, e a faixa de tolerância de ±1 %.

**Figura 16. Resposta ao degrau**

![Figura 16](assets/fig16_resposta_degrau.png)

> **Imagem a inserir:** gráfico do curso (%) em função do tempo para um degrau de 40 para 50 %, com duas sintonias:
> - KP = 40 e TIME = 1 s, com sobressinal;
> - KP = 40 e TIME = 5 s, sem sobressinal.
>
> Marque o tempo morto, o tempo para 63 % e o sobressinal.

---

## 11. Bancada virtual no LasecSimul

| Situação real | Como reproduzir no LasecSimul |
|---|---|
| Calibrador em modo fonte de mA | slider "Saída do controlador (mA)" |
| Posicionador sem Auto Setup | estado inicial do projeto (escala de fábrica do sensor) |
| Auto Setup | FY301: SETUP = Executar |
| Bench set errado | Válvula: "Faixa da mola" (por exemplo, 0,4 a 1,2 bar) |
| Gaxeta apertada (atrito alto) | Válvula: "Atrito da gaxeta" (por exemplo, 5 %); compare a pressão no atuador na mesma posição |
| Falta de ar | Válvula: "Pressão de suprimento de ar" = 0 |
| Falta de sinal | slider em 4 mA e depois abaixo de 3,8 mA (Propriedades do slider: mínimo 0) |
| Efeito do ar trocado | FY301: AIR_T = AIR_CLOSED |
| Atuador ar para fechar | Válvula: "Ação do atuador" = Ar para fechar, com FY301 AIR_T = AIR_CLOSED |
| Válvula igual porcentagem | Válvula: "Característica inerente" = Igual porcentagem |
| Caracterização pelo posicionador | FY301: CHAR |
| Split range | FY301: correntes do 0 % e do 100 % |

A eletrônica simulada do FY301 é ideal: ela lê exatamente a corrente aplicada. O simulador mostra os efeitos da válvula, do atuador, da configuração e do Auto Setup. O trim da corrente de entrada (seção 10.6) deve ser praticado no FY301 real do laboratório.

**Figura 17. Atrito alto: o posicionador compensa, a pressão revela**

![Figura 17](assets/fig17_atrito_pressao.png)

> **Imagem a inserir:** duas capturas da válvula no LasecSimul em 12 mA (50 %), montadas lado a lado:
> - atrito da gaxeta em 0,5 %, com a pressão no atuador perto de 0,60 bar;
> - atrito em 5 %, perto de 0,62 bar.
>
> Destaque que a abertura é a mesma e a pressão necessária é maior.

**Figura 18. Modem HART em paralelo**

![Figura 18](assets/fig18_modem_paralelo.png)

> **Imagem a inserir:** captura ampliada da região do laço no LasecSimul, com o painel **Properties** do Modem HART aberto mostrando **Ligação na linha = Em paralelo na linha**. Devem aparecer a saída analógica, o amperímetro e os bornes do FY301.

---

## 12. Sugestão de atividade experimental

### Parte A: reconhecimento e sinal de 4–20 mA

1. Abra o projeto, clique em **Run** e anote a corrente, o display do FY301 e o curso real da válvula com 12 mA.
2. Explique por que o FY301 mostra 50 % e a válvula está 75 % aberta.
3. Calcule a tensão nos bornes do FY301 a 20 mA e diga qual deve ser a capacidade da saída analógica do controlador.

### Parte B: Auto Setup

1. Execute o Auto Setup e acompanhe a válvula fechar e abrir.
2. Com 12 mA, confira o curso (50 %, 10 mm) e a pressão no atuador (0,6 bar).

### Parte C: teste de curso e histerese

1. Faça o teste de cinco pontos de ida e volta (seção 10.4) e calcule erro e histerese.
2. Com 12 mA, anote a pressão no atuador. Aumente o atrito da gaxeta para 5 %, espere estabilizar e anote de novo.
3. A posição quase não muda: o posicionador aumenta a pressão até vencer o atrito. A mesma posição de 50 % passa a exigir mais pressão (cerca de 0,60 bar com atrito de 0,5 % e 0,62 bar com 5 %). Explique por que acompanhar a pressão necessária para cada posição revela o desgaste da gaxeta: é a ideia da "assinatura da válvula" que os posicionadores inteligentes registram.

### Parte D: efeito do ar e posição de falha

1. Troque AIR_T para AIR_CLOSED e observe a válvula. Explique.
2. Volte para AIR_OPEN. Leve o sinal para 2 mA e depois corte o suprimento de ar. Para onde vai a válvula em cada caso?
3. Mude a válvula para "Ar para fechar". Qual AIR_T é o certo agora? Qual é a posição de falha?

### Parte E: características e caracterização

1. Com a válvula linear, registre a vazão em 0, 25, 50, 75 e 100 %.
2. Mude a característica inerente para igual porcentagem e repita.
3. Volte para linear e use CHAR = EP 1:50 no FY301. Compare com o item anterior.

| Curso (%) | 0 | 25 | 50 | 75 | 100 |
|---|---:|---:|---:|---:|---:|
| Vazão, válvula linear (m³/h) | | | | | |
| Vazão, válvula igual % (m³/h) | | | | | |
| Vazão, linear + CHAR EP 1:50 (m³/h) | | | | | |

### Parte F: sintonia do servo

1. Aplique degraus de 40 para 50 % com KP = 40, KP = 10 e KP = 1, e meça o tempo de resposta e o erro.
2. Com KP = 40, compare TIME = 1 s e TIME = 5 s.

### Parte G: split range

1. Configure o split range para 4–12 mA.
2. Registre o curso em 4, 8, 12 e 16 mA.
3. Para que serve uma segunda válvula em 12–20 mA? Dê um exemplo de processo.

### Questões para o relatório

1. Por que o FY301 **lê** o sinal de 4–20 mA, enquanto o LD301 e o TT301 o **escrevem**? Qual a consequência para a fonte de 4–20 mA?
2. Por que o configurador HART é ligado em paralelo, e não há resistor de 250 Ω no laço do posicionador?
3. Qual a diferença entre ar para abrir e falha fechada? Como escolher a posição de falha?
4. O que acontece com uma válvula sem posicionador quando o atrito da gaxeta aumenta?
5. Para que serve o Auto Setup? O que o posicionador mede durante ele?
6. Qual a diferença entre a característica inerente da válvula e a caracterização feita pelo posicionador?
7. Por que o teste de cinco pontos não basta para avaliar uma válvula de controle?

Essa atividade conecta a mecânica da válvula, a pneumática do atuador e a configuração e calibração de um posicionador HART.

---

## 13. Conclusão

A prática mostra, de forma aplicada, a cadeia completa do elemento final de controle:

- o controlador escreve o sinal de 4–20 mA, e o FY301 o lê, alimentado pelo próprio laço;
- o FY301 compara o setpoint com a posição medida pelo ímã e comanda a pressão no atuador;
- a mola e o diafragma transformam pressão em curso, e o atrito gera banda morta e histerese;
- o obturador transforma curso em vazão pela característica da válvula;
- o Auto Setup, os ajustes de posição, o trim de entrada e os testes de curso, de banda morta, de resposta e de falha garantem que a válvula vá para onde o controlador mandou.

Com isso, o roteiro serve tanto como material de estudo quanto como apoio para a aula prática de instrumentação.

---

## 14. Referências

- Apostila de Instrumentação Industrial II - UFU
- Smar. *FY301 — Manual de Instruções, Operação e Manutenção* (versão 4.XX):
  - seção 1: ligação elétrica, Figuras 1.10 a 1.12 e suprimento de ar;
  - seção 2: transdutor piezoelétrico, carretel, sensor Hall e display;
  - seção 3: configuração, Trim e Setup;
  - seção 4: ajuste local e procedimento de calibração da válvula;
  - seção 6: especificações.
- IEC 60534-2-1. *Industrial-process control valves — Flow capacity — Sizing equations for fluid flow.*
- IEC 60534-2-4. *Industrial-process control valves — Inherent flow characteristics and rangeability.*
- ISA-75.25.01. *Test Procedure for Control Valve Response Measurement from Step Inputs.*
- Emerson. [Control Valve Handbook (5ª edição)](https://d1-auth.emerson.com/documents/automation/control-valve-handbook-en-3661206.pdf).
- AutomationForum. [What is stroke checking of control valves?](https://automationforum.co/stroke-checking-of-control-valve/)
- AutomationForum. [Why is control valve actuator bench set important?](https://automationforum.co/why-is-control-valve-actuator-bench-set-important/)
- CN Control Valve. [Valve positioner commissioning, calibration and troubleshooting guide](https://cncontrolvalve.com/valve-positioner-commissioning-troubleshooting-guide/).
- CN Control Valve. [Positioner error and deadband in control valve performance](https://cncontrolvalve.com/positioner-error-deadband-control-valve-performance/).
- Emerson Exchange 365. [Use of 250 ohm resistance in HART communication](https://emersonexchange365.com/community-hubs/valves-actuators-regulators/f/valves-discussions-questions/4513/use-of-250ohm-resistance-in-hart-communication).
- Baker Hughes. [Druck DPI620G — HART application note](https://dam.bakerhughes.com/m/37bc03eed810623e/original/Druck-DPI620G-IS-series-Application-Note.pdf).
- [LasecSimul — LASEC/UFU](https://github.com/LASEC-UFU/LasecSimul)

---

## 15. Apêndice: lista de imagens

Todas as imagens ficam na pasta [`assets/`](assets), com os nomes exatos da tabela. Cada figura do texto já traz, no quadro "Imagem a inserir", o que ela precisa mostrar. Este apêndice reúne como produzir cada uma.

### 15.1 Como produzir as imagens

- **Capturas do LasecSimul:** VS Code maximizado (1920 × 1080), barra lateral fechada (`Ctrl+B`), escala do Windows em 100 %, simulação rodando e valores estabilizados. Use **Ajustar tudo** ou **Ajustar seleção** e a Ferramenta de Captura (`Win+Shift+S`). Salve em PNG.
- **Capturas do PACTware:** janela maximizada, FY301 conectado (ícones verdes). Capture antes de aplicar quando a figura mostra valores digitados. As telas de configuração própria do FY301 devem vir do aparelho real.
- **Fotos:** luz difusa, sem flash direto nos displays e manômetros, câmera paralela ao instrumento, régua de curso legível. Salve em JPG.
- **Desenhos e gráficos:** diagrams.net (draw.io), Inkscape, PowerPoint, planilha ou Python, fundo branco, PNG com cerca de 1600 px de largura.
- **Anotações:** setas e números em vermelho, explicados na legenda. Nos desenhos pneumáticos, use azul para o ar.

### 15.2 Figuras

| Figura | Arquivo | Tipo | Como obter | O que destacar |
|---:|---|---|---|---|
| 1 | `fig01_valvula_fy301.jpg` | Foto | Foto da válvula do laboratório com o FY301 montado. | Corpo, sentido da vazão, atuador, haste, ímã, régua de curso e filtro-regulador. |
| 2 | `fig02_caracteristicas_inerentes.png` | Gráfico | Planilha ou Python com as equações da seção 2.1. | As três curvas e os pontos de 50 % de curso. |
| 3 | `fig03_atuador_mola_diafragma.png` | Desenho | Corte de atuador ar para abrir e ar para fechar. | Câmara de ar, mola, forças e a posição de falha. |
| 4 | `fig04_transdutor_piezo_carretel.png` | Desenho | Redesenho da Figura 2.1 do manual. | Piezo, bico, câmara piloto, carretel e OUT1/OUT2. |
| 5 | `fig05_diagrama_blocos.png` | Desenho | Diagrama de blocos a partir da Figura 2.2 do manual. | O laço interno de posição (servo PI) e o sensor Hall. |
| 6 | `fig06_ligacao_fy301.png` | Desenho | Baseado na Figura 1.10 do manual, com o transmissor ao lado. | Quem escreve e quem lê o sinal; o configurador em paralelo. |
| 7 | `fig07_bornes_fy301.jpg` | Foto | Foto do FY301 com a tampa dos terminais aberta e das conexões pneumáticas. | +, −, TEST, COMM, terra, SUP, OUT1, OUT2 e manômetros. |
| 8 | `fig08_projeto_completo.png` | Captura LasecSimul | Abra `posicionador_fy301.lsproj`, **Run**, Auto Setup, controlador em 12 mA e **Ajustar tudo**. | Válvula, FY301 em 50 %, amperímetro, saída analógica, slider e modem. |
| 9 | `fig09_planta_valvula.png` | Captura LasecSimul | Selecione a válvula, **Ajustar seleção** e recolha o painel Properties. | Indicador de curso, PI-101 em 0,6 bar, curso de 10 mm e vazão de 5 m³/h. |
| 10 | `fig10_propriedades_fy301.png` | Captura LasecSimul | Selecione o FY301 e abra **Properties → Propriedades exportadas**. | Os parâmetros do ajuste local, com a lista de CHAR aberta. |
| 11 | `fig11_pactware_conectado.png` | Captura PACTware | Conecte o FY301 e abra **Identification**. | Ícones verdes, SMAR, FY301, Tag, Descriptor e Message. |
| 12 | `fig12_dtm_configuracao.png` | Captura PACTware (montagem) | Telas de configuração do DTM do FY301 real, antes de aplicar. | Tipo de válvula, curva, ar para abrir/fechar, limites, split range e Setup. |
| 13 | `fig13_lcd_fy301.png` | Captura LasecSimul (montagem) | Aproxime o display do FY301 (zoom de cerca de 300 %) e capture as duas páginas. | Posição e setpoint de posição. |
| 14 | `fig14_calibrador_ma.jpg` | Foto | Foto da bancada com o calibrador em modo fonte de mA no FY301 e o multímetro em série. | Corrente legível e a régua de curso. |
| 15 | `fig15_grafico_curso.png` | Gráfico | Planilha ou Python com as tabelas da seção 10.4. | Reta ideal, como encontrado, como deixado e a faixa de ±1 %. |
| 16 | `fig16_resposta_degrau.png` | Gráfico | Curso × tempo de dois degraus no simulador (ou no real), lidos dos displays a cada 0,1 s. | Tempo morto, tempo para 63 % e sobressinal. |
| 17 | `fig17_atrito_pressao.png` | Captura LasecSimul (montagem) | 12 mA com atrito de 0,5 % e de 5 %, estabilizado. | Mesma abertura, mais pressão no atuador com atrito alto. |
| 18 | `fig18_modem_paralelo.png` | Captura LasecSimul | Selecione o Modem HART e abra o painel Properties. | "Ligação na linha = Em paralelo na linha" e os bornes do FY301. |
