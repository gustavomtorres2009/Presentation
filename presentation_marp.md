---
marp: true
theme: default
paginate: true
header: 'Android Automotive OS (AAOS) • Android 16 + Raspberry Pi 5'
footer: 'Target: aosp_rpi5_car-bp1a-userdebug | Hardware: BCM2712 ARM64'
style: |
  section {
    font-family: 'Segoe UI', Calibri, sans-serif;
    color: #1f2937;
    background-color: #ffffff;
  }
  h1 {
    color: #111827;
  }
  h2 {
    color: #1d4ed8;
    border-bottom: 2px solid #e5e7eb;
    padding-bottom: 8px;
  }
  pre {
    background-color: #f1f5f9;
    color: #0f172a;
    border-left: 4px solid #1d4ed8;
  }
  code {
    font-family: 'Consolas', monospace;
  }
  table {
    font-size: 0.85rem;
  }
  th {
    background-color: #e2e8f0;
  }
---

<!-- _class: lead -->
# Android Automotive OS (AAOS)
### Technical Overview & Hardware Bring-up
**AOSP Android 16 compilado nativamente na Raspberry Pi 5 (ARM64)**

*Apresentação Técnica (45 minutos)*

---

## Agenda Executiva (45 min)

- **Bloco 1 (06 min):** Introdução Técnica: Ecossistema AAOS & Stack Automotivo
- **Bloco 2 (12 min):** Deep Dive na Arquitetura Interna do AAOS (`CarService` & `VHAL`)
- **Bloco 3 (15 min):** Engenharia do Build: AOSP Android 16 para Raspberry Pi 5
- **Bloco 4 (07 min):** Demo, Debugging & Desafios de Bring-up na bancada
- **Bloco 5 (05 min):** Roadmap Técnico de Evolução do Produto & Q&A

---

## Bloco 1: Android Auto vs. AAOS vs. GAS

| Critério Técnico | Android Auto (Projeção) | Android Automotive OS (AAOS) | GAS (Google Automotive Services) |
| :--- | :--- | :--- | :--- |
| **Onde roda?** | Smartphone (Head-unit apenas renderiza stream H.264) | **Nativo no SoC do veículo** (ex.: RPi 5 BCM2712) | Roda sobre o AAOS nativo |
| **Acesso ao Carro** | Quase nulo | **Total via VHAL:** HVAC, marchas, bateria, áudio, câmeras | Total via APIs do AAOS + apps Google |
| **Código & Licença**| Fechado (APK móvel) | **100% Open Source (AOSP)** | Proprietário (Licenciado Google) |

> **Nosso Escopo:** Construímos nossa plataforma sobre o **AOSP Android 16 AAOS puro**, com total controle de Kernel, SELinux, VHAL e SystemUI.

---

## Bloco 1: Por que Raspberry Pi 5 + Android 16?

### Emulador (`x86_64`) vs. Hardware Real (`arm64-v8a`)

- **Limitações do Emulador:** Roda sobre CPU do host de desenvolvimento com drivers sintéticos (`virtio`). Não valida problemas de clock, termal, I/O real nem Device Trees.
- **Vantagens da Raspberry Pi 5:**
  - **SoC Broadcom BCM2712:** Quad-core Cortex-A76 64-bit a 2.4GHz (mesma família arquitetural dos SoCs automotivos de mercado).
  - **GPU VideoCore VII:** Aceleração gráfica OpenGL ES 3.1 e Vulkan 1.2 via stack Mesa3D (`v3d`) + `drm_hwcomposer`.
  - **Conexões de Bancada:** Header de 40 pinos (SPI/UART para CAN Bus e debug serial) + Dual Micro-HDMI.

---

## Bloco 2: Arquitetura em Camadas do AAOS

1. **System & Car Apps (`packages/apps/Car/*`):**
   - `CarLauncher`, `CarSystemUI` (Barra de Ar-Condicionado), `CarSettings`, `CarMediaApp`.
2. **Car Framework & Services (`packages/services/Car`):**
   - `CarPropertyService`, `CarAudioService`, `CarPowerManagementService`, `CarUxRestrictionsService`.
3. **Treble HAL Layer em Stable AIDL (`hardware/interfaces/automotive`):**
   - `IVehicle.aidl` (VHAL), `AudioControl HAL`, `EVS HAL` (câmeras).
4. **Linux Kernel 6.x & BSP Broadcom (`kernel/brcm/rpi` + `device/brcm/rpi5`):**
   - Mesa3D V3D, ALSA Drivers, SocketCAN e Device Tree (`bcm2712-rpi-5-b.dtb`).

---

## Bloco 2: O Modelo de Propriedades (`CarPropertyManager`)

Todo sinal ou atuador do veículo é mapeado como uma propriedade tipada no AIDL:

- `PERF_VEHICLE_SPEED` (`0x11600207`): Velocidade atual (`FLOAT`, `CONTINUOUS`).
- `HVAC_TEMPERATURE_SET` (`0x15600503`): Ajuste de ar-condicionado por zona de assento.
- `GEAR_SELECTION` (`0x11400400`): Posição da alavanca de câmbio (`PARK`, `DRIVE`, etc.).

```java
Car car = Car.createCar(context);
CarPropertyManager propMgr = (CarPropertyManager) car.getCarManager(Car.PROPERTY_SERVICE);
propMgr.registerCallback(new CarPropertyEventCallback() {
    @Override
    public void onChangeEvent(CarPropertyValue val) {
        float speed = (Float) val.getValue();
        updateSpeedometer(speed);
    }
}, VehiclePropertyIds.PERF_VEHICLE_SPEED, CarPropertyManager.SENSOR_RATE_UI);
```

---

## Bloco 2: VHAL (Vehicle HAL) em Stable AIDL

- **Substituição do HIDL:** A partir das versões recentes e consolidado no Android 16, o VHAL utiliza **Stable AIDL Binder** (`IVehicle.aidl`).
- **Principais Operações:**
  - `getAllPropConfigs()`: Publica a lista de propriedades suportadas no boot.
  - `getValues()` / `setValues()`: Leitura e escrita de propriedades no veículo.
  - `subscribe()`: Registro de callbacks para envio assíncrono de eventos de sensores.
- **Reference VHAL na RPi 5:** O processo `vendor.vehicle-hal-default` inicializa com configurações JSON (`DefaultConfig.json`), permitindo plugar um backend C++ conectado ao `SocketCAN`.

---

## Bloco 2: Áudio Multizona, Energia e Segurança de UX

- **CarAudioService:** Áudio multizona (`car_audio_configuration.xml`) para separar saída do motorista da saída dos passageiros, com regras estritas de prioridade e *audio ducking*.
- **CarPowerManagementService:** Estados de suspensão na memória RAM (*Suspend-to-RAM / Deep Sleep*) e janela de manutenção silenciosa (*Garage Mode*).
- **CarUxRestrictions (Driver Distraction):**
  - Monitora o estado de condução (`PARKED` $\rightarrow$ `MOVING`).
  - Bloqueia digitação em teclado, streaming de vídeo e telas de configurações complexas com o carro em movimento.
- **Headless System User Mode (HSUM):** `User 0` roda oculto apenas para processos de sistema; o motorista atua sempre como usuário secundário (`User 10+`).

---

## Bloco 3: Pipeline de Build (AOSP Android 16 $\rightarrow$ RPi 5)

```text
[1. Host Ubuntu] ──> [2. repo init] ──> [3. local_manifests] ──> [4. repo sync]
       │
       └──> [5. lunch aosp_rpi5_car] ──> [6. m -j$(nproc)] ──> [7. rpi5-mkimg.sh]
```

- **Requisitos Críticos da Máquina de Build:**
  - **SO:** Ubuntu 22.04 LTS (x86_64).
  - **RAM:** 64 GB recomendados (mínimo 32 GB + 32 GB Swap).
  - **Storage:** +500 GB NVMe SSD livre.
  - **Ferramentas específicas:** `meson`, `ninja-build`, `kpartx`, `parted`, `dosfstools`, `mtools`.

---

## Bloco 3: O Segredo do `.repo/local_manifests`

O manifesto padrão do Google não possui os arquivos de placa da Raspberry Pi 5. Injetamos o manifesto comunitário:

```bash
git clone https://github.com/raspberry-vanilla/android_local_manifest \
  .repo/local_manifests -b android-16.0
repo sync -c --no-clone-bundle --no-tags -j$(nproc)
```

- `device/brcm/rpi5`: Makefiles (`aosp_rpi5_car.mk`), tabelas de partição (`fstab.rpi5`) e regras SELinux.
- `kernel/brcm/rpi`: Kernel Linux 6.x adaptado ao SoC BCM2712 e overlay gráfico `vc4-kms-v3d-pi5.dtbo`.
- `external/mesa3d`: Drivers gráficos aceleração de hardware Broadcom V3D.

---

## Bloco 3: Configuração do Target e Compilação

```bash
# 1. Carregar variáveis de ambiente de build
source build/envsetup.sh

# 2. Selecionar o target automotivo da RPi 5 (Release bp1a em userdebug)
lunch aosp_rpi5_car-bp1a-userdebug

# 3. Disparar a compilação paralela com Soong e Ninja
m -j$(nproc)
```

- O makefile herda `packages/services/Car/car_product/build/car.mk`.
- A variante `userdebug` mantém o SELinux ativo (*enforcing*), mas disponibiliza `adb root` e injeção de eventos para depuração na bancada.

---

## Bloco 3: Particionamento e Gravação no Disco

Ao término do build, executamos o empacotamento das partições:

```bash
./rpi5-mkimg.sh
# Imagem gerada: out/target/product/rpi5/aosp_rpi5_car-android16.img
```

| Partição | Posição no Bloco | Filesystem | Conteúdo |
| :--- | :--- | :--- | :--- |
| **p1 (`boot`)** | `/dev/block/mmcblk0p1` | `FAT32` | `config.txt`, `Image.gz`, `ramdisk.img`, Device Tree |
| **p2 (`system`)**| `/dev/block/mmcblk0p2` | `EXT4` | Android 16 OS, `CarService`, `CarLauncher` |
| **p3 (`vendor`)**| `/dev/block/mmcblk0p3` | `EXT4` | Drivers Broadcom, Mesa3D, binário do VHAL |
| **p4 (`userdata`)**| `/dev/block/mmcblk0p4`| `EXT4` | Perfis multi-usuário (`User 10+`), cache e bancos |

---

## Bloco 4: Boot Flow e Demonstração do Produto

### Cadeia de Inicialização Física:
1. **EEPROM Bootloader:** Lê o arquivo `config.txt` na partição FAT32 primária.
2. **Kernel 6.x:** Carrega o overlay `vc4-kms-v3d-pi5.dtbo` e monta o `ramdisk.img`.
3. **Android Init:** Monta `/system`, `/vendor` e `/data` conforme o `fstab.rpi5`.
4. **Zygote64 & SystemServer:** Sobe os serviços essenciais e dispara o `CarService`.
5. **CarLauncher:** Apresenta a interface gráfica automotiva com dock inferior de navegação e controle de clima (HVAC).

---

## Bloco 4: Injeção de Eventos VHAL via ADB

Comandos executados ao vivo via terminal para inspecionar e testar o sistema:

```bash
# Conectar via ADB pela rede
adb connect 192.168.1.50:5555

# Injetar veículo em movimento (0x11600207 = PERF_VEHICLE_SPEED a 18.0 m/s = ~65 km/h)
adb shell cmd car_service inject-vhal-event 0x11600207 18.0

# O CarUxRestrictions entra em ação imediatamente:
# -> Bloqueia teclado virtual e menus complexos de configuração na tela!

# Voltar para Estacionado (0.0 m/s)
adb shell cmd car_service inject-vhal-event 0x11600207 0.0
```

---

## Bloco 4: Lições Aprendidas no Bring-up da RPi 5

1. **Integridade de Alimentação Elétrica:**
   - Durante a inicialização pesada da GPU e dos módulos de sistema, fontes de 5V/3A provocam *under-voltage*. Adoção obrigatória da fonte oficial **USB-C PD 5.1V / 5A (27W)**.
2. **Pipeline Gráfico e Resoluções no `config.txt`:**
   - Telas e monitores ultrawide sem EDID padrão precisam de ajuste explícito de modo KMS no arquivo `config.txt` da partição de boot.
3. **Gargalo de I/O em Cartões SD:**
   - O primeiro boot do Android 16 compila dezenas de APKs via `dex2oat`. Cartões MicroSD comuns sofrem travamentos; o uso de **MicroSD Classe A2** ou **SSD NVMe via HAT** é essencial.

---

## Bloco 5: Próximos Passos de Engenharia

- **1. Integração de Barramento CAN Físico (`SocketCAN` $\leftrightarrow$ `VHAL`):**
  - Instalação de transceptor CAN MCP2515 (SPI) e mapeamento de frames OBD-II/J1939 diretamente para propriedades do Android.
- **2. Customização Visual via RROs (*Runtime Resource Overlays*):**
  - Alteração da identidade visual do `CarSystemUI` e `CarLauncher` preservando compatibilidade com futuras versões do AOSP.
- **3. Dual-Display (Cluster Digital):**
  - Utilização da segunda porta Micro-HDMI da Raspberry Pi 5 para acionar o painel de instrumentos digital do motorista (`ClusterHome`).

---

<!-- _class: lead -->
# Conclusão & Perguntas (Q&A)

- **Domínio Completo do Stack AOSP Android 16**
- **Pipeline de Engenharia e Build 100% Reprodutível**
- **Plataforma Física em Hardware ARM64 Pronta para Prototipação Real**

*Obrigado! Espaço aberto para discussão técnica.*
