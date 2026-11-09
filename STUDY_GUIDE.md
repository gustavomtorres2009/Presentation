# Guia de Estudo e Preparação Técnica: Android Automotive OS (Android 16 + Raspberry Pi 5)

Este documento foi elaborado como **material de estudo aprofundado** para preparar você para a apresentação de 45 minutos. Aqui você encontrará a explicação conceitual e técnica de **cada tópico, sigla, camada arquitetural e comando** presente nos slides, além de analogias práticas para responder dúvidas de engenheiros com total segurança.

---

## Sumário
1. [A Narrativa da Apresentação (Como Conectar os Blocos)](#1-a-narrativa-da-apresentação-como-conectar-os-blocos)
2. [Bloco 1: Fundamentos do Ecossistema Automotivo e Hardware (Slides 1 a 3)](#2-bloco-1-fundamentos-do-ecossistema-automotivo-e-hardware-slides-1-a-3)
3. [Bloco 2: Deep Dive na Arquitetura Interna do AAOS (Slides 4 a 8)](#3-bloco-2-deep-dive-na-arquitetura-interna-do-aaos-slides-4-a-8)
4. [Bloco 3: Engenharia de Compilação do AOSP Android 16 na RPi 5 (Slides 9 a 15)](#4-bloco-3-engenharia-de-compilação-do-aosp-android-16-na-rpi-5-slides-9-a-15)
5. [Bloco 4: Sequência de Boot, Debugging e Desafios de Bancada (Slides 16 a 18)](#5-bloco-4-sequência-de-boot-debugging-e-desafios-de-bancada-slides-16-a-18)
6. [Bloco 5: Roadmap de Evolução e Fechamento (Slides 19 a 20)](#6-bloco-5-roadmap-de-evolução-e-fechamento-slides-19-a-20)
7. [Glossário Rápido de Siglas Técnicas (Vocabulário de Bolso)](#7-glossário-rápido-de-siglas-técnicas-vocabulário-de-bolso)

---

## 1. A Narrativa da Apresentação (Como Conectar os Blocos)

Uma ótima apresentação técnica conta uma história lógica de engenharia. A sua história segue este fio condutor:
1. **O Contexto (Bloco 1):** *"O mercado automotivo mudou para veículos definidos por software. O Android Automotive OS (AAOS) é o sistema operacional nativo dessa transformação, e nós escolhemos a Raspberry Pi 5 com Android 16 para ter uma bancada ARM64 real em vez de apenas um emulador."*
2. **A Teoria/Arquitetura (Bloco 2):** *"Antes de mostrar como compilamos, precisamos entender o que torna o Android Automotivo diferente do Android de celular: o `CarService`, o modelo de propriedades e a camada `VHAL`."*
3. **A Construção Prática (Bloco 3):** *"Com a arquitetura clara, veja como nossa engenharia baixou mais de 1.100 repositórios do Android 16, injetou os drivers da Raspberry Pi 5 e compilou do zero nossa própria imagem de sistema."*
4. **A Prova Real e Depuração (Bloco 4):** *"Aqui está o sistema inicializando no hardware físico, como nós injetamos eventos de velocidade via linha de comando para testar a segurança da interface e quais gargalos elétricos/gráficos superamos."*
5. **O Futuro do Produto (Bloco 5):** *"Agora que dominamos o ciclo completo do AOSP na RPi 5, nossos próximos passos são plugar um barramento CAN real, customizar a interface via Overlays (RRO) e usar duas telas."*

---

## 2. Bloco 1: Fundamentos do Ecossistema Automotivo e Hardware (Slides 1 a 3)

### 2.1 Por que o AAOS existe? (Contexto de Mercado e Engenharia)
Antigamente, cada montadora (OEM) construía sua própria central multimídia (*In-Vehicle Infotainment* — **IVI**) sobre sistemas Linux customizados ou QNX. O problema: manter navegadores, stacks Bluetooth modernos, streaming de música e interfaces fluidas a 60 fps custava bilhões e ficava defasado rapidamente.
O **AAOS (Android Automotive OS)** resolve isso trazendo a maturidade do Android diretamente para o computador de bordo do veículo.

### 2.2 A Diferença Clássica: Android Auto vs. AAOS vs. GAS (Slide 2)
É muito comum que até desenvolvedores experientes confundam os três termos. Explique assim:

* **Android Auto (Projeção):**
  * **Como funciona tecnicamente:** O aplicativo roda no processador do **celular** do motorista. O celular renderiza a tela, codifica um stream de vídeo (`H.264`/`H.265`) e envia pelo cabo USB ou Wi-Fi Direct para a tela do carro. Quando você toca na tela do carro, a central multimídia apenas envia as coordenadas `(X, Y)` do toque de volta para o celular.
  * **Limitação:** O Android Auto **não manda no carro**. Ele não consegue ligar o ar-condicionado, ler a temperatura da bateria do veículo elétrico ou mudar configurações do chassi.
* **Android Automotive OS — AAOS (O Sistema Operacional Base no AOSP):**
  * **Como funciona tecnicamente:** Não precisa de celular. O Android está gravado na memória flash do próprio carro e roda no processador da central multimídia. Ele faz parte do **AOSP (Android Open Source Project)** — ou seja, é código aberto mantido pela Google e parceiros.
  * **Poder:** Como é o SO nativo do veículo, ele conversa com os módulos eletrônicos (ECUs) do carro através do **VHAL**, controlando ar-condicionado (HVAC), bancos aquecidos, câmeras de ré e painel de instrumentos.
* **GAS (Google Automotive Services):**
  * **Como funciona tecnicamente:** Como o AAOS é aberto (AOSP), ele vem com aplicativos básicos de referência, mas **não vem** com Google Maps, Google Play Store ou Google Assistant (assim como um Android AOSP de celular não vem com os apps proprietários da Google, chamados de GMS).
  * **Modelo de negócio:** Quando uma montadora (ex.: Volvo, Renault, GM) quer os serviços da Google nativos no painel, ela licencia o pacote **GAS** e passa por testes de certificação (**CTS/VTS**). No nosso projeto na Raspberry Pi 5, usamos o **AAOS puro (AOSP Android 16)**, o que nos dá liberdade total para criar nossa própria experiência e loja/ecossistema sem depender de licença da Google.

### 2.3 Por que escolhemos a Raspberry Pi 5 em vez do Emulador? (Slide 3)
Se alguém perguntar *"Por que vocês tiveram o trabalho de compilar para a Raspberry Pi 5 em vez de usar o emulador do Android Studio?"*, seus argumentos técnicos são:
1. **Arquitetura de CPU Real (`arm64-v8a`):** O computador do desenvolvedor geralmente é `x86_64` (Intel/AMD). Já os processadores usados em carros de verdade (como a linha *Qualcomm Snapdragon Cockpit* ou *NXP i.MX8*) são todos **ARM de 64 bits**. A Raspberry Pi 5 usa o SoC **Broadcom BCM2712** (Quad-core ARM Cortex-A76 de 64 bits rodando a 2.4 GHz). Compilar para a RPi 5 valida *cross-compilation* real para ARM64.
2. **Stack Gráfico Real (GPU VideoCore VII):** No emulador, o desenho de tela usa drivers virtuais (`virtio-gpu`) repassados para a placa de vídeo do PC. Na Raspberry Pi 5, exercitamos o pipeline gráfico real do Linux embarcado: **DRM/KMS** + **Mesa3D** (`v3d`).
3. **Pinos de Hardware Físicos (GPIO / SPI / I2C / UART):** Um emulador não tem pinos físicos. A Raspberry Pi 5 possui um barramento de 40 pinos onde podemos ligar um módulo transceptor **CAN Bus** real, um GPS serial ou botões físicos de painel.

---

## 3. Bloco 2: Deep Dive na Arquitetura Interna do AAOS (Slides 4 a 8)

Este é o bloco onde você demonstra domínio sobre o funcionamento interno do Android.

### 3.1 A Separação Treble: `/system` vs. `/vendor` (Slide 4)
Desde o Android 8 (*Project Treble*), o Android é dividido em duas metades independentes:
* **Camada de Framework (`/system`):** Código genérico do Android escrito em Java/Kotlin/C++ pela Google (`CarService`, `CarLauncher`, `SystemUI`). Não conhece detalhes específicos do chip da Broadcom ou de como os fios do carro são ligados.
* **Camada de Hardware / BSP (`/vendor`):** Código específico da placa (`Raspberry Pi 5`) e da montadora. Contém os **HALs (Hardware Abstraction Layers)** — pequenos serviços em C++ que conversam com os drivers do Kernel Linux.
* **Como as duas metades conversam?** Através de **Binder IPC** (comunicação entre processos ultra-rápida do Android) usando contratos formais escritos em **Stable AIDL** (*Android Interface Definition Language*).

### 3.2 O `CarService` e o `CarPropertyManager` (Slide 5)
No Android comum de celular, você tem serviços como `LocationManager` ou `SensorManager`. No Android Automotive, existe um serviço mestre chamado **`CarService`** (localizado em `packages/services/Car/service/`).

#### Como o Android enxerga os dados do carro? O Modelo de Propriedades!
Para o Android não precisar saber se um dado vem de uma rede CAN, LIN, Ethernet Automotiva ou SPI, tudo no carro é padronizado como uma **Propriedade (`VehicleProperty`)**.
Cada propriedade possui um **ID numérico de 32 bits (hexadecimal)** definido no arquivo `VehicleProperty.aidl`.

> **Dica de Ouro para impressionar na apresentação:** Explique que o número hexadecimal como `0x11600207` (`PERF_VEHICLE_SPEED`) não é aleatório! Ele é uma máscara de bits (bitmask) que codifica metadados na própria estrutura do ID:
> - **`0x10000000` (Group):** `SYSTEM` (propriedade padrão definida pelo AOSP; se começasse com `0x20000000`, seria `VENDOR`, ou seja, criada de forma customizada pela nossa equipe!).
> - **`0x01000000` (Area):** `GLOBAL` (a velocidade é do carro inteiro; já no ar-condicionado `0x15600503`, o `5` indica `SEAT`, pois cada assento pode ter uma temperatura diferente!).
> - **`0x00600000` (Type):** `FLOAT` (número decimal em ponto flutuante; no câmbio `0x11400400`, o `4` indica `INT32`).
> - **`0x00000207` (Index):** Identificador único da propriedade Velocidade.

#### Como um App lê uma Propriedade?
O aplicativo Android nunca acessa o hardware diretamente. Ele instancia o objeto `Car`, pede o `CarPropertyManager` e registra um *listener* (`registerCallback`). Quando a velocidade muda lá no hardware, o evento sobe:
`Hardware/CAN` $\rightarrow$ `VHAL (C++)` $\rightarrow$ `CarPropertyService (Java)` $\rightarrow$ `Callback do App`.

### 3.3 O VHAL (Vehicle Hardware Abstraction Layer) em Stable AIDL (Slide 6)
O **VHAL** é o componente mais importante de qualquer projeto Android Automotive.
* **Onde fica no código do AOSP?** `hardware/interfaces/automotive/vehicle/aidl/`
* **Evolução no Android 16:** Em versões antigas do Android (Android 10 a 12), o VHAL era escrito em *HIDL*. A partir do Android 13/14 e consolidado no **Android 16**, o VHAL utiliza exclusivamente **Stable AIDL** (`IVehicle.aidl`).
* **Como o VHAL funciona na nossa Raspberry Pi 5?**
  Como a Raspberry Pi 5 recém-instalada na mesa não está plugada no motor de um carro, o AOSP fornece o **Reference VHAL** (`vendor.vehicle-hal-default`).
  Ele é dividido em duas partes em C++:
  1. `DefaultVehicleHal`: Cuida de conversar com o Android (`getValues`, `setValues`, `subscribe`).
  2. `IVehicleHardware`: É a classe abstrata que fala com o mundo físico. Na imagem padrão, ela usa um simulador baseado em arquivos JSON (`/vendor/etc/automotive/vhalconfig/DefaultConfig.json`). Para transformar nosso protótipo em um produto conectado ao carro, nós só precisamos substituir essa implementação de `IVehicleHardware` para ler/escrever na porta **CAN Bus** da Raspberry Pi 5!

### 3.4 Áudio Multizona (`CarAudioService`) e Power Management (`CPMS`) (Slide 7)
Por que os subsistemas de áudio e energia de um celular não servem para um carro?

1. **Áudio (`CarAudioService`):**
   * No celular, só existe 1 pessoa usando e 1 saída ativa por vez (ou alto-falante, ou fone Bluetooth).
   * No carro, temos **múltiplas zonas acústicas** (`car_audio_configuration.xml`). O motorista pode precisar ouvir o alerta de radar do GPS apenas nas caixas de som dianteiras, enquanto os filhos no banco traseiro assistem a um desenho em outra tela/saída de áudio.
   * Além disso, existe o **Ducking de Segurança**: se o sensor de estacionamento apitar (`USAGE_ASSISTANCE_SONification` / `SAFETY`), o `AudioControl HAL` baixa o volume do Spotify instantaneamente por questão de segurança física.
2. **Energia (`CarPowerManagementService` - CPMS):**
   * Se o Android ficasse ligado 24h por dia com o carro estacionado, ele drenaria a bateria de 12V do carro em poucos dias.
   * Se o Android desligasse 100% (*Cold Shutdown*) toda vez que você gira a chave, o motorista teria que esperar 30 a 40 segundos de boot toda vez que entrasse no carro.
   * **A Solução Automotiva:**
     * **Suspend-to-RAM (Deep Sleep):** Quando você desliga o carro, o CPMS fecha os apps pesados, apaga a tela, desliga os periféricos e coloca a memória RAM em modo de auto-retenção (consumindo pouquíssimos miliwatts). Quando você abre a porta do carro, ele acorda em **menos de 2 segundos**.
     * **Garage Mode:** Antes de entrar em *Deep Sleep*, o Android entra por alguns minutos no *Garage Mode* (com a tela apagada) para baixar atualizações de sistema (OTA) e sincronizar dados sem atrapalhar o motorista.

### 3.5 Segurança do Motorista (`CarUxRestrictions`) e Multi-User `HSUM` (Slide 8)
* **Prevenção de Distração (`CarDrivingStateService` $\rightarrow$ `CarUxRestrictionsManagerService`):**
  * Legislações de trânsito mundiais (NHTSA, UNECE) proíbem que o painel permita digitar textos longos ou assistir vídeos enquanto o veículo está em movimento.
  * No AAOS, isso é nativo do sistema operacional: assim que o VHAL informa que a marcha saiu de `PARK` (`GEAR_SELECTION`) e a velocidade (`PERF_VEHICLE_SPEED`) passou de zero, o estado muda de `STATE_PARKED` para `STATE_MOVING`.
  * Automaticamente, o Android esconde o teclado virtual, pausa vídeos e coloca um aviso de bloqueio nos menus de Configuração (`CarSettings`) que não possuem a anotação Java `@DistractionOptimized`.
* **HSUM (*Headless System User Mode*):**
  * No celular, o dono do aparelho é o `User 0` (que é ao mesmo tempo o administrador do sistema e o usuário da interface gráfica).
  * Um carro é um bem compartilhado (marido, esposa, manobrista). Por isso, o AAOS roda em modo **HSUM**: o `User 0` roda "sem cabeça" (*headless*, sem interface gráfica), cuidando apenas de serviços vitais do carro. O primeiro motorista humano que você vê na tela já é o **`User 10`**. Se outro motorista logar, será o **`User 11`**, garantindo que um motorista não veja o WhatsApp, histórico do Maps ou contas do outro.

---

## 4. Bloco 3: Engenharia de Compilação do AOSP Android 16 na RPi 5 (Slides 9 a 15)

Neste bloco você explica **como o produto foi construído na prática** seguindo a metodologia documentada na *Make Android*.

### 4.1 Por que compilar o AOSP exige um "Supercomputador"? (Slide 10)
* O código-fonte do Android 16 sincronizado ocupa cerca de **180 GB** em disco (são milhões de arquivos em C, C++, Java, Kotlin e Rust).
* Quando compilamos, o diretório `out/` gera mais **200 GB a 250 GB** de objetos intermediários e imagens.
* **Por que 64 GB de RAM?** Durante a compilação paralela (`m -j$(nproc)`), se o seu processador tem 16 ou 24 threads, o sistema de build abre 24 processos simultâneos do compilador `clang++`, `rustc` e do analisador de APIs Java `metalava`. Na hora de juntar (*linkar*) bibliotecas gigantescas (como o `libwebviewchromium.so` ou o `framework.jar`), cada thread pode consumir de 2 GB a 3 GB de RAM. Se faltar RAM + Swap, o Kernel Linux do Ubuntu aciona o **OOM Killer** (*Out-Of-Memory Killer*) e aborta o build com erro `Killed` (código 137).

### 4.2 O Utilitário `repo` da Google (Slide 11)
* **Por que não usamos apenas `git clone`?** Porque o Android não é um único repositório Git. Ele é dividido em **mais de 1.100 repositórios Git separados** (um para o Bluetooth, um para a calculadora, um para o `CarService`, um para o Kernel, etc.).
* O **`repo`** é um script em Python criado pela Google que lê um arquivo XML mestre (`manifest.xml`) e faz o `git clone` e `git checkout` coordenado desses +1.100 repositórios.
* **Por que usamos `--depth=1` no `repo init`?** Sem essa flag, o Git baixaria todo o histórico de commits do Android desde 2008. Com `--depth=1` (*shallow clone*), baixamos apenas a fotografia exata do Android 16 atual, economizando mais de 80 GB de rede e disco.

### 4.3 O Segredo do Projeto: `.repo/local_manifests` (Slide 12)
> **Atenção:** Este é o ponto mais importante do Bloco 3!

Se você baixar o AOSP Android 16 puro dos servidores da Google (`android.googlesource.com`), ele só sabe gerar imagens para os celulares **Google Pixel** e para o emulador **Cuttlefish**. Ele **não faz ideia** do que é uma Raspberry Pi 5.

Para ensinar o AOSP a compilar para a Raspberry Pi 5 sem alterar o arquivo original da Google, criamos a pasta `.repo/local_manifests/` e clonamos o manifesto de hardware comunitário (*Raspberry Vanilla / KonstaKANG* para Android 16).
Quando rodamos `repo sync`, o `repo` junta os repositórios da Google com 3 diretórios cruciais da Raspberry Pi 5:

1. **`device/brcm/rpi5` (Configuração da Placa e Produto):**
   * Contém o `BoardConfig.mk` (diz ao compilador que a CPU é um ARM64 Cortex-A76, define o tamanho de cada partição em megabytes).
   * Contém o `fstab.rpi5` (*File System Table*: ensina o Android quais partições do cartão SD são `/system`, `/vendor` e `/data`).
   * Contém o `aosp_rpi5_car.mk` (o arquivo que define nosso produto automotivo).
2. **`kernel/brcm/rpi` (Kernel Linux 6.x + Device Trees):**
   * No mundo ARM (diferente do PC x86 que tem BIOS/ACPI para descobrir dispositivos sozinho), o Kernel Linux precisa de um arquivo chamado **Device Tree Blob (`.dtb`)** — no nosso caso, `bcm2712-rpi-5-b.dtb` — que descreve em quais endereços de memória física estão o controlador USB, o PCIe, a GPU, o SPI e o chip de I/O **RP1** da Raspberry Pi 5.
3. **`external/mesa3d` (Aceleração Gráfica Open-Source):**
   * Fornece os drivers gráficos `v3d` e `vc4` compilados com `meson` e `ninja` para que a GPU **VideoCore VII** da Raspberry Pi 5 desenhe a interface do Android 16 a 60 quadros por segundo com suporte a OpenGL ES 3.1 e Vulkan.

### 4.4 Entendendo o `lunch aosp_rpi5_car-bp1a-userdebug` (Slide 13)
Quando rodamos `source build/envsetup.sh`, carregamos funções especiais no terminal do Linux. Uma delas é o comando **`lunch`**, que configura todas as variáveis de ambiente para um alvo específico.
Vamos dissecar o nome **`aosp_rpi5_car-bp1a-userdebug`**:
* **`aosp_rpi5_car`**: É o nome do produto definido em `device/brcm/rpi5/aosp_rpi5_car.mk`. Dentro desse arquivo existe a linha mágica:
  ```makefile
  $(call inherit-product, packages/services/Car/car_product/build/car.mk)
  ```
  Essa linha importa todos os pacotes automotivos (`CarService`, `CarSystemUI`, `CarLauncher`, `VHAL`) para dentro da imagem da Raspberry Pi 5!
* **`bp1a`**: É o código da revisão de plataforma do Android 16.
* **`userdebug`**: O Android possui 3 variantes de build:
  1. `user`: Usado no carro vendido ao consumidor final. É totalmente trancado (sem acesso root, sem `adb` avançado).
  2. `userdebug` *(Nossa escolha)*: Idêntico ao `user` em performance e segurança (o SELinux continua ligado em modo *enforcing*), mas permite rodar `adb root` e usar a ferramenta `cmd car_service` para injetar eventos de teste no carro.
  3. `eng`: Build de desenvolvimento rápido com várias verificações desligadas.

### 4.5 Compilação (`m`) e Empacotamento (`rpi5-mkimg.sh`) (Slides 14 e 15)
* **O que é o Soong e o Ninja?** Antigamente o Android usava o lento `GNU Make` (`Android.mk`). Hoje ele usa o **Soong**, que lê arquivos declarativos `Android.bp` (*Blueprint*) muito mais rápidos e gera um arquivo de instruções para o executor ultrarrápido **Ninja**.
* **Por que precisamos rodar `./rpi5-mkimg.sh` depois do build?**
  * Quando o comando `m -j$(nproc)` termina, ele gera arquivos separados na pasta `out/target/product/rpi5/`: `boot.img`, `system.img`, `vendor.img` e `userdata.img`.
  * Em um celular comum, você conectaria o cabo USB e usaria o comando `fastboot flash system system.img`.
  * Porém, a **Raspberry Pi 5 não nasce em modo fastboot**. O chip de boot (SPI EEPROM) da Raspberry Pi 5 exige um cartão MicroSD ou SSD NVMe que tenha uma tabela de partições padrão onde a **Partição 1 seja obrigatoriamente FAT32** contendo o arquivo `config.txt` e o Kernel.
  * O script `rpi5-mkimg.sh` cria um arquivo de imagem vazio, usa o `parted` para criar 4 partições (`p1: boot FAT32`, `p2: system EXT4`, `p3: vendor EXT4`, `p4: userdata EXT4`) e copia cada `.img` gerado pelo AOSP para dentro da sua respectiva partição, entregando um único arquivo `.img` pronto para ser gravado no cartão SD via **BalenaEtcher** ou `dd`.

---

## 5. Bloco 4: Sequência de Boot, Debugging e Desafios de Bancada (Slides 16 a 18)

### 5.1 A Cadeia de Boot Completa na Raspberry Pi 5 (Slide 16)
Se alguém perguntar *"O que acontece desde o momento em que você pluga o cabo USB-C na Raspberry Pi 5 até abrir a tela do carro?"*, siga estes 5 passos:
1. **Hardware / SPI EEPROM Bootloader:** A memória EEPROM soldada na placa da RPi 5 inicializa a CPU/RAM, acessa o cartão MicroSD, abre a Partição 1 (`FAT32`) e lê o arquivo de texto `config.txt`.
2. **Kernel Linux 6.x + Device Tree:** O bootloader carrega o Kernel compactado (`Image.gz`), aplica a árvore de dispositivos da placa (`bcm2712-rpi-5-b.dtb`) e o overlay de vídeo (`vc4-kms-v3d-pi5.dtbo`), e entrega o controle para o Kernel.
3. **Android `init` (First Stage):** O Kernel monta o `ramdisk.img` em memória e executa o processo `/init` (PID 1). Nessa primeira fase, o `init` lê o arquivo `/vendor/etc/fstab.rpi5` e monta as partições `/system` (`mmcblk0p2`), `/vendor` (`mmcblk0p3`) e `/data` (`mmcblk0p4`).
4. **Android `init` (Second Stage) & HALs:** O `init` carrega as regras de segurança do **SELinux**, lê os scripts `.rc` e inicia os processos nativos em C++ da camada `/vendor` — incluindo o nosso **VHAL (`vendor.vehicle-hal-default`)** e o **`Zygote64`** (o processo pai de todos os aplicativos Java/Kotlin).
5. **SystemServer & CarService:** O `Zygote64` dá origem ao `SystemServer`, que inicia o **`CarService`**. O `CarService` se conecta ao `VHAL` via Binder AIDL, lê o estado inicial do veículo e finalmente abre na tela o **`CarLauncher`** e a barra de **`CarSystemUI`** (com os controles de ar-condicionado).

### 5.2 Como funciona a Injeção de Eventos via ADB (Slide 17)
Durante o desenvolvimento, precisamos testar como o nosso sistema se comporta a 100 km/h ou quando o ar-condicionado falha, sem precisar dirigir um carro de verdade a 100 km/h com um notebook no colo!
Com a Raspberry Pi 5 conectada na mesma rede Wi-Fi ou Ethernet:
* Conectamos via `adb connect <IP>:5555`.
* Usamos o utilitário interno do `CarService`:
  ```bash
  adb shell cmd car_service inject-vhal-event 0x11600207 18.0
  ```
* **O que esse comando faz por baixo dos panos?** Ele faz o `CarPropertyService` fingir que acabou de receber um pacote do `VHAL` dizendo que a propriedade `0x11600207` (`PERF_VEHICLE_SPEED`) agora vale `18.0` metros por segundo ($18{,}0 \times 3{,}6 = 64{,}8\text{ km/h}$). Todos os aplicativos e o gerenciador de segurança (`CarUxRestrictions`) reagem instantaneamente na tela!

### 5.3 Os 3 Desafios Reais Superados na Bancada (Slide 18)
Estes pontos mostram maturidade prática da sua equipe com hardware embarcado:
1. **Fonte de Energia de 5.1V / 5A (27W):**
   * Carregadores de celular comuns fornecem 5V / 2A ou 3A (ou só entregam mais potência se negociarem 9V/12V/20V, o que a RPi 5 não aceita — ela precisa de 5V!).
   * No momento do boot do Android 16, os 4 núcleos Cortex-A76 e a GPU VideoCore VII vão a 100% de uso simultaneamente. Com uma fonte comum de 3A, a tensão cai abaixo de 4.63V (*brownout*), travando o cartão SD ou causando reinicialização em loop. A fonte oficial USB-C PD com perfil **5.1V / 5A** resolve isso completamente.
2. **Configuração de Vídeo KMS (`config.txt`):**
   * Muitas telas touchscreen automotivas baratas não enviam corretamente seus dados de resolução (tabela **EDID**) pelo cabo HDMI. Ajustar os parâmetros de KMS e `dtoverlay=vc4-kms-v3d-pi5` no arquivo `config.txt` da partição de boot garante que o compositor gráfico do Android (`drm_hwcomposer`) abra na resolução correta.
3. **Velocidade de I/O do Armazenamento (MicroSD Classe A2 vs. Comum):**
   * No primeiro boot, o Android 16 compila e otimiza os aplicativos em background (`dex2oat`) e cria dezenas de bancos de dados SQLite na partição `/data`. Cartões SD comuns têm péssima performance de escrita aleatória (IOPS baixo), deixando a interface lenta. Usar cartões **MicroSD Classe A2** (projetados para IOPS de aplicativos) ou um **SSD NVMe** via adaptador PCIe transforma a fluidez do sistema.

---

## 6. Bloco 5: Roadmap de Evolução e Fechamento (Slides 19 a 20)

Para fechar a apresentação mostrando para onde o produto vai evoluir:

1. **Integração com Barramento CAN Real (`SocketCAN` + `MCP2515`):**
   * Os carros reais usam o protocolo **CAN Bus** (um par de fios trançados `CAN_H` e `CAN_L` onde todas as unidades eletrônicas conversam).
   * Na Raspberry Pi 5, podemos ligar uma plaquinha **MCP2515** nos pinos **SPI** (ou usar um adaptador USB-CAN). O Kernel Linux possui um subsistema nativo excelente chamado **SocketCAN**, que faz o barramento CAN aparecer no Linux como se fosse uma placa de rede comum chamada `can0`! Assim, nosso código C++ no VHAL só abre um socket padrão Linux para ler a velocidade e o RPM reais do motor.
2. **Customização Visual OEM com RRO (*Runtime Resource Overlays*):**
   * Como mudar todas as cores, fontes, ícones e formato dos botões do Android 16 para ter a cara da nossa empresa sem alterar uma única linha do código original da Google?
   * Através de **RROs**. Um RRO é um APK especial que não tem código Java, apenas arquivos de recursos XML/imagens (`res/`). Quando instalado na partição `/vendor/overlay/`, o Android substitui visualmente o tema do `CarSystemUI` e do `CarLauncher` em tempo de execução. Se no ano que vem atualizarmos para o Android 17, nossa customização continua funcionando sem conflitos de `git merge`!
3. **Dual-Display (Central Multimídia + Painel de Instrumentos Digital):**
   * A Raspberry Pi 5 possui **duas saídas Micro-HDMI**. O AAOS suporta nativamente múltiplos displays: a saída `HDMI 0` roda a tela central touch (**IVI**), enquanto a saída `HDMI 1` roda o **`ClusterHome`** (o painel de instrumentos atrás do volante mostrando velocímetro, marcha e mapa em tela cheia).

---

## 7. Glossário Rápido de Siglas Técnicas (Vocabulário de Bolso)

Use esta tabela para revisão rápida 10 minutos antes de começar a apresentar:

| Sigla | Nome Completo | O que significa em 1 frase |
| :--- | :--- | :--- |
| **AOSP** | *Android Open Source Project* | O código-fonte aberto do sistema operacional Android mantido pela Google. |
| **AAOS** | *Android Automotive Operating System* | A vertente do AOSP projetada para rodar nativamente dentro do computador de bordo dos veículos. |
| **GAS** | *Google Automotive Services* | O pacote comercial fechado da Google (Maps, Play Store, Assistant) que montadoras licenciam para instalar sobre o AAOS. |
| **IVI** | *In-Vehicle Infotainment* | A central multimídia do veículo (tela central de entretenimento, navegação e controle de conforto). |
| **VHAL** | *Vehicle Hardware Abstraction Layer* | O serviço nativo em C++ que serve de tradutor entre o Android (`CarService`) e a rede física do carro (CAN Bus). |
| **AIDL** | *Android Interface Definition Language* | Linguagem de contrato usada no Android 16 para comunicação IPC entre o Framework Java e os serviços nativos C++ (como o VHAL). |
| **BSP** | *Board Support Package* | Conjunto de códigos específicos de uma placa de hardware (Kernel, drivers, Device Tree e configurações de partição da RPi 5). |
| **DTB / DTBO** | *Device Tree Blob / Overlay* | Arquivo binário lido pelo Kernel Linux em processadores ARM para saber quais componentes físicos existem na placa Raspberry Pi 5. |
| **KMS / DRM** | *Kernel Mode Setting / Direct Rendering Manager* | Subsistemas do Kernel Linux responsáveis por gerenciar a placa de vídeo e enviar a imagem para a porta HDMI. |
| **HSUM** | *Headless System User Mode* | Modo obrigatório no AAOS onde o usuário `0` do sistema roda oculto em background e os motoristas usam perfis a partir do usuário `10`. |
| **RRO** | *Runtime Resource Overlay* | Mecanismo oficial do Android para trocar o visual (cores, layouts, ícones) de aplicativos do sistema sem modificar o código-fonte original. |
| **CPMS** | *CarPowerManagementService* | Serviço do AAOS que controla o ciclo de energia do veículo, incluindo o *Deep Sleep* (Suspend-to-RAM) e o *Garage Mode*. |
| **EVS** | *Exterior View System* | Subsistema de câmeras de baixa latência do AAOS projetado para exibir a câmera de ré em menos de 2 segundos após ligar o carro. |
| **Soong / Ninja** | *Android Build System* | Ferramentas modernas que leem os arquivos `Android.bp` e executam a compilação paralela massiva do código-fonte do AOSP. |
