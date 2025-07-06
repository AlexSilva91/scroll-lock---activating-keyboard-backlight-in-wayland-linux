# Scroll Lock LED Watchdog para Linux (Wayland e X11)

## Descrição

Este projeto oferece um script Python que mantém o LED do **Scroll Lock** sempre aceso em sistemas Linux, especialmente em ambientes Wayland (ex: Fedora com KDE Plasma), onde comandos tradicionais para controle de LEDs (como `xset`) não funcionam corretamente.

O script monitora eventos do teclado diretamente no dispositivo `/dev/input/eventX` usando a biblioteca `evdev`. Sempre que outro LED (Caps Lock, Num Lock, etc.) for ativado ou desativado, o script verifica e **reacende o LED do Scroll Lock automaticamente**, evitando que ele apague ou "pisque".

---

## Problema

- Em ambientes modernos com Wayland, `xset led 3` e similares não funcionam para ativar LEDs do teclado.
- O comando `brightnessctl` pode acender o LED Scroll Lock, mas o sistema apaga esse LED ao alterar o estado de outros LEDs.
- Os LEDs do teclado são controlados via bits em um único registro; alterações em um LED reescrevem o estado geral, sobrescrevendo o Scroll Lock.
- O objetivo é manter o LED Scroll Lock permanentemente ligado, sem interferência dos demais LEDs.

---

## Solução Técnica

- Usar Python com a biblioteca [`evdev`](https://python-evdev.readthedocs.io/en/latest/) para acesso direto aos eventos do teclado.
- Monitorar eventos do tipo `EV_LED` para detectar mudanças nos LEDs.
- Após qualquer evento de LED, verificar se o Scroll Lock está apagado.
- Caso esteja, reacender imediatamente usando `brightnessctl`.
- Rodar o script como serviço `systemd` para execução contínua em segundo plano.

---

## Pré-requisitos

- Sistema Linux com Python 3
- Permissões para acessar `/dev/input/eventX` (normalmente requer root ou usuário no grupo `input`)
- `brightnessctl` instalado e funcional (`sudo dnf install brightnessctl` ou equivalente)
- Biblioteca Python `evdev`

---

## Instalação

### 1. Instalar a biblioteca Python `evdev`

```bash
pip install --user evdev
```

> **Nota:** Para rodar como root ou em serviço systemd, recomenda-se instalar para o root também:

```bash
sudo pip install evdev
```

### 2. Identificar o dispositivo do teclado

Liste os dispositivos de entrada com:

```bash
sudo python3 -m evdev.evtest
```

Procure pela linha correspondente ao seu teclado (exemplo):

```
2   /dev/input/event2    USB usb keyboard
```

Anote o caminho `/dev/input/event2` para uso no script.

### 3. Permissões de acesso ao dispositivo

Para evitar usar `sudo` sempre, adicione seu usuário ao grupo `input`:

```bash
sudo usermod -aG input $USER
reboot
```

---

## Uso

### Script Python `scrolllock_watchdog.py`

Salve o conteúdo abaixo em `/usr/local/bin/scrolllock_watchdog.py` (ajuste `DEVICE_PATH` conforme o seu dispositivo):

```python
#!/usr/bin/env python3
import evdev
import os

DEVICE_PATH = '/dev/input/event2'        # Ajuste conforme seu teclado
DEVICE_NAME = 'input2::scrolllock'       # Nome do dispositivo no brightnessctl
BRIGHTNESSCTL = '/usr/bin/brightnessctl' # Caminho do brightnessctl

def get_scrolllock_state():
    try:
        result = os.popen(f"{BRIGHTNESSCTL} --device='{DEVICE_NAME}' g").read().strip()
        return int(result)
    except:
        return 0

def turn_on_scroll_lock():
    os.system(f"{BRIGHTNESSCTL} --device='{DEVICE_NAME}' set 1")

def main():
    try:
        device = evdev.InputDevice(DEVICE_PATH)
        print(f"🔍 Monitorando teclado: {device.name} ({DEVICE_PATH})")

        # Garante que Scroll Lock começa aceso
        if get_scrolllock_state() == 0:
            turn_on_scroll_lock()

        # Loop para monitorar eventos EV_LED e garantir Scroll Lock ligado
        for event in device.read_loop():
            if event.type == evdev.ecodes.EV_LED:
                if get_scrolllock_state() == 0:
                    turn_on_scroll_lock()

    except PermissionError:
        print("❌ Permissão negada. Use sudo ou adicione o usuário ao grupo 'input'.")
    except FileNotFoundError:
        print("❌ Dispositivo não encontrado. Verifique o caminho.")
    except KeyboardInterrupt:
        print("\n✅ Encerrado pelo usuário.")

if __name__ == "__main__":
    main()
```

Dê permissão de execução:

```bash
sudo chmod +x /usr/local/bin/scrolllock_watchdog.py
```

Teste rodando:

```bash
sudo /usr/local/bin/scrolllock_watchdog.py
```

---

## Configuração como serviço systemd

Para manter o script rodando sempre, crie o serviço:

```bash
sudo nano /etc/systemd/system/scrolllock-watchdog.service
```

Conteúdo:

```ini
[Unit]
Description=Scroll Lock LED Watchdog
After=multi-user.target

[Service]
ExecStart=/usr/bin/python3 /usr/local/bin/scrolllock_watchdog.py
Restart=always
User=root

[Install]
WantedBy=multi-user.target
```

Habilite e inicie o serviço:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now scrolllock-watchdog.service
```

Verifique o status:

```bash
sudo systemctl status scrolllock-watchdog.service
```

---

## Resultado Esperado

- O LED do Scroll Lock permanecerá sempre aceso.
- Mesmo que Caps Lock, Num Lock ou outros LEDs sejam ativados/desativados, o Scroll Lock é automaticamente reativado sem piscadas.
- Funciona tanto em Wayland quanto X11, pois acessa diretamente o hardware via `/dev/input`.

---

## Dicas e Segurança

- Rodar o script como root ou adicionar o usuário ao grupo `input` é necessário para acessar `/dev/input/eventX`.
- Evite rodar scripts Python com `sudo` diretamente, prefira configurar o serviço systemd para maior controle e segurança.
- `brightnessctl` deve reconhecer corretamente o nome do dispositivo LED (`input2::scrolllock`), caso contrário ajuste com:

```bash
brightnessctl --list-devices
```

---

## Referências

- [python-evdev Documentation](https://python-evdev.readthedocs.io/)
- [brightnessctl GitHub](https://github.com/Hummer12007/brightnessctl)
- Fórum Fedora & KDE Plasma no Wayland sobre LEDs

---

## Contato

Desenvolvido por Alex Alves - para dúvidas, melhorias e sugestões, entre em contato!

---

*Este README foi gerado com suporte do ChatGPT e contém todas as informações para replicar a solução de manter o LED Scroll Lock sempre aceso em sistemas Linux modernos.*  
