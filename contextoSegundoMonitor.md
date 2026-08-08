Contexto Técnico: Bug de Tela Preta na Saída DisplayPort no Pop!_OS (GPU AMD RX 5500)

> ✅ **STATUS: RESOLVIDO (08/08/2026)** — diagnóstico final, evidências e solução aplicada nas seções 5 a 9.
🖥️ 1. Hardware e Ambiente de Software

    Sistema Operacional: Pop!_OS 22.04 LTS (Kernel 7.0.x / GNOME / GDM3).

    Placa de Vídeo (GPU): AMD Radeon RX 5500 / RX 5500M (Arquitetura Navi 14 - Mesa / amdgpu driver open-source).

    Monitor Principal: LG Ultrawide 2560x1080 via porta HDMI-A-0 (Funciona 100%).

    Monitor Secundário: Full HD 1920x1080 (Apenas entradas HDMI e VGA).

    Porta de Saída com Problema: DisplayPort-0 (Mapeada pelo kernel como card1-DP-1 ou DisplayPort-0 no X11).

🔌 2. Diagnóstico Físico e Hardware (O que sabemos com certeza)

    Comprovação de Hardware Funcional:

        Todo o conjunto (GPU + Cabo/Adaptador + Monitor) funciona perfeitamente no Windows 10 em Dual Boot.

        Não há defeito físico na placa de vídeo, nas portas, no monitor ou nos cabos.

    Comportamento por Tipo de Cabo/Adaptador:

        Adaptador DP -> HDMI (Passivo): Dá imagem normalmente na tela da BIOS (GOP/VESA). Porém, ao carregar o driver amdgpu do Pop!_OS, a tela apaga e fica preta.

        Cabo Direto DP -> HDMI 2.1 (Ativo / Chipset interno): NÃO dá imagem nem na BIOS, nem no Linux. O driver amdgpu e a BIOS da placa de vídeo ignoram o handshake elétrico do chip deste cabo.

    Diagnóstico no Linux (xrandr / dmesg):

        O driver amdgpu reporta a porta como DisplayPort-0 disconnected (ou card1-DP-1).

        Sintoma Visual: Ao forçar resolução via xrandr, o X11 cria a área de trabalho virtual (o mouse passa para o lado direito e o Screen Layout Editor mostra o retângulo do monitor), mas a placa não envia sinal elétrico/luz para o monitor secundário.

        Log do Kernel (dmesg): O driver amdgpu reporta o erro [drm] Failed to setup vendor infoframe on connector HDMI-A-1: -22 e não dispara o sinal de HPD (Hot Plug Detect).

🚫 3. O que JÁ FOI TESTADO e NÃO DEU CERTO (Não repetir)
A. Tentar usar o Cabo Ativo 8K/4K

    Resultado: Falha total. A GPU RX 5500 não inicializa o cabo nem no nível da BIOS.

B. Forçar Parâmetros no Boot via Kernelstub (Systemd-boot)

    video=DisplayPort-0:1920x1080@60e

    video=DP-1:1920x1080@60e

    amdgpu.dc=0

    amdgpu.runpm=0

    Resultado: Fazer o override direto de portas com video= travou o processo de boot do Pop!_OS antes do login, exigindo rollback e remoção via terminal de emergência.

C. Limpeza de Modos Customizados e Edid Manual

    Criação de custom modes via cvt / xrandr --newmode "1920x1080r" e injeção via xrandr --addmode.

    Resultado: O espaço de tela virtual é alocado pelo gerenciador de janelas, mas o hardware da placa de vídeo continua sem transmitir sinal físico (tela preta).

D. Forçar relink do conector via /sys/class/drm/

    echo detect | sudo tee /sys/class/drm/card1-DP-1/status

    Resultado: Ignorado pelo subsistema AMD DC (Display Core).

📍 4. Estado Atual e Próximos Passos para Pesquisa

    O sistema voltou ao estado estável padrão (boot do Pop!_OS funcionando sem travamentos).

    Fato-chave para a nova pesquisa: O adaptador Passivo funciona na BIOS e no Windows 10, o que prova que a porta transmite sinal TMDS/HDMI via modo DP++. O problema é 100% restrito à gestão de conectores do driver de código aberto amdgpu (AMD Display Core / DC) na arquitetura Navi 14 (RX 5000) no Linux.

Objetivos da Nova Pesquisa:

    Contornar a falha de HPD (Hot Plug Detect) no driver amdgpu para carregamento de EDID customizada via arquivo na pasta /lib/firmware/edid/.

    Verificar bugs conhecidos e workarounds específicos do kernel Linux para conversores DP++ passivos na família AMD RX 5000 (Navi 14).

✅ 5. Diagnóstico Final — Causa Raiz Confirmada (08/08/2026)

Ambiente confirmado em produção: Kernel 7.0.11-76070011-generic, Pop!_OS 22.04, sessão X11 (Xorg + driver AMDGPU), GNOME/GDM3.

Mapa de conectores (verificado no sistema):

    /sys/class/drm/ expõe: card1-DP-1, card1-DP-2, card1-DP-3, card1-HDMI-A-1

    Nomes do MESMO conector em cada camada (fonte de confusão recorrente):
        Kernel (parâmetro video=):  DP-1, DP-2, DP-3, HDMI-A-1
        xrandr/X11:                 DisplayPort-0, DisplayPort-1, DisplayPort-2, HDMI-A-0
        sysfs:                      card1-DP-1 ... card1-HDMI-A-1

    Monitor principal (LG HDR WFHD 2560x1080): HDMI física = HDMI-A-1 (kernel) = HDMI-A-0 (xrandr)
    Monitor secundário (LG 22MP55 1920x1080, via adaptador DP→HDMI passivo): DP-1 (kernel) = DisplayPort-0 (xrandr)

Cadeia de evidências:

    a) HPD morto: com o monitor secundário fisicamente conectado e ligado, as três portas DP reportam disconnected:
           cat /sys/class/drm/card1-DP-{1,2,3}/status   →   disconnected

    b) DDC/EDID 100% íntegro através do adaptador passivo. Topologia i2c do amdgpu (i2cdetect -l):
           i2c-3..6 = "AMDGPU DM i2c hw bus 0..3" → DDC de cada conector (ordem: DP-1, DP-2, DP-3, HDMI-A-1)
           i2c-7..9 = "AMDGPU DM aux hw bus 0..2" → protocolo AUX nativo das três DP
       Leituras:
           sudo get-edid -b 3  → 256 bytes = EDID do LG 22MP55 (o secundário, lido ATRAVÉS do adaptador passivo)
           sudo get-edid -b 6  → 256 bytes = EDID do LG HDR WFHD (md5 idêntico a /sys/class/drm/card1-HDMI-A-1/edid)
           i2c-7..9            → 0 bytes (esperado: adaptador passivo não fala protocolo AUX; o DDC trafega
                                 pelos pinos AUX em modo I2C elétrico, por isso aparece no barramento "i2c hw")

    c) O erro "[drm] Failed to setup vendor infoframe on connector HDMI-A-1: -22" é PISTA FALSA:
           - Dispara em TODO boot, na porta do monitor que FUNCIONA (a HDMI do ultrawide), que permanece operacional.
           - É um aviso não-fatal (drm_warn_once) introduzido no kernel 6.16 pelo commit 6027cbee1900
             ("drm/amd/display: Add error check for avi and vendor infoframe setup function"); o -22 (EINVAL)
             ocorre quando o conector tipado como HDMI não tem HDMI infoframe disponível — condição que a
             própria documentação do kernel classifica como "erro que pode ser ignorado com segurança".

    d) CONCLUSÃO: a única falha real é o HPD não reportado ao driver. DDC e TMDS funcionam no Linux.
       O bug pertence à classe "detecção de dongle passivo DP→HDMI no AMD DC", que a AMD corrige
       repetidamente desde 2019 — ex.: commit bc2fde42e241 "drm/amd/display: Passive DP->HDMI dongle
       detection fix" (v5.4) — e que segue ativa: patch "Add passive dongle handling in force_to_use_aux
       case" (amd-gfx, jul/2026, alvo drm-next-7.3).

Por que as tentativas anteriores (seção 3) falharam:

    - amdgpu.dc=0: na arquitetura Navi NÃO existe caminho de vídeo sem o DC; desativá-lo elimina TODO o
      display. Esta foi a causa real do boot travado — não foi o parâmetro video=.
    - video=DisplayPort-0:...: nome inválido para o kernel. O parâmetro video= exige o nome kernel-style
      (DP-1); "DisplayPort-0" é nome do xrandr e é silenciosamente ignorado.
    - echo detect > .../status: o valor "detect" apenas repete a detecção baseada em HPD (que está morto).
      O sysfs do DRM (drivers/gpu/drm/drm_sysfs.c, status_store) também aceita "on"/"off", que FORÇAM o
      estado do conector — era isso que faltava.
    - xrandr --newmode/--addmode: só aloca framebuffer virtual no X; com o conector marcado disconnected,
      o transmissor físico nunca é armado (tela preta com desktop estendido).

🔧 6. Solução Aplicada (testada e funcionando em 08/08/2026)

A) Teste em runtime — sem reboot, totalmente reversível:

    echo on | sudo tee /sys/class/drm/card1-DP-1/status
    xrandr --output DisplayPort-0 --off        # limpa modo fantasma das tentativas anteriores
    xrandr --output DisplayPort-0 --auto --right-of HDMI-A-0

    Resultado observado: "DisplayPort-0 connected 1920x1080", monitor acende na hora. O DRM pula a
    detecção por HPD e lê o EDID real via DDC (que funciona), fornecendo os modos corretos.
    Desfazer: echo detect | sudo tee /sys/class/drm/card1-DP-1/status

B) Rotação para retrato:

    xrandr --output DisplayPort-0 --rotate left     # vira 1080x1920; use "right" se ficar invertido

C) Solução permanente (sobrevive a reboot):

    sudo kernelstub -a "video=DP-1:1920x1080@60e"
    sudo kernelstub -p        # confere se o parâmetro entrou na cmdline

    - A flag "e" força o conector habilitado desde o boot — mesmo mecanismo DRM do item A.
    - NÃO adicionar amdgpu.dc=0 (ver seção 5).
    - Para remover: sudo kernelstub -d "video=DP-1:1920x1080@60e"
    - Rotação persistente: Ajustes GNOME → Monitores → selecionar o 22MP55 → Orientação: Retrato
      (grava em ~/.config/monitors.xml e reaplica a cada login).

D) Emergência no boot: no menu do systemd-boot, teclar "e", apagar o parâmetro video= da linha de
   opções e prosseguir o boot normalmente.

🧪 7. Fallback Documentado (não foi necessário neste caso)

Se o DDC também estivesse quebrado, o caminho suportado seria injetar a EDID via firmware — o EDID real
já havia sido capturado em /tmp/edid-3.bin durante o diagnóstico:

    sudo cp /tmp/edid-3.bin /lib/firmware/edid/22mp55.bin
    sudo kernelstub -a "video=DP-1:e drm.edid_firmware=DP-1:edid/22mp55.bin"
    sudo update-initramfs -u -k all

    Notas: em kernels recentes o parâmetro foi movido para drm_kms_helper.edid_firmware= (o nome antigo
    drm.edid_firmware= permanece como alias depreciado). Há ainda a alternativa em runtime via debugfs:
    /sys/kernel/debug/dri/*/DP-1/override_edid (requer root).

⚠️ 8. Parâmetros do amdgpu Investigados que NÃO Resolvem Este Bug

    - amdgpu.dc=0                        → remove TODO o display na Navi (causa do boot travado, seção 3.B)
    - amdgpu.dcdebugmask                 → nenhum bit força HPD/detecção de dongle (0x200000 apenas PULA
                                           o link training de detecção)
    - amdgpu.hdmi_hpd_debounce_delay_ms  → só filtra toggles espúrios de desconexão; não cria HPD ausente
    - amdgpu.noretry                     → compute (retry de page-fault GFX9), sem relação com display
    - amdgpu.forcelongtraining           → treino de VRAM, não de link DP
    - amdgpu.runpm=0                     → gerenciamento de energia; sem efeito sobre HPD

📝 9. Resumo Executivo

    Sintoma: monitor secundário em porta DP via adaptador passivo DP→HDMI funciona na BIOS e no Windows,
    mas no Linux (amdgpu/Navi 14) a porta consta "disconnected" e forçar modo via xrandr dá tela preta.

    Causa: HPD não reportado ao driver (classe de bug conhecida de detecção de dongle passivo no AMD DC);
    DDC e TMDS íntegros.

    Solução: forçar o conector no DRM — em runtime: echo on > /sys/class/drm/card1-DP-1/status;
    permanente: video=DP-1:1920x1080@60e na cmdline (via kernelstub). Nunca usar amdgpu.dc=0 na Navi.
    Rotação retrato: xrandr --rotate left + Ajustes GNOME para persistência.
