# Android Automotive OS: An Overview (Android 16 + Raspberry Pi 5)

Apresentação técnica detalhada sobre a arquitetura do **Android Automotive OS (AAOS)** e o processo de compilação (*bring-up*) do **AOSP Android 16** direcionado para a placa **Raspberry Pi 5 (ARM64)**.

Baseado no procedimento prático documentado em:
- [Build AOSP Android 16 Automotive OS for Raspberry Pi 5 and flash it (Medium - Make Android)](https://medium.com/make-android/build-aosp-android-16-automotive-os-for-raspberry-pi-5-and-flash-it-cc24efd24410)

---

## 📁 Conteúdo do Repositório

1. **[`Android_Automotive_Overview_RPi5_Clean_Light.pptx`](./Android_Automotive_Overview_RPi5_Clean_Light.pptx)**
   - Apresentação em PowerPoint (formato 16:9 widescreen) com **estilo limpo e fundo claro**, ideal para aplicação direta do slide mestre/tema corporativo.
   - Contém 20 slides técnicos estruturados para 45 minutos com notas de orador integradas.

2. **[`Android_Automotive_Overview_RPi5.pptx`](./Android_Automotive_Overview_RPi5.pptx)**
   - Apresentação em PowerPoint com **estilo dark automotivo** de alto contraste.

3. **[`index.html`](./index.html)**
   - Deck de slides interativo para navegador Web com:
     - Cronômetro de 45 minutos acoplado.
     - Painel lateral de notas do apresentador (tecla `N`).
     - Simulador interativo de injeção de eventos no VHAL (`PERF_VEHICLE_SPEED` -> `CarUxRestrictions`).

4. **[`presentation_marp.md`](./presentation_marp.md)**
   - Apresentação em formato Marp Markdown para exportação rápida via CLI ou extensão do VS Code/Cursor.

5. **[`instructions.md`](./instructions.md)**
   - Planejamento slide a slide, controle de tempo, referências técnicas, checklist de bancada, cheat sheet de comandos ADB/VHAL para Live Demo e guia de Q&A técnico.

---

## 🎯 Estrutura da Apresentação (45 minutos)

| Bloco | Conteúdo | Duração |
| :--- | :--- | :---: |
| **Bloco 1** | Introdução Técnica: Ecossistema AAOS & Stack Automotivo | 06 min |
| **Bloco 2** | Deep Dive na Arquitetura Interna do AAOS (`CarService` & `VHAL`) | 12 min |
| **Bloco 3** | Engenharia do Build: AOSP Android 16 para Raspberry Pi 5 | 15 min |
| **Bloco 4** | Demo, Debugging & Desafios de Bring-up na RPi 5 | 07 min |
| **Bloco 5** | Roadmap de Engenharia & Q&A Técnico | 05 min |
