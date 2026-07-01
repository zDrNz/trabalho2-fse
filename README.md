# Nest Learning Thermostat — Estudo de Sistema Embarcado

## Integrantes

| Nome Completo | Matrícula |
|---|---|
| Carlos Daniel de Godoy Barros Nascimento | 190042303 |
| Daniel Rodrigues | 231037665 |
| Wesley Pedrosa dos Santos | 180029240 |

**Disciplina**: Fundamentos de Sistemas Embarcados — UnB  
**Semestre**: 2026/1  
**Data de Entrega**: 11/06/2026

---

## 1. Descrição do Produto Selecionado

### 1.1 Funções Principais, Público-Alvo e Contexto de Uso

O **Nest Learning Thermostat** (1ª e 2ª geração) é um termostato inteligente lançado em outubro de 2011 pela Nest Labs — empresa fundada por Tony Fadell e Matt Rogers, ex-engenheiros da Apple — e adquirida pelo Google em janeiro de 2014 por US$ 3,2 bilhões [[1]](#ref1). Foi o primeiro produto da categoria a popularizar o conceito de *learning thermostat*: um dispositivo capaz de aprender automaticamente as preferências de temperatura do usuário, construindo uma programação semanal sem intervenção manual.

**Público-alvo**: residências unifamiliares em países com sistemas de aquecimento, ventilação e ar-condicionado (HVAC) centralizados operando a 24V — predominantemente EUA, Canadá, Reino Unido e Europa Ocidental. Inicialmente direcionado a usuários preocupados com economia de energia e conforto térmico.

**Contexto de uso**: substitui termostatos programáveis convencionais (Honeywell, Carrier). Instalado na parede em substituição ao termostato original, é conectado ao sistema HVAC via cabeamento de baixa tensão (fios R, C, W, Y, G). Configuração e monitoramento remoto feitos por aplicativo móvel (iOS/Android).

**Principais funcionalidades**:

- **Auto-Schedule**: aprende os horários e temperaturas preferidos do usuário durante a primeira semana de uso e cria automaticamente uma programação semanal.
- **Auto-Away / Home & Away Assist**: utiliza sensor PIR e informações do smartphone (geolocalização) para detectar ausência e reduzir o consumo energético de forma automática.
- **Early On**: calcula antecipadamente o instante de partida do sistema HVAC para atingir a temperatura desejada no horário programado, com base no histórico de resposta do sistema.
- **Nest Leaf**: indicador visual exibido quando o usuário seleciona uma temperatura economicamente eficiente.
- **Energy History**: histórico de uso energético com relatórios diários e mensais via aplicativo.
- **Acesso remoto**: controle e monitoramento via Wi-Fi 802.11 b/g/n com integração à nuvem Google.

### Fotos do Produto

| Vista Frontal |
|---|
| ![Nest Gen 2 - Frente](NestThermostat\images\Diagrama.svg) | 


### 1.2 Componentes e Sensores Utilizados

A tabela abaixo resume os principais componentes internos do Nest Learning Thermostat (1ª e 2ª geração), identificados por meio de *teardowns* e documentação técnica disponível [[2]](#ref2)[[3]](#ref3):

| Componente | Especificação | Função |
|---|---|---|
| **Processador principal** | TI OMAP3 — ARM Cortex-A8 @ 600 MHz | Processamento central, Linux embarcado, algoritmos de aprendizado |
| **Co-processador** | ARM Cortex-M3 | Gestão de energia em modo *sleep*, leitura de sensores de baixo consumo |
| **Memória RAM** | 256 MB LPDDR | Execução do SO e algoritmos |
| **Armazenamento** | 2 GB eMMC Flash | Sistema operacional, logs e dados de aprendizado |
| **Display** | LCD circular 480 × 480 px, 2,08" | Exibição de temperatura atual/alvo, modo de operação e interface |
| **Sensor de temperatura** | Array de termistores NTC (múltiplos pontos internos) | Medição precisa de temperatura ambiente, eliminando gradientes locais |
| **Sensor de umidade** | Capacitivo interno | Leitura de umidade relativa exibida no aplicativo |
| **Sensor de presença** | PIR (*Passive Infrared*) — campo de visão cônico | Detecção de ocupação/ausência para Auto-Away |
| **Sensor de luz** | Fotodiodo | Ajuste adaptativo de brilho e ativação automática do display |
| **Wi-Fi** | Módulo IEEE 802.11 b/g/n (2,4 GHz) | Conectividade com nuvem Nest/Google e controle remoto via app |
| **Zigbee** *(Gen 2 em diante)* | IEEE 802.15.4 / Zigbee | Comunicação com Nest Protect e sensores de temperatura remotos [[5]](#ref5) |
| **Bateria** | Li-ion recarregável (~120 mAh) | Backup de energia e operação durante instalação |
| **Interface de usuário** | Anel magnético com *reed switches* (encoder) + clique central | Navegação nos menus e ajuste de temperatura |
| **Terminais HVAC** | 24V AC: R, RC, C, W, W2, Y, Y2, G, O/B, E, H | Controle elétrico do sistema de aquecimento, resfriamento e ventilação |

### 1.3 Tecnologias de Comunicação e Controle Embarcadas

- **Wi-Fi 802.11 b/g/n (2,4 GHz)**: Comunicação principal com a nuvem Nest/Google. Utiliza HTTPS/TLS para segurança e MQTT internamente para troca de mensagens de estado [[4]](#ref4).
- **Zigbee (IEEE 802.15.4)** *(a partir da 2ª geração)*: Protocolo de malha de baixo consumo para comunicação com o Nest Protect (detector de fumaça/CO) e sensores de temperatura remotos. Consome menos de 1% do tempo em transmissão ativa [[5]](#ref5).
- **Linux Embarcado (kernel 2.6.x)**: Sistema operacional base executado no ARM Cortex-A8. Permite execução de algoritmos de aprendizado de máquina em tempo real com isolamento de processos.
- **FreeRTOS no co-processador M3**: Gerencia sensores e consumo de energia com latência determinística em modo de baixo consumo, enquanto o Cortex-A8 permanece em *sleep*.
- **API REST — Nest Developer Platform**: Interface pública para integração com Google Home, Amazon Alexa, Apple HomeKit (via ponte) e sistemas de terceiros. Descontinuada em 2019 com a migração para Google Home SDK.

---

## 2. Análise Técnica do Funcionamento

### 2.1 Principais Módulos do Sistema

O Nest Learning Thermostat pode ser decomposto em cinco módulos funcionais:

#### Módulo 1 — Aquisição Sensorial

Responsável pela leitura contínua e filtragem de dados ambientais:

- **Temperatura**: array de termistores NTC internos, amostrado a cada ~5 s. A média de múltiplos sensores elimina gradientes causados pelo calor interno do próprio hardware.
- **Umidade relativa**: sensor capacitivo; utilizado para calcular ponto de orvalho e exibir conforto no app.
- **Presença humana**: sensor PIR ativo continuamente — detecta radiação infravermelha em movimento para inferir ocupação. Intervalo de detecção: ~5 m.
- **Luminosidade**: fotodiodo — ativa o display automaticamente quando o usuário se aproxima (*proximity wake*).

#### Módulo 2 — Processamento e Aprendizado

Núcleo diferenciador do produto. Executa em Linux embarcado sobre o ARM Cortex-A8:

- **Auto-Schedule**: análise de série temporal dos ajustes manuais realizados pelo usuário. Após 1–2 semanas, constrói uma grade horária semanal por inferência estatística de padrões de uso.
- **Auto-Away**: fusão de dados do PIR com geolocalização do smartphone (via app). Quando ambas as fontes indicam ausência, reduz a temperatura-alvo para o modo *Eco* (ex.: 17°C no inverno, 28°C no verão).
- **Early On** (controle preditivo): com base no histórico de tempo de resposta do sistema HVAC e na temperatura atual, calcula o instante de partida antecipado para atingir a temperatura desejada no horário programado — princípio análogo ao *Model Predictive Control* [[7]](#ref7).
- **Demand Response**: integração com o programa *Rush Hour Rewards* (EUA) — concessionárias de energia podem modular remotamente a temperatura durante picos de demanda na rede elétrica [[11]](#ref11).

#### Módulo 3 — Interface Humano-Máquina (IHM)

- **Display circular LCD 480×480**: renderiza temperatura atual, temperatura-alvo, modo de operação (calor/frio/ventilador/desligado), umidade relativa e ícone Nest Leaf.
- **Anel giratório magnético**: implementado com array de *reed switches* posicionados ao redor do anel permanentemente magnetizado. A rotação gera pulsos sequenciais nos switches, emulando um encoder incremental. A pressão central do anel confirma seleções.
- **Retroiluminação adaptativa**: controlada pelo fotodiodo + timer de inatividade; apaga automaticamente após ~10 s sem interação.

#### Módulo 4 — Controle do Atuador HVAC

Controla o sistema de aquecimento/resfriamento via chaveamento de sinais 24V AC:

| Terminal | Função |
|---|---|
| **W** | Aquecimento (liga caldeira ou resistência elétrica) |
| **Y** | Resfriamento (aciona compressor do ar-condicionado) |
| **G** | Ventilador (*fan only*) |
| **O/B** | Válvula reversora de bomba de calor (*heat pump*) |
| **C** | Fio comum — alimentação contínua para carregamento da bateria |

Lógica de histerese: intervalo mínimo de 5 minutos entre ciclos de liga/desliga (*minimum cycle time*) para proteger o compressor de *short-cycling*.

#### Módulo 5 — Conectividade e Nuvem

- Comunicação Wi-Fi constante com servidores Nest/Google Cloud (HTTPS/TLS + MQTT).
- Sincronização de estado, histórico energético, configurações de agenda e alertas.
- *Over-the-Air* (OTA) firmware updates automáticos.
- Integração com Google Home, Amazon Alexa e outros assistentes via API.

### 2.2 Identificação de Tecnologias Críticas

| Tecnologia | Relevância no Produto |
|---|---|
| **ARM Cortex-A8 + Linux embarcado** | Permite execução local de algoritmos de aprendizado sem dependência constante de nuvem; garante funcionamento mesmo sem internet |
| **Sensor PIR** | Viabiliza o Auto-Away de forma passiva, sem intervenção do usuário; entrada-chave do algoritmo de detecção de ocupação |
| **IEEE 802.15.4 / Zigbee** | Comunicação de baixíssimo consumo (µW em *sleep*) com sensores externos e Nest Protect em topologia de malha [[5]](#ref5) |
| **Modelo PMV/PPD de conforto térmico** | Base científica (ISO 7730) para definição do que constitui "temperatura confortável" no algoritmo de Auto-Schedule |
| **Model Predictive Control (MPC)** | Fundamento teórico da funcionalidade *Early On*: previsão do comportamento futuro do sistema HVAC para acionar o aquecimento/resfriamento no momento ótimo [[7]](#ref7) |
| **MQTT sobre Wi-Fi** | Protocolo *publish/subscribe* de baixo overhead para comunicação eficiente com a nuvem; suporte a QoS e reconexão automática [[4]](#ref4) |
| **OTA Firmware Update** | Permite correções de segurança e novas funcionalidades sem visita técnica — indispensável para produto residencial com base instalada de milhões de unidades |
| **Fusão de sensores** | Combinação de PIR + geolocalização do smartphone + histórico de uso para decisões de presença mais precisas do que cada sensor individualmente |

---

## 3. Proposta de Reprodução com ESP32

### 3.1 Descrição Conceitual

A proposta consiste em reproduzir as funcionalidades essenciais do Nest Learning Thermostat utilizando um **ESP32 DevKit C** como unidade central de processamento, aproveitando o suporte nativo a Wi-Fi 802.11 b/g/n, Bluetooth Low Energy e o ecossistema ESP-IDF com FreeRTOS.

O sistema proposto implementa as seguintes funcionalidades:

1. **Leitura de temperatura e umidade** via sensor **BME280** (I²C) — temperatura (±0,5°C), umidade relativa (±3%) e pressão barométrica com conversão automática de altitude.
2. **Interface de usuário** via **encoder rotativo KY-040** (CLK/DT/SW em GPIO com interrupção) + **display OLED SSD1306 0,96" 128×64 px** (I²C) — ajuste de temperatura-alvo, seleção de modo e visualização de status.
3. **Detecção de presença** via **sensor PIR HC-SR501** — ativa o display ao detectar aproximação e sinaliza ao algoritmo de Auto-Away a presença ou ausência de ocupantes.
4. **Controle do atuador** via **módulo relé de 2 canais (5V/10A)** — canal 1: aquecimento (sinal W), canal 2: resfriamento (sinal Y). Lógica de histerese (±0,5°C) e tempo mínimo de ciclo (5 min) implementados em software.
5. **Conectividade** via **Wi-Fi integrado** com protocolo **MQTT** (broker local: Mosquitto no Home Assistant, ou cloud: HiveMQ) — permite controle remoto, visualização de dados históricos e integração com automações.
6. **Algoritmo de agendamento** via **FreeRTOS tasks** — grade horária semanal configurável por MQTT/dashboard web; modo Auto-Away ativado após 30 minutos sem detecção pelo PIR.

### 3.2 Diagrama Conceitual do Sistema



![Diagrama Conceitual do Sistema](NestThermostat\images\Diagrama.svg)


### 3.3 Lista de Componentes

| Componente | Modelo | Interface | Qtd | Custo Estimado |
|---|---|---|---|---|
| Microcontrolador | ESP32 DevKit C (USB-C) | — | 1 | ~R$ 45 |
| Sensor Temp/Umi/Pressão | BME280 (módulo GY-BME280) | I²C (0x76) | 1 | ~R$ 25 |
| Display OLED | SSD1306 0,96" 128×64 px | I²C (0x3C) | 1 | ~R$ 18 |
| Encoder rotativo | KY-040 com botão integrado | GPIO (CLK, DT, SW) | 1 | ~R$ 10 |
| Sensor de presença | HC-SR501 (PIR ajustável) | GPIO | 1 | ~R$ 12 |
| Módulo relé | 2 canais 5V, 10A/250V AC | GPIO (ativo baixo) | 1 | ~R$ 20 |
| Fonte de alimentação | HLK-PM01 5V/3W (bivolt 100–240V) | — | 1 | ~R$ 30 |
| Resistores/capacitores | 10kΩ pull-up, 100nF bypass | — | kit | ~R$ 5 |
| Caixa plástica ABS | ~85×85×35 mm | — | 1 | ~R$ 15 |
| **Total estimado** | | | | **~R$ 180** |

### 3.4 Limitações e Desafios

| Desafio | Impacto | Possível Mitigação |
|---|---|---|
| **Ausência de display circular de alta resolução** | Interface visual muito simplificada: OLED monocromático 128×64 px vs LCD colorido 480×480 px | Substituir por display TFT 2,4" ILI9341 (SPI) com biblioteca LVGL para interface gráfica mais rica |
| **Poder de processamento limitado para ML** | ESP32 tem 520 KB SRAM — inviável rodar modelos de ML localmente com histórico extenso | Implementar heurística de agendamento baseada em regras simples ou delegar inferência ao servidor Home Assistant via MQTT |
| **Incompatibilidade com HVAC brasileiro** | Sistemas HVAC residenciais no Brasil operam em 127/220V split, sem fiação de baixa tensão 24V | Adaptar relé para controle do sinal de 12V do painel de controle do split, ou usar optoacoplador + SSR |
| **Alimentação via fiação HVAC** | Nest se alimenta do próprio cabeamento de 24V (fio C); ESP32 necessita de 5V DC | Usar fonte HLK-PM01 externa embutida na caixa, ou bateria LiPo 3,7V com carregador TP4056 |
| **Precisão do sensor PIR** | HC-SR501 tem zona morta central (~1 m) e maior taxa de falso positivo vs sensor PIR especializado do Nest | Combinar com leitura de RSSI de dispositivos Wi-Fi conhecidos (detecção de smartphone do morador) para reduzir falsos positivos |
| **Ausência de aprendizado automático real** | Sem dados de treinamento histórico suficientes e capacidade de ML local | Pré-programar horários via MQTT/dashboard e refinar manualmente; implementar agendamento adaptativo simples após coleta de 2 semanas de dados |
| **Segurança MQTT** | Broker sem autenticação expõe controle do HVAC | Configurar TLS (porta 8883) + autenticação usuário/senha + ACL no broker Mosquitto |
| **Certificações regulatórias** | Produto comercial requer aprovação ANATEL e INMETRO para dispositivos Wi-Fi e para equipamentos elétricos conectados à rede | Fora do escopo para protótipo acadêmico; relevante apenas para produto final |

---

## 4. Pesquisa Bibliográfica e Tecnológica

### 4.1 Artigos sobre Tecnologias que Viabilizam o Produto

---

#### [T1] Internet of Things: A Survey on Enabling Technologies, Protocols, and Applications

**Referência completa**: AL-FUQAHA, A.; GUIZANI, M.; MOHAMMADI, M.; ALEDHARI, M.; AYYASH, M. Internet of Things: A Survey on Enabling Technologies, Protocols, and Applications. **IEEE Communications Surveys & Tutorials**, v. 17, n. 4, p. 2347–2376, 4. trimestre 2015. DOI: [10.1109/COMST.2014.2346348](https://doi.org/10.1109/COMST.2015.2444095)

**Link do artigo**: [https://ieeexplore.ieee.org/document/7123563](https://ieeexplore.ieee.org/document/7123563)

**Resumo e relação com sistemas embarcados**: Este artigo apresenta uma revisão abrangente das tecnologias habilitadoras da Internet das Coisas (IoT), cobrindo protocolos de comunicação (MQTT, CoAP, AMQP, WebSocket), arquiteturas de rede (Wi-Fi, Zigbee, 6LoWPAN, Z-Wave, LTE-M) e plataformas de middleware. A relevância para o Nest Learning Thermostat é direta: o produto depende fundamentalmente de conectividade Wi-Fi 802.11 b/g/n e do protocolo MQTT para comunicação em tempo real com a nuvem Google. O artigo discute as vantagens do MQTT — protocolo *publish/subscribe* de baixo overhead desenvolvido pela IBM — em dispositivos com recursos energéticos restritos: cabeçalho mínimo de 2 bytes, suporte a QoS de 0 a 2 e reconexão automática por sessão persistente. Para um termostato embarcado que precisa manter conectividade constante enquanto gerencia múltiplas tasks de controle em FreeRTOS, a análise comparativa do artigo justifica a escolha do MQTT frente ao HTTP polling convencional. Adicionalmente, o trabalho discute desafios de segurança em dispositivos IoT residenciais — tópico de alta relevância dado que o Nest controla infraestrutura física da residência.

---

#### [T2] Wireless Sensor Networks: A Survey on the State of the Art and the 802.15.4 and ZigBee Standards

**Referência completa**: BARONTI, P.; PILLAI, P.; CHOOK, V. W.; CHESSA, S.; GOTTA, A.; HU, Y. F. Wireless sensor networks: A survey on the state of the art and the 802.15.4 and ZigBee standards. **Computer Communications**, v. 30, n. 7, p. 1655–1695, maio 2007. DOI: [10.1016/j.comcom.2006.12.020](https://doi.org/10.1016/j.comcom.2006.12.020)

**Link do artigo**: [https://www.science.smith.edu/~jcardell/Courses/EGR328/Readings/WSN_Zigbee%20Ovw.pdf](https://www.science.smith.edu/~jcardell/Courses/EGR328/Readings/WSN_Zigbee%20Ovw.pdf)

**Resumo e relação com sistemas embarcados**: Revisão técnica aprofundada do protocolo IEEE 802.15.4 e do padrão Zigbee, com análise das camadas PHY e MAC, topologias suportadas (estrela, árvore, malha) e comparação de desempenho em termos de latência, taxa de transferência, alcance e consumo energético. O Nest Learning Thermostat (2ª geração em diante) incorpora Zigbee para comunicação com o Nest Protect (detector de fumaça e CO) e para receber dados de temperatura de sensores de quarto remotos. O artigo demonstra as vantagens do Zigbee sobre o Wi-Fi para sensores de baixo consumo: ciclo de trabalho (*duty cycle*) de menos de 1% na camada MAC, corrente em transmissão de ~30 mA vs ~200 mA do Wi-Fi, e suporte a até 65.000 nós por rede. A topologia em malha (*mesh*) é particularmente relevante para residências maiores, onde o termostato precisa se comunicar com sensores remotos além do alcance de linha direta. O trabalho constitui a base técnica para compreender por que Zigbee foi escolhido como protocolo complementar ao Wi-Fi no ecossistema Nest.

---

#### [T3] Reinforcement Learning for Energy Conservation and Comfort in Buildings

**Referência completa**: DALAMAGKIDIS, K.; KOLOKOTSA, D.; KALAITZAKIS, K.; STAVRAKAKIS, G. S. Reinforcement learning for energy conservation and comfort in buildings. **Building and Environment**, v. 42, n. 7, p. 2686–2698, jul. 2007. DOI: [10.1016/j.buildenv.2006.07.010](https://doi.org/10.1016/j.buildenv.2006.07.010)

**Link do artigo**: [https://www.sciencedirect.com/science/article/pii/S0360132306001971](https://www.sciencedirect.com/science/article/pii/S0360132306001971)

**Resumo e relação com sistemas embarcados**: Este artigo propõe e avalia a aplicação de *Reinforcement Learning* (RL) — especificamente o algoritmo Q-Learning — para o controle autônomo de sistemas HVAC em edifícios, com o objetivo duplo de minimizar o consumo energético e manter o conforto térmico dos ocupantes. No paradigma de RL, um agente de controle aprende, por tentativa e erro ao longo do tempo, quais ações de controle (ligar/desligar aquecimento ou resfriamento, ajustar setpoint) maximizam uma função de recompensa que penaliza tanto o consumo excessivo de energia quanto o desvio da zona de conforto. Este princípio é o alicerce do algoritmo de Auto-Schedule do Nest Learning Thermostat: ao observar os ajustes manuais do usuário (sinais de recompensa implícita) e o comportamento resultante do sistema HVAC ao longo de dias e semanas, o termostato atualiza progressivamente sua política de controle até convergir para uma programação que reflete as preferências reais do ocupante. O artigo demonstra que agentes de RL adaptam-se a variações sazonais e mudanças de comportamento dos usuários sem necessidade de reprogramação manual — exatamente a proposta de valor central do Nest. Os resultados experimentais mostram que o RL supera controladores baseados em regras fixas em cenários de ocupação variável, validando a abordagem de aprendizado contínuo para sistemas embarcados de controle ambiental.

---

#### [T4] Use of Model Predictive Control and Weather Forecasts for Energy Efficient Building Climate Control

**Referência completa**: OLDEWURTEL, F.; PARISIO, A.; JONES, C. N.; GYALISTRAS, D.; GWERDER, M.; STAUCH, V.; LEHMANN, B.; MORARI, M. Use of model predictive control and weather forecasts for energy efficient building climate control. **Energy and Buildings**, v. 45, p. 15–27, fev. 2012. DOI: [10.1016/j.enbuild.2011.09.022](https://doi.org/10.1016/j.enbuild.2011.09.022)

**Link do artigo**: [https://www.researchgate.net/publication/257226788_Use_of_model_predictive_control_and_weather_forecasts_for_energy_efficient_building_climate_control](https://www.researchgate.net/publication/257226788_Use_of_model_predictive_control_and_weather_forecasts_for_energy_efficient_building_climate_control)

**Resumo e relação com sistemas embarcados**: Artigo do grupo de pesquisa de Manfred Morari (ETH Zurich) que demonstra, com validação experimental em um edifício real, a aplicação de *Model Predictive Control* (MPC) para controle de clima em edifícios, incorporando previsões meteorológicas como variáveis de entrada do modelo. O MPC resolve um problema de otimização em horizonte finito a cada intervalo de controle, calculando a sequência ótima de ações que minimiza o consumo energético sujeito a restrições de conforto térmico (banda de temperatura admissível). A funcionalidade *Early On* do Nest implementa uma variante simplificada desse conceito: ao aprender o tempo de resposta térmico do sistema HVAC de cada residência, o termostato calcula antecipadamente o instante de partida para atingir a temperatura desejada no horário programado — comportamento análogo ao *receding horizon* do MPC. Os resultados experimentais do artigo reportam reduções de 17–41% no consumo energético em relação a controladores do tipo liga/desliga com banda de histerese, validando cientificamente a abordagem preditiva que o Nest tornou acessível ao mercado residencial. O trabalho fundamenta matematicamente o principal diferencial de eficiência energética do produto estudado.

---

### 4.2 Artigos sobre Aplicação e Uso do Produto

---

#### [A1] How People Use Thermostats in Homes: A Review

**Referência completa**: PEFFER, T.; PRITONI, M.; MEIER, A.; ARAGON, C.; PERRY, D. How people use thermostats in homes: A review. **Building and Environment**, v. 46, n. 12, p. 2529–2541, dez. 2011. DOI: [10.1016/j.buildenv.2011.06.002](https://doi.org/10.1016/j.buildenv.2011.06.002)

**Link do artigo**: [https://www.sciencedirect.com/science/article/pii/S0360132311001739](https://www.sciencedirect.com/science/article/pii/S0360132311001739)

**Resumo e relação com sistemas embarcados**: Revisão sistemática de 15 estudos de campo sobre o comportamento humano em relação a termostatos residenciais, conduzida por pesquisadores da Lawrence Berkeley National Laboratory / UC Berkeley. O estudo revela que a maioria dos usuários *não programa* seus termostatos convencionais — seja por dificuldade de uso da interface (interfaces com muitos botões e menus contraintuitivos), falta de interesse ou simples esquecimento — resultando em consumo energético sistematicamente acima do ótimo. Esta constatação é a justificativa central de mercado para o Nest Learning Thermostat: ao eliminar a necessidade de programação manual e substituí-la por aprendizado automático com interface minimalista (um único anel giratório + display intuitivo), o produto resolve um problema comportamental documentado e mensurado. O artigo também identifica que reduções de temperatura durante períodos de ausência e à noite representam o maior potencial de economia — exatamente as duas funcionalidades prioritárias do Nest (Auto-Away e configuração noturna do Auto-Schedule). Constitui a referência bibliográfica que fundamenta a *razão de existir* do produto estudado.

---

#### [A2] Predicting Household Occupancy for Smart Heating Control

**Referência completa**: KLEIMINGER, W.; MATTERN, F.; SANTINI, S. Predicting household occupancy for smart heating control: A comparative performance analysis of state-of-the-art approaches. **Energy and Buildings**, v. 85, p. 493–505, dez. 2014. DOI: [10.1016/j.enbuild.2014.09.046](https://doi.org/10.1016/j.enbuild.2014.09.046)

**Link do artigo**: [https://www.sciencedirect.com/science/article/pii/S0378778814007737](https://www.sciencedirect.com/science/article/pii/S0378778814007737)

**Resumo e relação com sistemas embarcados**: Pesquisa do ETH Zurich que compara múltiplos algoritmos de predição de ocupação residencial — Cadeias de Markov Ocultas (HMM), redes neurais, Naive Bayes, regressão logística e *random forest* — usando dados reais de consumo elétrico e sensores PIR coletados em 30 residências suíças durante 12 meses. Os resultados mostram que modelos baseados em HMM atingem aproximadamente 85% de precisão na predição de presença com apenas 2 semanas de dados históricos — período análogo ao que o Nest utiliza para aprender a rotina do usuário. O trabalho demonstra que a combinação de leitura de sensores PIR (informação local imediata) com padrões históricos temporais (dia da semana, hora do dia) é a abordagem de maior custo-benefício para sistemas embarcados com recursos limitados, validando precisamente a estratégia de hardware e software adotada pelo Nest. Para a proposta de implementação com ESP32, este artigo orienta diretamente a escolha e implementação do algoritmo de Auto-Away mais eficaz dentro das restrições de memória e processamento do microcontrolador.

---

#### [A3] Energy Efficiency and the Misuse of Programmable Thermostats

**Referência completa**: PRITONI, M.; MEIER, A. K.; ARAGON, C.; PERRY, D.; PEFFER, T. Energy efficiency and the misuse of programmable thermostats: The effectiveness of crowdsourcing for understanding household behavior. **Energy Research & Social Science**, v. 8, p. 190–197, jul. 2015. DOI: [10.1016/j.erss.2015.06.002](https://doi.org/10.1016/j.erss.2015.06.002)

**Link do artigo**: [https://www.sciencedirect.com/science/article/abs/pii/S2214629615000730](https://www.sciencedirect.com/science/article/abs/pii/S2214629615000730)

**Resumo e relação com sistemas embarcados**: Estudo da UC Berkeley que analisa dados de 1.400 usuários de termostatos — incluindo usuários de termostatos inteligentes Nest — coletados via metodologia de *crowdsourcing*, investigando se termostatos programáveis e seus sucessores inteligentes efetivamente reduzem o consumo energético na prática. Os resultados confirmam que termostatos inteligentes com aprendizado automático geram economia real e consistente quando o usuário historicamente não ajustava o termostato manual, mas o benefício é reduzido para usuários já disciplinados na programação. O artigo quantifica o efeito de *setback* (redução de temperatura em período de ausência ou noturno): cada 1°C de redução na temperatura de aquecimento gera economia de aproximadamente 3% na conta de energia. Para o Nest, que anuncia economias de 10–12% [[1]](#ref1), isso implica um *setback* médio efetivo de 3–4°C aplicado de forma consistente — dado verificável e mensurável que pode ser replicado em uma implementação com ESP32 com algoritmo de Auto-Away funcional. O trabalho também demonstra o valor da instrumentação remota via Wi-Fi para coleta de dados de uso em larga escala, validando o modelo de produto conectado à nuvem.

---

#### [A4] Achieving Controllability of Electric Loads

**Referência completa**: CALLAWAY, D. S.; HISKENS, I. A. Achieving controllability of electric loads. **Proceedings of the IEEE**, v. 99, n. 1, p. 184–199, jan. 2011. DOI: [10.1109/JPROC.2010.2081652](https://doi.org/10.1109/JPROC.2010.2081652)

**Link do artigo**: [https://ieeexplore.ieee.org/document/5590109](https://ieeexplore.ieee.org/document/5590109)

**Resumo e relação com sistemas embarcados**: Artigo seminal publicado nos *Proceedings of the IEEE* sobre *demand response* (resposta à demanda) em redes elétricas inteligentes (*smart grids*), com foco no controle agregado de cargas termostáticas (TCL — *Thermostatically Controlled Loads*): termostatos de HVAC, aquecedores de água e refrigeradores. O trabalho propõe modelos matemáticos baseados em equações de calor e estratégias de controle distribuído para que operadores de redes elétricas possam modular o consumo agregado de grandes populações de termostatos residenciais em resposta a variações de frequência da rede — sem impacto perceptível no conforto dos usuários individuais. Esta capacidade é implementada no Nest pelo programa **Rush Hour Rewards** (EUA e Reino Unido): concessionárias de energia ajustam remotamente a temperatura-alvo dos termostatos Nest cadastrados em períodos de pico de demanda, em troca de créditos na conta de energia do usuário. O artigo estabelece a base teórica para essa integração entre termostatos inteligentes e *smart grids*, demonstrando que o Nest não é apenas um produto de conforto residencial, mas também um nó ativo de controle de demanda em infraestrutura crítica de energia. Representa uma das aplicações mais impactantes de sistemas embarcados conectados na modernização da rede elétrica.

---

## 5. Comparativo com Produtos Similares

Os produtos abaixo representam o espectro histórico e contemporâneo de termostatos residenciais, evidenciando a evolução tecnológica da categoria desde a era pré-conectada até os modelos atuais.

| Característica | **Nest Learning Thermostat Gen 1** (2011) | **Honeywell VisionPRO 8000** (2008) | **Ecobee3** (2014) | **Emerson Sensi ST55** (2016) | **Honeywell Home T9** (2019) | **Google Nest Thermostat** (2020) |
|---|---|---|---|---|---|---|
| **Preço de lançamento** | US$ 249 | US$ 100–150 | US$ 249 | US$ 130 | US$ 199 | US$ 130 |
| **Processador** | ARM Cortex-A8 @ 600 MHz | MCU 8-bit proprietário | ARM Cortex-A8 | MCU 32-bit | ARM Cortex-A | ARM Cortex-A |
| **Sistema operacional** | Linux Embarcado (kernel 2.6) | Firmware proprietário | Linux Embarcado | FreeRTOS | Linux Embarcado | Android Things / Linux |
| **Display** | LCD circular 480×480 px, 2,08" | LCD retangular 3" | LCD colorido 3,5" | LCD monocromático 2,5" | LCD colorido 2,9" | LCD circular 2,08" |
| **Wi-Fi** | 802.11 b/g/n (2,4 GHz) | Não | 802.11 b/g/n | 802.11 b/g/n | 802.11 b/g/n | 802.11 b/g/n |
| **Zigbee / Z-Wave** | Não (Gen 2+: Zigbee) | Não | Não | Não | Zigbee (sensores de quarto) | Não |
| **Bluetooth** | Não | Não | Não | BLE (configuração inicial) | BLE | BLE |
| **Sensor de temperatura** | NTC Thermistor (array interno) | NTC (interno) | NTC (interno + sensores remotos) | NTC (interno) | NTC (interno + quarto via Zigbee) | NTC (interno) |
| **Sensor de umidade** | Sim | Não | Sim | Sim | Sim | Não |
| **Sensor de presença (PIR)** | Sim | Não | Sim | Não | Sim | Sim |
| **Sensor de luminosidade** | Sim | Não | Não | Não | Não | Sim |
| **Aprendizado automático** | **Sim** (Auto-Schedule) | Não | Parcial (SmartRecovery) | Não | Não | **Não** |
| **Sensores de quarto remotos** | Não (Protect via Zigbee) | Não | Sim (até 32 sensores adicionais) | Não | Sim (SmartRoom Sensor) | Não |
| **Controle preditivo (*Early On*)** | **Sim** | Não | Sim (Smart Recovery) | Não | Sim | Sim |
| **Aplicativo móvel** | iOS / Android | Acesso web via total Connect | iOS / Android | iOS / Android | iOS / Android | iOS / Android |
| **Geofencing (Auto-Away)** | Sim (Home & Away Assist) | Não | Sim | Sim | Sim | Sim |
| **Integração com assistentes de voz** | Google Home / Alexa | Alexa (via skill) | Google / Alexa / Siri | Google / Alexa | Google / Alexa | Google Home / Alexa |
| **API aberta para desenvolvedores** | Sim (Nest API — descontinuada 2019) | Não | Sim (ecobee API) | Não | Sim (Resideo API) | Sim (Google Home API) |
| **OTA Firmware Update** | Sim | Não | Sim | Sim | Sim | Sim |
| **Terminais HVAC suportados** | R, RC, C, W, W2, Y, Y2, G, O/B, E, H | R, C, W, Y, G, O/B, E, S | R, C, W, Y, G, O/B | R, C, W, Y, G | R, C, W, Y, G, O/B, S | R, C, W, Y, G, O/B |
| **Alimentação** | Fio C / Bateria Li-ion recarregável | 3× Bateria AA | Fio C obrigatório | Fio C / 2× Bateria AA | Fio C / 3× Bateria AAA | Fio C / Bateria interna |
| **Consumo típico** | ~1 W (ativo) / µW (sleep M3) | ~50 mW | ~1,5 W | ~800 mW | ~1 W | ~800 mW |
| **Dimensões** | Ø 83 mm × 33 mm | 97×97×26 mm | 107×107×29 mm | 83×83×28 mm | 91×91×30 mm | Ø 83 mm × 21 mm |
| **Certificações** | UL, FCC, IC, RoHS | UL, FCC | UL, FCC, IC | UL, FCC, IC | UL, FCC, IC | UL, FCC, IC, CE |

**Observações comparativas**:

- O **Honeywell VisionPRO 8000** representa a geração imediatamente anterior ao Nest: totalmente programável (7 dias × 4 períodos), mas sem conectividade, aprendizado ou sensores de presença. Demonstra o estado da arte que o Nest disruptou em 2011.
- O **Ecobee3** é o principal diferencial em relação ao Nest: suporte a até 32 sensores de temperatura remotos (SmartSensors com bateria e PIR próprios), resolvendo o problema de temperatura não-uniforme entre cômodos — limitação relevante do Nest.
- O **Emerson Sensi ST55** representa a categoria "conectado básico": adiciona Wi-Fi e app ao termostato tradicional com custo mínimo, sem aprendizado ou sensores avançados. Demonstra a segmentação do mercado pós-Nest.
- O **Honeywell Home T9** combina Zigbee + sensores de quarto + geofencing, herdando o ecossistema de sensores remotos do Ecobee mas com integração ao ecossistema Honeywell/Resideo. Concorre diretamente com o Nest Gen 3.
- O **Google Nest Thermostat (2020)** é o sucessor de posicionamento acessível: removeu o aprendizado automático para reduzir custo (US$ 130 vs US$ 249), mas manteve a estética circular e o ecossistema Google Home. Representa a consolidação da marca após a aquisição pelo Google.

---

## Referências Bibliográficas

<a name="ref1"></a>[1] Google Nest. *Energy Savings from Google Nest Thermostats*. Mountain View, CA: Google Nest, 2024. Disponível em: [https://storage.googleapis.com/mannequin/blobs/e1e272e8-a82e-4a2d-a864-71f6fe505369.pdf?utm_source=syndication](https://storage.googleapis.com/mannequin/blobs/e1e272e8-a82e-4a2d-a864-71f6fe505369.pdf?utm_source=syndication).

<a name="ref2"></a>[2] iFixit. *Nest Learning Thermostat Teardown*. Disponível em: [https://pt.ifixit.com/Teardown/Nest+Learning+Thermostat+2nd+Generation+Teardown/13818](https://pt.ifixit.com/Teardown/Nest+Learning+Thermostat+2nd+Generation+Teardown/13818). Acesso em: jun. 2026.

<a name="ref3"></a>[3] FCC ID Database. *Nest Learning Thermostat — FCC Equipment Authorization*. ID: ZQAT30. Disponível em: [https://fccid.io/ZQAT30](https://fccid.io/ZQAT30). Acesso em: jun. 2026.

<a name="ref4"></a>[4] AL-FUQAHA, A.; GUIZANI, M.; MOHAMMADI, M.; ALEDHARI, M.; AYYASH, M. Internet of Things: A Survey on Enabling Technologies, Protocols, and Applications. **IEEE Communications Surveys & Tutorials**, v. 17, n. 4, p. 2347–2376, 2015. DOI: 10.1109/COMST.2014.2346348.

<a name="ref5"></a>[5] BARONTI, P.; PILLAI, P.; CHOOK, V. W.; CHESSA, S.; GOTTA, A.; HU, Y. F. Wireless sensor networks: A survey on the state of the art and the 802.15.4 and ZigBee standards. **Computer Communications**, v. 30, n. 7, p. 1655–1695, 2007. DOI: 10.1016/j.comcom.2006.12.020.

<a name="ref6"></a>[6] DALAMAGKIDIS, K.; KOLOKOTSA, D.; KALAITZAKIS, K.; STAVRAKAKIS, G. S. Reinforcement learning for energy conservation and comfort in buildings. **Building and Environment**, v. 42, n. 7, p. 2686–2698, 2007. DOI: 10.1016/j.buildenv.2006.07.010.

<a name="ref7"></a>[7] OLDEWURTEL, F.; PARISIO, A.; JONES, C. N.; GYALISTRAS, D.; GWERDER, M.; STAUCH, V.; LEHMANN, B.; MORARI, M. Use of model predictive control and weather forecasts for energy efficient building climate control. **Energy and Buildings**, v. 45, p. 15–27, 2012. DOI: 10.1016/j.enbuild.2011.09.022.

[8] PEFFER, T.; PRITONI, M.; MEIER, A.; ARAGON, C.; PERRY, D. How people use thermostats in homes: A review. **Building and Environment**, v. 46, n. 12, p. 2529–2541, 2011. DOI: 10.1016/j.buildenv.2011.06.002.

[9] KLEIMINGER, W.; MATTERN, F.; SANTINI, S. Predicting household occupancy for smart heating control: A comparative performance analysis of state-of-the-art approaches. **Energy and Buildings**, v. 85, p. 493–505, 2014. DOI: 10.1016/j.enbuild.2014.09.046.

[10] PRITONI, M.; MEIER, A. K.; ARAGON, C.; PERRY, D.; PEFFER, T. Energy efficiency and the misuse of programmable thermostats: The effectiveness of crowdsourcing for understanding household behavior. **Energy Research & Social Science**, v. 8, p. 190–197, 2015. DOI: 10.1016/j.erss.2015.06.002.

[11] CALLAWAY, D. S.; HISKENS, I. A. Achieving controllability of electric loads. **Proceedings of the IEEE**, v. 99, n. 1, p. 184–199, 2011. DOI: 10.1109/JPROC.2010.2081652.

---
