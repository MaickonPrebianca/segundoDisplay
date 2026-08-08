📄 [RESOLVIDO] Bug de Tela Preta: AMD RX 5500 + Adaptador DP→HDMI Passivo no Pop!_OS

    STATUS: RESOLVIDO, DINÂMICO E ESTÁVEL
    Resumo: O problema foi identificado como uma falha no sinal de HPD (Hot Plug Detect) do driver amdgpu (AMD DC) ao utilizar adaptadores passivos na arquitetura Navi (RX 5000/6000). A solução forçada via cmdline no boot instabilizava o GDM. A solução final utiliza ativação inteligente em pós-login com checagem de EDID, garantindo que a tela só acenda se o cabo e o monitor estiverem fisicamente conectados.

🖥️ 1. Ambiente e Hardware

    SO: Pop!_OS 22.04 LTS (Kernel 7.0.x / X11 / GNOME).

    GPU: AMD Radeon RX 5500 / Navi 14 (amdgpu open-source).

    Configuração de Monitores:

        Principal: HDMI-A-0 (Conexão HDMI direta).

        Secundário: DisplayPort-0 (Adaptador Passivo DP→HDMI).

    Sintoma: O monitor secundário funcionava na BIOS e no Windows, mas no Linux o driver marcava a porta como disconnected, impedindo a transmissão de vídeo.

🚫 2. O que NÃO funcionou (Evite estas abordagens)

    amdgpu.dc=0: Incompatível com a arquitetura RDNA/Navi. Desativa completamente a saída de vídeo da GPU (boot travado).

    kernelstub -a "video=DP-1:...": Forçar o conector na inicialização do kernel causava tela cinza no GDM, pois o gerenciador de exibição tentava aplicar a resolução antes do carregamento completo do driver DRM.

    echo detect > status: A varredura padrão de HPD falha em reconhecer dongles passivos sem intervenção manual no estado da porta.

🔧 3. Solução Definitiva (Script Inteligente Pós-Login)

A solução consiste em delegar a ativação da porta para o ambiente do usuário (pós-login), validando primeiro se há dados no barramento DDC/EDID da porta antes de ligar a tela.
Passo A: Script de Ativação Dinâmica

Crie o arquivo no caminho ~/scripts/ativar_monitor_dp.sh:
Bash

#!/bin/bash

# 1. Alimenta brevemente o conector para habilitar a leitura DDC
echo on | sudo tee /sys/class/drm/card1-DP-1/status > /dev/null

# 2. Aguarda a estabilização do barramento I2C
sleep 0.5

# 3. Verifica se existe um bloco EDID válido (se há monitor conectado)
if [ -s /sys/class/drm/card1-DP-1/edid ]; then
    # [MONITOR CONECTADO]
    sleep 1.5
    # Liga e posiciona a segunda tela à esquerda do display principal
    xrandr --output DisplayPort-0 --auto --left-of HDMI-A-0
else
    # [CABO DESCONECTADO / MONITOR DESLIGADO]
    # Retorna o conector ao estado 'detect' e desliga a saída no X11
    echo detect | sudo tee /sys/class/drm/card1-DP-1/status > /dev/null
    xrandr --output DisplayPort-0 --off
fi

Dê permissão de execução:
Bash

chmod +x ~/scripts/ativar_monitor_dp.sh

Passo B: Permissão de Escrita sem Senha (Sudoers)

Para permitir que o script altere o status da porta no /sys sem exigir a senha do usuário a cada boot:

    Digite sudo visudo.

    Adicione a seguinte linha ao final do arquivo:
    Plaintext

    seu_usuario ALL=(ALL) NOPASSWD: /usr/bin/tee /sys/class/drm/card1-DP-1/status

    (Substitua seu_usuario pelo seu login no sistema).

Passo C: Configuração do Autostart no GNOME

Para executar o script automaticamente assim que o desktop carregar:

    Crie o arquivo ~/.config/autostart/ativar_monitor.desktop:

Ini, TOML

[Desktop Entry]
Type=Application
Exec=/home/seu_usuario/scripts/ativar_monitor_dp.sh
Hidden=false
NoDisplay=false
X-GNOME-Autostart-enabled=true
Name=Ativar Monitor DP Dinamico

(Ajuste o caminho da /home/seu_usuario/ conforme a sua instalação).
💡 4. Vantagens do Método Inteligente

    Boot Seguro: O sistema faz o carregamento padrão sem alterar parâmetros críticos no kernelstub / GRUB.

    Sem Telas Fantasma: Se você desconectar o adaptador ou desligar o segundo monitor, o script detecta o EDID vazio (0 bytes), mantém a porta desligada (xrandr --off) e impede que o ponteiro do mouse fique "perdido" em um espaço invisível do desktop.

    Independência de GDM: Evita telas cinzas e engasgos no login.
