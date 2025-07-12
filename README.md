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
import os
import time
import threading
import evdev
import pyudev
from pathlib import Path
from evdev import InputDevice, ecodes

BRIGHTNESSCTL = '/usr/bin/brightnessctl'
HEARTBEAT_INTERVAL = 5  # segundos


def find_scrolllock_led():
    leds_path = Path("/sys/class/leds")
    for led in leds_path.iterdir():
        if "scrolllock" in led.name:
            return led.name  # ex: input2::scrolllock
    return None


def get_scrolllock_state(device_name):
    try:
        result = os.popen(f"{BRIGHTNESSCTL} --device='{device_name}' g").read().strip()
        return int(result)
    except Exception as e:
        print(f"[Erro] Estado do Scroll Lock: {e}")
        return 0


def set_scrolllock(device_name, value):
    os.system(f"{BRIGHTNESSCTL} --device='{device_name}' set {value}")


def blink_led(device_name):
    """Pisca o Scroll Lock como heartbeat."""
    while True:
        set_scrolllock(device_name, 0)
        time.sleep(0.2)
        set_scrolllock(device_name, 1)
        time.sleep(HEARTBEAT_INTERVAL)


def find_keyboard_device():
    for path in evdev.list_devices():
        try:
            device = InputDevice(path)
            capabilities = device.capabilities()
            if ecodes.EV_KEY in capabilities:
                keys = capabilities[ecodes.EV_KEY]
                # Pode ser lista de ints ou lista de tuplas (int, any)
                if isinstance(keys[0], tuple):
                    keys = [code for code, _ in keys]
                if ecodes.KEY_A in keys:
                    return path
        except Exception as e:
            continue  # ignora dispositivos problemáticos
    return None


def monitor_keyboard(input_device_path, led_device):
    """Monitora eventos de LED e reativa Scroll Lock caso desligado."""
    try:
        device = InputDevice(input_device_path)
        print(f"✅ Monitorando: {device.name} ({input_device_path})")
        for event in device.read_loop():
            if event.type == ecodes.EV_LED:
                if get_scrolllock_state(led_device) == 0:
                    set_scrolllock(led_device, 1)
    except Exception as e:
        print(f"[Erro] ao ler o teclado: {e}")
        print("⏳ Aguardando reconexão do teclado...")


def handle_udev_events(led_device):
    """Observa eventos do udev para reconectar automaticamente."""
    context = pyudev.Context()
    monitor = pyudev.Monitor.from_netlink(context)
    monitor.filter_by('input')
    observer = pyudev.MonitorObserver(monitor, callback=lambda action, device: on_device_event(action, device, led_device))
    observer.start()


def on_device_event(action, device, led_device):
    if action == "add" and 'event' in device.device_node:
        print("🔌 Novo dispositivo conectado. Reanalisando teclado...")
        time.sleep(1)
        new_device = find_keyboard_device()
        if new_device:
            threading.Thread(target=monitor_keyboard, args=(new_device, led_device), daemon=True).start()


def main():
    led_device = find_scrolllock_led()
    if not led_device:
        print("❌ Nenhum Scroll Lock LED encontrado.")
        return

    input_device_path = find_keyboard_device()
    if not input_device_path:
        print("❌ Nenhum teclado encontrado.")
        return

    # Garante LED ligado no início
    set_scrolllock(led_device, 1)

    # Inicia heartbeat
    threading.Thread(target=blink_led, args=(led_device,), daemon=True).start()

    # Inicia monitoramento de reconexão
    handle_udev_events(led_device)

    # Inicia monitoramento principal
    monitor_keyboard(input_device_path, led_device)


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
