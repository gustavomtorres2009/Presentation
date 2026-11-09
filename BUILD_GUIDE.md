# Guia Técnico de Replicação: AOSP Android 16 Automotive OS na Raspberry Pi 5

Este guia consolida todos os comandos de terminal, configurações de manifesto e etapas de diagnóstico necessários para reproduzir a compilação e gravação do **Android 16 Automotive OS (AAOS)** para a **Raspberry Pi 5 (ARM64)**.

> **Referência Base:** [Build AOSP Android 16 Automotive OS for Raspberry Pi 5 and flash it (Make Android - Medium)](https://medium.com/make-android/build-aosp-android-16-automotive-os-for-raspberry-pi-5-and-flash-it-cc24efd24410)

---

## 1. Pré-Requisitos de Hardware e Ambiente

### 1.1 Máquina de Compilação (Build Host)
- **Sistema Operacional:** Ubuntu 22.04.5 LTS (`x86_64`)
- **Processador:** CPU Multi-Core de alta densidade (ex.: AMD Ryzen 9 / Threadripper, 16+ threads)
- **Memória RAM:** 64 GB recomendados (mínimo de 32 GB físicos + 32 GB de arquivo Swap)
- **Armazenamento:** +500 GB livres em SSD NVMe (~180 GB para o repositório `.repo` e código-fonte + ~250 GB para o diretório `out/`)

### 1.2 Hardware Alvo (Bancada do Produto)
- **Single-Board Computer:** Raspberry Pi 5 Model B (4 GB ou 8 GB RAM)
- **Armazenamento:** Cartão MicroSD de 64 GB Classe A2 (com leitor USB 3.0) ou SSD NVMe via HAT PCIe
- **Alimentação Elétrica:** Fonte oficial USB-C PD de **5.1V / 5A (27W)**
- **Periféricos:** Cabo Micro-HDMI para HDMI, Monitor/Display Touchscreen USB, Mouse e Teclado USB

---

## 2. Preparação do Host Ubuntu 22.04 LTS

### 2.1 Instalação de Pacotes e Dependências de Build
Inclui as ferramentas padrão do AOSP, o **OpenJDK 17**, o sistema de build **Meson/Ninja** (exigido pelo driver gráfico Mesa3D da RPi 5) e os utilitários de particionamento (`kpartx`, `parted`, `dosfstools`, `mtools`):

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git-core gnupg flex bison build-essential zip curl \
  zlib1g-dev libc6-dev-i386 libncurses5 lib32ncurses-dev x11proto-core-dev \
  libx11-dev libgl1-mesa-dev libxml2-utils xsltproc unzip fontconfig \
  python3 python-is-python3 openjdk-17-jdk libssl-dev rsync \
  meson ninja-build kpartx parted dosfstools mtools
```

### 2.2 Instalação do Utilitário `repo` da Google
```bash
mkdir -p ~/.bin
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/.bin/repo
chmod a+rx ~/.bin/repo
export PATH="${HOME}/.bin:${PATH}"
echo 'export PATH="${HOME}/.bin:${PATH}"' >> ~/.bashrc
```

### 2.3 Configuração de Swap Adicional (Caso o Host possua 32 GB de RAM)
Para evitar que o *OOM Killer* encerre processos do `clang++`, `rustc` ou `metalava` durante a linkagem:
```bash
sudo fallocate -l 32G /swapfile_aosp
sudo chmod 600 /swapfile_aosp
sudo mkswap /swapfile_aosp
sudo swapon /swapfile_aosp
```

---

## 3. Download do Código-Fonte AOSP Android 16 e BSP da RPi 5

### 3.1 Inicialização do Workspace AOSP
```bash
mkdir -p ~/aosp-rpi5-android16
cd ~/aosp-rpi5-android16

# Inicializa o manifesto apontando para a revisão do Android 16
repo init -u https://android.googlesource.com/platform/manifest -b android-16.0.0_r1 --depth=1
```

### 3.2 Adição do `local_manifests` para Raspberry Pi 5
O manifesto oficial do AOSP não possui a *Device Tree*, o Kernel Linux nem os HALs gráficos do SoC Broadcom BCM2712. Adicionamos os repositórios comunitários de hardware em `.repo/local_manifests/`:

```bash
git clone https://github.com/raspberry-vanilla/android_local_manifest .repo/local_manifests -b android-16.0
```

Principais módulos injetados pelo manifesto local:
- `device/brcm/rpi5`: Configurações de placa (`BoardConfig.mk`), `fstab.rpi5`, regras SELinux e o target automotivo `aosp_rpi5_car.mk`.
- `kernel/brcm/rpi`: Kernel Linux 6.x com suporte ao SoC BCM2712, chip de I/O RP1 e overlay KMS `vc4-kms-v3d-pi5.dtbo`.
- `external/mesa3d`: Stack gráfico OpenGL ES 3.1 / Vulkan 1.2 para a GPU VideoCore VII.

### 3.3 Sincronização da Árvore de Código (`repo sync`)
```bash
repo sync -c --no-clone-bundle --no-tags -j$(nproc)
```

---

## 4. Configuração do Target Automotivo e Compilação

### 4.1 Inicialização do Ambiente e Seleção do Target (`lunch`)
```bash
source build/envsetup.sh
lunch aosp_rpi5_car-bp1a-userdebug
```

- **`aosp_rpi5_car`**: Seleciona o hardware `rpi5` herdando a pilha completa do Android Automotive OS (`packages/services/Car/car_product/build/car.mk`).
- **`bp1a`**: Identificador de build da árvore do Android 16.
- **`userdebug`**: Variante recomendada para engenharia (mantém o SELinux em modo *enforcing*, mas habilita `adb root` e ferramentas de injeção no VHAL).

### 4.2 Execução da Compilação (`m`)
```bash
m -j$(nproc)
```
Ao finalizar, os artefatos de partição serão gerados em `out/target/product/rpi5/`:
- `boot.img` (128 MB, FAT32)
- `system.img` (~2.5 GB, EXT4)
- `vendor.img` (~512 MB, EXT4)
- `userdata.img` (EXT4)

---

## 5. Empacotamento da Imagem Única e Gravação (Flashing)

### 5.1 Unificação das Partições em Arquivo `.img`
Como a EEPROM da Raspberry Pi 5 exige um disco particionado iniciando com uma partição primária FAT32, executamos o script de montagem da imagem:

```bash
./rpi5-mkimg.sh
```
O arquivo final unificado será salvo em `out/target/product/rpi5/` (ex.: `aosp_rpi5_car-android16.img`).

### 5.2 Gravação no Cartão MicroSD ou NVMe
Você pode gravar o arquivo `.img` utilizando a interface gráfica do **BalenaEtcher**, **Raspberry Pi Imager** (*Use Custom*) ou diretamente via `dd` no terminal Linux:

```bash
# Substitua /dev/sdX pelo identificador correto do seu cartão MicroSD/NVMe (verifique com lsblk)
sudo dd if=out/target/product/rpi5/aosp_rpi5_car-android16.img of=/dev/sdX bs=4M status=progress conv=fsync
sync
```

---

## 6. Primeiro Boot e Diagnóstico via ADB / VHAL

1. Insira o cartão MicroSD na Raspberry Pi 5.
2. Conecte o cabo Micro-HDMI na porta **HDMI 0** (mais próxima do conector USB-C de energia).
3. Conecte o cabo de rede Ethernet (ou configure o Wi-Fi na interface) e ligue a fonte de **5.1V / 5A**.

### Comandos Úteis de Verificação e Simulação Automotiva
```bash
# 1. Conectar via ADB pela rede local
adb connect <IP_DA_RPI5>:5555
adb root

# 2. Inspecionar serviços registrados pelo CarService
adb shell dumpsys car_service

# 3. Listar todas as propriedades suportadas pelo VHAL (Stable AIDL)
adb shell dumpsys android.hardware.automotive.vehicle.IVehicle/default --list

# 4. Simular veículo em movimento (Marcha Drive = 8, Velocidade = 18.0 m/s = ~65 km/h)
adb shell cmd car_service inject-vhal-event 0x11400400 8
adb shell cmd car_service inject-vhal-event 0x11600207 18.0

# 5. Simular veículo estacionado novamente (Marcha Park = 4, Velocidade = 0.0 m/s)
adb shell cmd car_service inject-vhal-event 0x11600207 0.0
adb shell cmd car_service inject-vhal-event 0x11400400 4
```
