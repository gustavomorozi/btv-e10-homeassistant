# BTV E10 + Home Assistant

Guia prático (em português) para **depurar, controlar e configurar a BTV E10** (TV box Android 9, Amlogic) pelo **Home Assistant**, usando ADB pela rede. Tudo aqui foi testado em uma BTV E10 real.

> **English summary:** how to find the (non-standard) ADB port on a BTV E10, control it from Home Assistant with fast `sendevent` key presses (≈1.3 s instead of ≈2–5 s with `input keyevent`), open apps/YouTube, install APKs from HA, mirror your PC/phone screen with AirScreen (Cast), and why Miracast is not possible. Ready-to-paste `scripts.yaml` included.

## O que você vai conseguir

- Botões de controle remoto no painel do HA: início, voltar, setas, OK, menu, volume, mudo, play/pause, próxima, anterior.
- Abrir apps, abrir o YouTube e buscar/tocar vídeos, digitar texto, ligar/suspender a caixa.
- Instalar APK pelo HA ou pelo PC.
- Espelhar a tela do PC/celular na TV (Google Cast pelo AirScreen).
- Comandos de voz ("aumentar volume do receptor") via agente de conversa com apelidos.

## 1. Ativar a depuração e achar a porta do ADB

1. Na caixa: **Configurações → Sobre → toque 7× em "Número da versão"** para liberar as Opções do desenvolvedor, e ligue **Depuração USB** (e, se existir, **Depuração por rede/ADB**).
2. O cabo USB nem sempre mostra o aparelho no PC. Use a **rede**.
3. **A porta NÃO é a 5555** nesta caixa (foi `11993` na que testei, e pode ser outra na sua). Descubra o IP da caixa e varra as portas (PowerShell):

```powershell
$ip = "192.168.1.50"   # IP da sua caixa
$abertas = 1024..65535 | ForEach-Object -Parallel {
    $c = New-Object Net.Sockets.TcpClient
    $t = $c.ConnectAsync($using:ip, $_)
    if ($t.Wait(150) -and $c.Connected) { $_ }
    $c.Close()
} -ThrottleLimit 200
$abertas
```

   (PowerShell 7+. No Windows PowerShell 5.1, use `nmap -p 1024-65535 192.168.1.50`.)
4. Teste cada porta aberta: `adb connect 192.168.1.50:PORTA`. A caixa pede para autorizar o computador (marque "sempre permitir").
5. Dica: reserve um IP fixo para a caixa no roteador (DHCP reservation), senão o IP muda e quebra tudo.

## 2. Ligar o ADB ao Home Assistant

1. **Configurações → Dispositivos e serviços → Adicionar integração → Android Debug Bridge** (`androidtv`). Informe IP e a **porta** que você achou.
2. Isso cria a entidade `media_player.android_tv_<ip>` e o serviço `androidtv.adb_command`.
3. Cole os scripts de [`homeassistant/scripts.yaml`](homeassistant/scripts.yaml) no seu `scripts.yaml` (trocando a entidade de exemplo `media_player.android_tv_192_168_1_50` pela sua).

## 3. Por que `sendevent` e não `input keyevent`

`adb shell input keyevent 4` leva **~2 s por tecla**, porque o comando `input` sobe uma máquina Java a cada chamada. Pelo HA chega a 2–5 s. Escrever direto no teclado da caixa (`/dev/input/event1`, o `aml_keypad`) com `sendevent` é nativo: **~0,5 s direto e ~1,3 s pelo HA**.

```
sendevent /dev/input/event1 1 <KEY> 1; sendevent /dev/input/event1 0 0 0; sendevent /dev/input/event1 1 <KEY> 0; sendevent /dev/input/event1 0 0 0
```

| Botão | Código | Botão | Código |
|---|---|---|---|
| Início | 102 | Mudo | 113 |
| OK | 353 | Voltar | 158 |
| Cima / Baixo | 103 / 108 | Menu | 139 |
| Esquerda / Direita | 105 / 106 | Play/Pause | 164 |
| Volume + / − | 115 / 114 | Próxima / Anterior | 163 / 165 |
| Avançar | 208 | Rebobinar | 168 (não testado) |
| Acordar | 143 | Enter / Backspace | 28 / 14 |

Confirme o seu dispositivo com `adb shell getevent -lp` (procure o que tem as teclas). **Volume:** 1 toque = 1 nível (HDMI de 0 a 15). Teste devagar para não derrubar o volume da TV.

Outros comandos úteis (esses precisam de `input`/`am`, então levam ~2 s):

```
input keyevent 224      # acordar          input keyevent 223   # suspender
monkey -p <pacote> -c android.intent.category.LAUNCHER 1     # abrir app
am start -a android.intent.action.VIEW -d '<url>' -p com.google.android.youtube.tv   # abrir vídeo
input text 'texto'      # digitar (sem acentos; espaço = %s)
```

A caixa **só suspende**, não desliga de verdade.

## 4. Espelhar a tela (e o que não funciona)

- **Miracast é impossível.** O Android 9 bloqueia Miracast para apps de terceiros, e o AirScreen mostra "Desativado". O **Smart View da Samsung nunca vai enxergar a caixa**.
- Funciona **Google Cast** e **AirPlay** com o app **AirScreen** (`com.ionitech.airscreen`; a caixa aparece como `AS-E10[Cast]`):
  - **PC:** Chrome → menu ⋮ → Salvar, compartilhar e transmitir → Transmitir → escolha a caixa → Fontes: transmitir tela.
  - **Celular:** app Google Home → Todos os dispositivos → **Dispositivos locais** → a caixa → **Transmitir tela** (fica "Indisponível" se outra sessão de cast estiver ativa).
  - Apps como YouTube e Netflix: ícone de Cast.
- Qualidade: a caixa só tem Wi‑Fi **2,4 GHz**, então o vídeo vem comprimido e o texto fica mole. O que mais ajuda é **cabo de rede** na porta Ethernet da caixa; no PC, escala do Windows em 125–150%.
- O Cast do Chrome **só espelha**, não estende a tela. Para estender, use um driver de monitor virtual (e transmita a tela virtual) ou o Spacedesk.

## 5. Instalar APK

- **Pelo PC:** `adb connect IP:PORTA` e `adb install -r app.apk`.
- **Pelo HA:** o script `btv_instalar_apk` envia o arquivo com `androidtv.upload` e roda `pm install -r`. O arquivo precisa estar numa pasta liberada em `allowlist_external_dirs` (padrão: `/config/www` e `/media`; coloque o APK em `/config/www/apks` pelo File editor, porque o upload de mídia do HA recusa `.apk`).
- O HA **não recebe a saída** do `adb_command`: confira na lista de apps se instalou.
- A caixa é de 32 bits (`armeabi-v7a`): APK só de 64 bits não instala.

## 6. Segurança (leia)

- **Não instale APK de origem desconhecida.** Caixas desse tipo costumam vir com apps de IPTV não oficiais; passe qualquer arquivo no [VirusTotal](https://www.virustotal.com) antes.
- ADB e SSH abertos na rede dão controle total da caixa. Use senhas fortes (nada de `123456`), não exponha as portas à internet e desligue o ADB quando não precisar.
- **Não exponha o script de instalar APK ao assistente de voz.**
- Não publique tokens, senhas de câmeras ou IPs reais ao pedir ajuda.

## 7. Problemas comuns

| Sintoma | Causa provável |
|---|---|
| `adb devices` não mostra o aparelho pelo cabo | Use a rede (`adb connect IP:PORTA`). |
| `connection refused` | A porta não é a 5555; faça a varredura (seção 1). |
| Botão demora 2–5 s | Está usando `input keyevent`; troque por `sendevent`. |
| Comando "funciona" no HA mas nada acontece | O HA não mostra erro do shell; teste no PC com `adb shell ...`. |
| Volume caiu sozinho | Algum teste de volume; leia `dumpsys audio` e restaure. |
| Daemon do adb reiniciou e perdeu a conexão | `adb connect IP:PORTA` de novo. |
| Smart View não acha a caixa | Miracast não existe aqui; use Cast (seção 4). |

## Contribuindo

Testou em outra BTV ou em outro modelo (E10, E8, etc.)? Abra uma *issue* ou *pull request* com a porta do ADB, o dispositivo de entrada (`getevent -lp`) e o que funcionou.

Licença: [MIT](LICENSE). Projeto da comunidade, sem relação com a BTV nem com o Home Assistant.
