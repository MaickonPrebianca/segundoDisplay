# 🖥️ Fix: Bug de Tela Preta em Adaptadores DP→HDMI Passivos no Pop!_OS (AMD RX 5500 / Navi)

> **Status:** ✅ RESOLVIDO, DINÂMICO E ESTÁVEL  
> **Data de Diagnóstico e Validação:** 08 de Agosto de 2026  
> **Sistema Operacional:** Pop!_OS 22.04 LTS (Kernel 7.0.x / GNOME / X11)  
> **GPU:** AMD Radeon RX 5500 / RX 5500M (Arquitetura Navi 14 - Driver open-source `amdgpu` / Mesa)

---

## 📋 Sumário
- [Descrição do Problema](#-descrição-do-problema)
- [Ambiente de Hardware e Software](#-ambiente-de-hardware-e-software)
- [Diagnóstico Técnico e Causa Raiz](#-diagnóstico-técnico-e-causa-raiz)
- [O Que NÃO Funciona (Abordagens a Evitar)](#-o-que-não-funciona-abordagens-a-evitar)
- [Solução Definitiva (Script Inteligente Pós-Login)](#-solução-definitiva-script-inteligente-pós-login)
  - [Passo 1: Script de Ativação Dinâmica](#passo-1-script-de-ativação-dinâmica)
  - [Passo 2: Permissão Sudo sem Senha (Sudoers)](#passo-2-permissão-sudo-sem-senha-sudoers)
  - [Passo 3: Autostart no GNOME](#passo-3-autostart-no-gnome)
- [Vantagens do Método](#-vantagens-do-método)
- [Galeria de Adaptadores e Testes](#-galeria-de-adaptadores-e-testes)
- [Licença](#-licença)

---

## 🔍 Descrição do Problema

Em setups dual-monitor utilizando placas de vídeo AMD da série RX 5000 (Arquitetura RDNA / Navi 14) no Linux, a saída **DisplayPort** conectada a um monitor secundário via **adaptador passivo DP→HDMI** apresenta tela preta contínua após o boot.

- **Sintoma:** O monitor secundário funciona perfeitamente na tela da BIOS e no Windows 10 (Dual Boot), comprovando a integridade física do hardware. Porém, no Linux, a porta é marcada como `disconnected` pelo driver `amdgpu`.
- **Tentativas via Boot (kernelstub):** Forçar o conector na inicialização via linha de comando do kernel (`video=DP-1:...`) causa instabilidade no GDM (Display Manager), gerando telas cinzas e engasgos no login.

---

## 💻 Ambiente de Hardware e Software

| Componente | Especificação / Configuração |
| :--- | :--- |
| **Sistema Operacional** | Pop!_OS 22.04 LTS (Kernel 7.0.11 / X11 / GDM3 / GNOME) |
| **Placa de Vídeo (GPU)** | AMD Radeon RX 5500 / RX 5500M (Navi 14) |
| **Monitor Principal** | LG Ultrawide 2560x1080 (Conectado via HDMI direta `HDMI-A-0`) |
| **Monitor Secundário** | LG 22MP55 Full HD 1920x1080 (Conectado via DP→HDMI `DisplayPort-0`) |
| **Conector afetado** | `card1-DP-1` (Kernel) / `DisplayPort-0` (xrandr/X11) |

---

## 🔬 Diagnóstico Técnico e Causa Raiz

A análise aprofundada dos barramentos do kernel (`/sys/class/drm/` e `i2c`) revelou:

1. **HPD (Hot Plug Detect) Ausente:** O conector `card1-DP-1` reporta estado `disconnected` mesmo com o cabo conectado e monitor ligado. O sinal elétrico de detecção imediata falha no subsistema AMD DC (*Display Core*).
2. **DDC/EDID 100% Funcional:** O canal DDC de comunicação via barramento I2C permanece perfeito. Comandos como `get-edid -b 3` retornam com sucesso o bloco EDID real de 256 bytes do monitor secundário através do adaptador passivo.
3. **Falso Positivo em Logs:** A mensagem de log no `dmesg`:
   ```text
   [drm] Failed to setup vendor infoframe on connector HDMI-A-1: -22
   ```
   refere-se à porta HDMI principal (que funciona) e trata-se de um aviso não-fatal introduzido na versão 6.16 do kernel.
4. **Causa Raiz:** Regressão/Falha de detecção de dongles passivos no módulo AMD DC (Display Core) para a arquitetura Navi 14 ao negociar sinais TMDS sobre linhas DP++.

---

## 🚫 O Que NÃO Funciona (Abordagens a Evitar)

- ❌ **`amdgpu.dc=0`:** Incompatível com a arquitetura RDNA/Navi. Desativa completamente o subsistema de exibição da GPU, resultando em travamento do boot.
- ❌ **`kernelstub -a "video=DP-1:1920x1080@60e"`:** O parâmetro forçado na cmdline do kernel faz o GDM tentar aplicar resoluções antes da inicialização completa do DRM, causando tela cinza no login.
- ❌ **`echo detect > /sys/class/drm/card1-DP-1/status`:** Repete a busca baseada no HPD, que está morto, mantendo o estado desativado.
- ❌ **Cabos Ativos DP→HDMI 8K/4K:** O chipset do cabo direto ativo não completa o handshake com a BIOS da placa RX 5500, falhando inclusive na BIOS/POST.

---

## 🔧 Solução Definitiva (Script Inteligente Pós-Login)

A solução consiste em delegar a inicialização da porta para a sessão de usuário pós-login. O script força a energização momentânea da porta, checa se há um arquivo **EDID válido** (garantindo que o monitor está conectado) e aplica a configuração de tela com `xrandr`.

### Passo 1: Script de Ativação Dinâmica

Crie o diretório e o arquivo de script:

```bash
mkdir -p ~/scripts
nano ~/scripts/ativar_monitor_dp.sh
```

Cole o seguinte conteúdo:

```bash
#!/bin/bash

# 1. Alimenta brevemente o conector para habilitar a leitura DDC
echo on | sudo tee /sys/class/drm/card1-DP-1/status > /dev/null

# 2. Aguarda a estabilização do barramento I2C
sleep 0.5

# 3. Verifica se existe um bloco EDID válido (se há monitor fisicamente conectado)
if [ -s /sys/class/drm/card1-DP-1/edid ]; then
    # [MONITOR CONECTADO]
    sleep 1.5
    # Liga e posiciona a segunda tela (ajuste a orientação/posição se necessário)
    xrandr --output DisplayPort-0 --auto --left-of HDMI-A-0
else
    # [CABO DESCONECTADO / MONITOR DESLIGADO]
    # Retorna o conector ao estado 'detect' e desliga a saída no X11
    echo detect | sudo tee /sys/class/drm/card1-DP-1/status > /dev/null
    xrandr --output DisplayPort-0 --off
fi
```

Torne o script executável:

```bash
chmod +x ~/scripts/ativar_monitor_dp.sh
```

---

### Passo 2: Permissão Sudo sem Senha (Sudoers)

Para permitir que o script altere o arquivo de status da interface DRM sem solicitar senha do usuário a cada inicialização:

1. Abra o editor do sudoers:
   ```bash
   sudo visudo
   ```
2. Adicione a seguinte linha ao final do arquivo (substitua `seu_usuario` pelo seu nome de usuário no Linux):
   ```text
   seu_usuario ALL=(ALL) NOPASSWD: /usr/bin/tee /sys/class/drm/card1-DP-1/status
   ```

---

### Passo 3: Autostart no GNOME

Para executar a verificação automaticamente assim que o ambiente de trabalho carregar:

1. Crie o arquivo de inicialização do desktop:
   ```bash
   nano ~/.config/autostart/ativar_monitor.desktop
   ```
2. Insira as seguintes linhas (ajustando o caminho da sua `/home/seu_usuario/`):
   ```ini
   [Desktop Entry]
   Type=Application
   Exec=/home/seu_usuario/scripts/ativar_monitor_dp.sh
   Hidden=false
   NoDisplay=false
   X-GNOME-Autostart-enabled=true
   Name=Ativar Monitor DP Dinamico
   Comment=Forca ativacao da porta DP no Pop!_OS para adaptadores passivos
   ```

---

## 🌟 Vantagens do Método

- 🛡️ **Boot Seguro:** Mantém o sistema de boot intacto sem modificações no kernelstub ou GRUB.
- 👻 **Sem Telas Fantasma:** Se o segundo monitor estiver desligado ou desconectado, o script detecta o EDID vazio (0 bytes), desliga a interface via `xrandr --off` e impede que o cursor do mouse se perca em uma área de trabalho invisível.
- ⚡ **Estabilidade no GDM:** Evita telas cinzas, travamentos ou engasgos na tela de login do Pop!_OS.

---

## 🖼️ Galeria de Adaptadores e Testes

| Adaptador | Imagem | Resultado |
| :--- | :---: | :--- |
| **Adaptador DP→HDMI Passivo** *(Funcional com a solução)* | ![Adaptador Funcional](https://plus.diolinux.com.br/uploads/default/original/3X/e/6/e6e4f2f2df9d5e3dbdd8ed630841a537dd0e84e7.jpeg) | ✅ **Aprovado** (Funciona perfeitamente com a ativação dinâmica pós-login). |
| **Cabo/Adaptador Ativo Direct DP→HDMI** | ![Adaptador Incompatível](https://plus.diolinux.com.br/uploads/default/original/3X/7/5/75bf40e304a48637d406cbba64c76f7ad2eecf6d.jpeg) | ❌ **Incompatível** (Não realiza handshake com o firmware/BIOS da RX 5500). |

---

## 📜 Licença

Distribuído sob a licença MIT. Sinta-se livre para adaptar e compartilhar.

## 📜 ADVERTÊNCIAS E CUIDADOS

Use as instruções por sua conta e risco, sem qualquer garantia para eventuais danos a dispositivos e arquivos. Use a descrição como balizador e identificação de seus hardware e softwares.
