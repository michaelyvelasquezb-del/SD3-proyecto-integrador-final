# proyecto integrador 2da entrega

## Integrantes
Michael Yesid Velasquez V.- Cod: 94882 Yeison Gabriel Niño J. - Cod: 61096 Carlos Eduardo Puentes L. - Cod: 89466


## Documentación
import time
import machine
import onewire
import ds18x20
import json
from machine import Pin 
from umqtt.robust import MQTTClient
import network
import os

# === Módulo WiFi integrado ===
def conectar_wifi():
    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)
    
    # Configura tus credenciales WiFi aquí
    WIFI_SSID = "vivo V23 5G"  # Cambia por tu SSID
    WIFI_PASSWORD = "yeison123"  # Cambia por tu password
    
    if not wlan.isconnected():
        print('Conectando a WiFi...')
        wlan.connect(WIFI_SSID, WIFI_PASSWORD)
        
        # Esperar hasta que se conecte
        timeout = 0
        while not wlan.isconnected():
            time.sleep(1)
            timeout += 1
            print(f'Intentando conectar... {timeout}s')
            if timeout > 20:  # 20 segundos de timeout
                print("Error: Timeout en conexión WiFi")
                return False
                
    print('Conexión WiFi exitosa!')
    print('Configuración de red:', wlan.ifconfig())
    return True

# === Conexión WiFi ===
if not conectar_wifi():
    print("Error: no se puede continuar sin conexión WiFi")
    while True:
        time.sleep(1)

# === Configuración de LEDs ===
leds = {
    "led1": Pin(2, Pin.OUT),   # GPIO2
    "led2": Pin(4, Pin.OUT),   # GPIO4
    "led3": Pin(5, Pin.OUT),   # GPIO5
    "led4": Pin(18, Pin.OUT),  # GPIO18
    "led5": Pin(19, Pin.OUT)   # GPIO19
}

# Estado inicial de los LEDs (todos apagados)
for led in leds.values():
    led.off()

# === Configuración MQTT ===
MQTT_BROKER = "10.171.40.138"  # Broker público para pruebas
# MQTT_BROKER = "192.168.1.100"   # O usa tu broker local
MQTT_PORT = 1883
MQTT_TOPIC_SUB = "esp32/leds/control"
MQTT_TOPIC_PUB = "esp32/leds/status"
CLIENT_ID = "esp32_led_controller_" + str(time.ticks_ms())

# Variables para control de timing de LEDs
led_timers = {
    "led1": {"last_change": 0, "interval": 1000, "state": False, "blink_enabled": False},
    "led2": {"last_change": 0, "interval": 1500, "state": False, "blink_enabled": False},
    "led3": {"last_change": 0, "interval": 2000, "state": False, "blink_enabled": False},
    "led4": {"last_change": 0, "interval": 2500, "state": False, "blink_enabled": False},
    "led5": {"last_change": 0, "interval": 3000, "state": False, "blink_enabled": False}
}

# === Callback para mensajes MQTT ===
def mqtt_callback(topic, msg):
    try:
        topic = topic.decode('utf-8')
        msg = msg.decode('utf-8')
        print(f"Mensaje recibido: {topic} -> {msg}")
        
        if topic == MQTT_TOPIC_SUB:
            data = json.loads(msg)
            led_id = data.get("led")
            action = data.get("action")
            interval = data.get("interval")
            
            if led_id in leds:
                if action == "on":
                    leds[led_id].on()
                    led_timers[led_id]["state"] = True
                    led_timers[led_id]["blink_enabled"] = False
                    print(f"LED {led_id} ENCENDIDO")
                    
                elif action == "off":
                    leds[led_id].off()
                    led_timers[led_id]["state"] = False
                    led_timers[led_id]["blink_enabled"] = False
                    print(f"LED {led_id} APAGADO")
                    
                elif action == "toggle":
                    current_state = leds[led_id].value()
                    leds[led_id].value(not current_state)
                    led_timers[led_id]["state"] = not current_state
                    led_timers[led_id]["blink_enabled"] = False
                    print(f"LED {led_id} ALTERNADO")
                    
                elif action == "blink":
                    if interval:
                        led_timers[led_id]["interval"] = interval
                    led_timers[led_id]["blink_enabled"] = True
                    print(f"LED {led_id} parpadeando cada {led_timers[led_id]['interval']}ms")
                
                elif action == "stop_blink":
                    led_timers[led_id]["blink_enabled"] = False
                    print(f"LED {led_id} parpadeo detenido")
                
                # Publicar estado actualizado
                publish_status()
                
    except Exception as e:
        print(f"Error procesando mensaje: {e}")

# === Publicar estado de los LEDs ===
def publish_status():
    try:
        status = {}
        for led_id, led_pin in leds.items():
            status[led_id] = {
                "state": "on" if led_pin.value() else "off",
                "blink_interval": led_timers[led_id]["interval"],
                "blink_enabled": led_timers[led_id]["blink_enabled"]
            }
        
        client.publish(MQTT_TOPIC_PUB, json.dumps(status))
        print("Estado publicado")
    except Exception as e:
        print(f"Error publicando estado: {e}")

# === Control automático de parpadeo ===
def update_led_blink():
    current_time = time.ticks_ms()
    
    for led_id, timer in led_timers.items():
        if timer["blink_enabled"]:
            if time.ticks_diff(current_time, timer["last_change"]) >= timer["interval"]:
                # Alternar estado del LED
                leds[led_id].value(not leds[led_id].value())
                timer["state"] = not timer["state"]
                timer["last_change"] = current_time

# === Conexión MQTT ===
try:
    client = MQTTClient(CLIENT_ID, MQTT_BROKER, port=MQTT_PORT, keepalive=60)
    client.set_callback(mqtt_callback)
    client.connect()
    client.subscribe(MQTT_TOPIC_SUB)
    print(f"Conectado a MQTT Broker: {MQTT_BROKER}")
    print(f"Suscrito a: {MQTT_TOPIC_SUB}")
    print(f"Client ID: {CLIENT_ID}")
    
    # Publicar estado inicial
    publish_status()
    
except Exception as e:
    print(f"Error conectando MQTT: {e}")
    while True:
        time.sleep(1)

# === Loop principal ===
print("Sistema listo. Esperando comandos...")
print("Comandos disponibles: on, off, toggle, blink, stop_blink")

last_status_publish = time.ticks_ms()
STATUS_INTERVAL = 30000  # Publicar estado cada 30 segundos

while True:
    try:
        # Verificar mensajes MQTT
        client.check_msg()
        
        # Actualizar parpadeo automático de LEDs
        update_led_blink()
        
        # Publicar estado periódicamente
        current_time = time.ticks_ms()
        if time.ticks_diff(current_time, last_status_publish) >= STATUS_INTERVAL:
            publish_status()
            last_status_publish = current_time
            
        time.sleep_ms(10)  # Pequeña pausa para evitar sobrecarga
        
    except Exception as e:
        print(f"Error en loop principal: {e}")
        time.sleep(1)
### Avances

![WhatsApp Image 2025-11-10 at 9 34 14 PM (4)](https://github.com/user-attachments/assets/ed831755-f673-4013-b951-b62b5cc6ccfb)
![WhatsApp Image 2025-11-10 at 9 34 14 PM (2)](https://github.com/user-attachments/assets/c24defc4-cdcd-4dd5-8315-31bed1f47e60)
![WhatsApp Image 2025-11-10 at 9 34 14 PM (3)](https://github.com/user-attachments/assets/156e2d21-3883-49fe-8ae0-0d8f52a5f99f)
![WhatsApp Image 2025-11-10 at 9 34 14 PM (1)](https://github.com/user-attachments/assets/e4c36a11-3f28-45aa-a113-ba8c9f4be424)
![WhatsApp Image 2025-11-10 at 9 34 14 PM](https://github.com/user-attachments/assets/efffae86-ce6f-47aa-9dc5-dca443adffda)
![WhatsApp Image 2025-11-10 at 9 34 13 PM (1)](https://github.com/user-attachments/assets/c6591485-20d9-47a9-842e-965785d5fb59)
![WhatsApp Image 2025-11-10 at 9 34 13 PM](https://github.com/user-attachments/assets/4bdc65ce-a13c-4001-9f24-b48704a8f7fe)
  
### 1. [Flujos](/SD3-proyecto-integrador/G1/flujos/flows.json)

### 2. [Programación micropython](/SD3-proyecto-integrador/G1/micropython/test.py)


