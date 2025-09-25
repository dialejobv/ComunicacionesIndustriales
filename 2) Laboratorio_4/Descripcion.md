# Práctica Número 4- Estudiantes de Ingeniería Electrónica de la Universidad Santo Tomás

## Primer Punto: Probando la configuración básica de un switch y un router.

Los estudiantes deben desarrollar una conexión básica de un router, un switch y una raspberry pi como se muestra en la figura:

<img width="2418" height="1196" alt="image" src="https://github.com/user-attachments/assets/0251792a-ba38-41dd-a89f-d1aee497a917" />

## Segundo Punto: Comunicación RS485 con Raspberry Pi Pico (Esclavo) y Raspberry Pi (Maestro) - Simplex, Half Dúplex y Full Duplex

### Objetivo del ítem:

* Implementar comunicación RS485 Simplex (unidireccional)
* Implementar comunicación RS485 Full Duplex usando 2 buses
* Comparar rendimiento entre ambos modos
* Visualizar datos en tiempo real con Streamlit

### Materiales Requeridos

* Raspberry Pi (Maestro)
* Raspberry Pi Pico (Esclavo) ×2
* Módulos RS485 (MAX485) ×3
* Cables jumper y protoboard
* Fuente de alimentación estable
* Resistencias de 120Ω (terminación)

## Arquitectura del sistema:

### Comunicación Simplex (Unidireccional)

```
Raspberry Pi (Maestro TX) → RS485 Bus → Raspberry Pi Pico (Esclavo RX)
```

Para la configuración simplex se tiene lo siguiente:

```
Raspberry Pi (Maestro TX) → MAX485 → Bus → MAX485 → Raspberry Pi Pico (Esclavo RX)

RPi GPIO:
- TX (GPIO 14) → DI del MAX485
- GND → GND
- 3.3V → VCC

MAX485 Maestro:
- DE/RE → 3.3V (siempre en transmisión)
- RO → Sin conectar
- A/B → Bus RS485

RPi Pico Esclavo:
- RX (GPIO 1) → RO del MAX485
- GND → GND
- 3.3V → VCC

MAX485 Esclavo:
- DE/RE → GND (siempre en recepción)
- DI → Sin conectar
- A/B → Bus RS485
```

Para la configuración modo simplex en modo recepción en el código Raspberry Pi Pico que funcionaría como Esclavo se tendría:

```python
# simplex_receiver.py
import machine
import utime
from micropython import const

# Configurar UART para RS485
uart = machine.UART(0, baudrate=9600, tx=machine.Pin(0), rx=machine.Pin(1))

# LED indicador
led = machine.Pin(25, machine.Pin.OUT)

# Variables de estadísticas
packet_count = 0
error_count = 0
last_packet_time = utime.ticks_ms()

def calculate_checksum(data):
    return sum(data) & 0xFF

def process_packet(packet):
    global packet_count, error_count
    
    if len(packet) < 5:  # Mínimo: tipo(1) + data(2) + seq(1) + checksum(1)
        error_count += 1
        return None
    
    # Verificar checksum
    received_checksum = packet[-1]
    calculated_checksum = calculate_checksum(packet[:-1])
    
    if received_checksum != calculated_checksum:
        error_count += 1
        return None
    
    packet_count += 1
    
    # Extraer datos del paquete
    packet_type = packet[0]
    data_value = (packet[1] << 8) | packet[2]
    sequence = packet[3]
    
    return {
        'type': packet_type,
        'data': data_value,
        'sequence': sequence,
        'timestamp': utime.ticks_ms()
    }

print("Esclavo Simplex - Listo para recibir...")

while True:
    if uart.any():
        # Leer datos disponibles
        data = uart.read()
        
        if data:
            # Procesar cada byte
            for byte in data:
                packet = process_packet(byte)  # Aquí deberías implementar un protocolo real
                if packet:
                    led.toggle()
                    print(f"Paquete {packet['sequence']}: Dato={packet['data']}")
    
    # Enviar estadísticas cada 10 segundos (por UART adicional si es necesario)
    if utime.ticks_diff(utime.ticks_ms(), last_packet_time) > 10000:
        print(f"Estadísticas: Paquetes={packet_count}, Errores={error_count}")
        last_packet_time = utime.ticks_ms()
    
    utime.sleep_ms(10)
```

El código para la Raspberry Pi como maestro para la Comunicación Simplex sería: 

```python
# maestro_simplex.py
import serial
import time
import struct
import threading
from datetime import datetime

class RS485SimplexMaster:
    def __init__(self, port='/dev/ttyAMA0', baudrate=9600):
        self.ser = serial.Serial(port, baudrate, timeout=1)
        self.sequence_number = 0
        self.stats = {
            'tx_packets': 0,
            'tx_bytes': 0,
            'start_time': time.time()
        }
        
    def calculate_checksum(self, data):
        return sum(data) & 0xFF
    
    def create_packet(self, data_value, packet_type=0x01):
        packet = bytearray()
        packet.append(packet_type)
        packet.extend(struct.pack('>H', data_value))  # 2 bytes big-endian
        packet.append(self.sequence_number & 0xFF)
        
        checksum = self.calculate_checksum(packet)
        packet.append(checksum)
        
        self.sequence_number += 1
        return packet
    
    def transmit_data(self, data_value):
        packet = self.create_packet(data_value)
        self.ser.write(packet)
        
        self.stats['tx_packets'] += 1
        self.stats['tx_bytes'] += len(packet)
        
        return packet
    
    def get_statistics(self):
        uptime = time.time() - self.stats['start_time']
        return {
            'tx_packets': self.stats['tx_packets'],
            'tx_bytes': self.stats['tx_bytes'],
            'uptime': uptime,
            'data_rate': self.stats['tx_bytes'] / uptime if uptime > 0 else 0
        }

# Uso del maestro simplex
if __name__ == "__main__":
    master = RS485SimplexMaster()
    
    try:
        counter = 0
        while True:
            # Transmitir dato simulado
            data = counter % 1000
            packet = master.transmit_data(data)
            
            print(f"Transmitido: {data}, Paquete: {packet.hex()}")
            
            # Estadísticas cada 10 transmisiones
            if counter % 10 == 0:
                stats = master.get_statistics()
                print(f"Estadísticas: {stats}")
            
            counter += 1
            time.sleep(2)
            
    except KeyboardInterrupt:
        master.ser.close()
```

### Comunicación Full Duplex (Bidireccional simultánea)

```
Raspberry Pi (Maestro) ↔ RS485 Bus 1 ↔ Raspberry Pi Pico 1 (Esclavo 1)
Raspberry Pi (Maestro) ↔ RS485 Bus 2 ↔ Raspberry Pi Pico 2 (Esclavo 2)
```
Para la configuración duplex se tiene lo siguiente:

```
# Bus 1: Maestro TX → Esclavo 1 RX
# Bus 2: Esclavo 2 TX → Maestro RX

Raspberry Pi:
- TX (GPIO 14) → MAX485_1 DI → Bus 1
- RX (GPIO 15) ← MAX485_2 RO ← Bus 2
- GPIO 18 → DE/RE MAX485_1 (control TX)
- GPIO 23 → DE/RE MAX485_2 (control RX)

RPi Pico 1 (Solo RX):
- RX (GPIO 1) ← MAX485 RO ← Bus 1
- DE/RE → GND (siempre recepción)

RPi Pico 2 (Solo TX):
- TX (GPIO 0) → MAX485 DI → Bus 2  
- DE/RE → 3.3V (siempre transmisión)
```

Para la configuración modo full dúplex en modo transmisor en el código Raspberry Pi Pico que funcionaría como Esclavo se tendría:


```python
# full_duplex_transmitter.py
import machine
import utime
import json

# Configurar UART para transmisión
uart = machine.UART(0, baudrate=9600, tx=machine.Pin(0), rx=machine.Pin(1))

# Sensor simulado (potenciómetro o temperatura)
sensor = machine.ADC(26)  # GP26 para entrada analógica

# Variables
sequence_number = 0
transmission_count = 0

def create_packet(sensor_value, packet_type=0x01):
    global sequence_number
    
    packet = bytearray()
    packet.append(packet_type)  # Tipo de paquete
    packet.append((sensor_value >> 8) & 0xFF)  # Byte alto
    packet.append(sensor_value & 0xFF)  # Byte bajo
    packet.append(sequence_number & 0xFF)  # Número de secuencia
    
    # Calcular checksum
    checksum = sum(packet) & 0xFF
    packet.append(checksum)
    
    sequence_number += 1
    return packet

print("Transmisor Full Duplex - Iniciado...")

while True:
    # Leer sensor
    sensor_value = sensor.read_u16()
    
    # Crear paquete
    packet = create_packet(sensor_value)
    
    # Transmitir
    uart.write(packet)
    transmission_count += 1
    
    # LED indicador
    machine.Pin(25, machine.Pin.OUT).toggle()
    
    print(f"Transmitido: {sensor_value}, Secuencia: {sequence_number-1}")
    
    utime.sleep_ms(1000)  # Transmitir cada segundo
```

El código maestro para la comunicación full dúplex es:

```python
# maestro_full_duplex.py
import serial
import time
import struct
import threading
import RPi.GPIO as GPIO

class RS485FullDuplexMaster:
    def __init__(self, tx_port='/dev/ttyAMA0', rx_port='/dev/ttyUSB0', baudrate=9600):
        # Configurar GPIO para control DE/RE
        GPIO.setmode(GPIO.BCM)
        self.tx_control_pin = 18
        self.rx_control_pin = 23
        
        GPIO.setup(self.tx_control_pin, GPIO.OUT)
        GPIO.setup(self.rx_control_pin, GPIO.OUT)
        
        # Puerto TX (transmisión)
        self.tx_ser = serial.Serial(tx_port, baudrate, timeout=0.1)
        # Puerto RX (recepción)
        self.rx_ser = serial.Serial(rx_port, baudrate, timeout=0.1)
        
        self.sequence_number = 0
        self.stats = {
            'tx_packets': 0, 'rx_packets': 0,
            'tx_bytes': 0, 'rx_bytes': 0,
            'errors': 0, 'start_time': time.time()
        }
        
        # Buffer para datos recibidos
        self.rx_buffer = []
        self.buffer_lock = threading.Lock()
        
    def set_tx_mode(self):
        GPIO.output(self.tx_control_pin, GPIO.HIGH)
        GPIO.output(self.rx_control_pin, GPIO.LOW)
        
    def set_rx_mode(self):
        GPIO.output(self.tx_control_pin, GPIO.LOW)
        GPIO.output(self.rx_control_pin, GPIO.HIGH)
    
    def calculate_checksum(self, data):
        return sum(data) & 0xFF
    
    def create_packet(self, data_value, packet_type=0x01):
        packet = bytearray()
        packet.append(packet_type)
        packet.extend(struct.pack('>H', data_value))
        packet.append(self.sequence_number & 0xFF)
        
        checksum = self.calculate_checksum(packet)
        packet.append(checksum)
        
        self.sequence_number += 1
        return packet
    
    def verify_packet(self, packet):
        if len(packet) < 5:
            return False
        
        received_checksum = packet[-1]
        calculated_checksum = self.calculate_checksum(packet[:-1])
        
        return received_checksum == calculated_checksum
    
    def transmit_data(self, data_value):
        self.set_tx_mode()
        time.sleep(0.001)  # Pequeña espera para estabilizar
        
        packet = self.create_packet(data_value)
        self.tx_ser.write(packet)
        
        self.stats['tx_packets'] += 1
        self.stats['tx_bytes'] += len(packet)
        
        self.set_rx_mode()  # Volver a modo recepción
        return packet
    
    def start_receiver(self):
        def receiver_thread():
            while True:
                if self.rx_ser.in_waiting > 0:
                    data = self.rx_ser.read(self.rx_ser.in_waiting)
                    
                    # Procesar datos recibidos
                    for byte in data:
                        # Aquí implementarías el protocolo de recepción
                        packet_data = {
                            'data': byte,
                            'timestamp': time.time(),
                            'valid': True
                        }
                        
                        with self.buffer_lock:
                            self.rx_buffer.append(packet_data)
                            self.stats['rx_packets'] += 1
                            self.stats['rx_bytes'] += 1
                
                time.sleep(0.01)
        
        thread = threading.Thread(target=receiver_thread)
        thread.daemon = True
        thread.start()
    
    def get_received_data(self, clear=True):
        with self.buffer_lock:
            data = self.rx_buffer.copy()
            if clear:
                self.rx_buffer.clear()
        return data
    
    def get_statistics(self):
        uptime = time.time() - self.stats['start_time']
        return {
            'tx_packets': self.stats['tx_packets'],
            'rx_packets': self.stats['rx_packets'],
            'tx_bytes': self.stats['tx_bytes'],
            'rx_bytes': self.stats['rx_bytes'],
            'errors': self.stats['errors'],
            'uptime': uptime,
            'tx_rate': self.stats['tx_bytes'] / uptime if uptime > 0 else 0,
            'rx_rate': self.stats['rx_bytes'] / uptime if uptime > 0 else 0
        }

# Uso del maestro full duplex
if __name__ == "__main__":
    master = RS485FullDuplexMaster()
    master.start_receiver()
    master.set_rx_mode()  # Iniciar en modo recepción
    
    try:
        counter = 0
        while True:
            # Transmitir cada 3 segundos
            if counter % 3 == 0:
                data = counter % 1000
                packet = master.transmit_data(data)
                print(f"📤 Transmitido: {data}")
            
            # Mostrar datos recibidos
            received = master.get_received_data()
            if received:
                print(f"📥 Recibidos {len(received)} paquetes")
            
            # Estadísticas cada 10 ciclos
            if counter % 10 == 0:
                stats = master.get_statistics()
                print(f"📊 Estadísticas: {stats}")
            
            counter += 1
            time.sleep(1)
            
    except KeyboardInterrupt:
        master.tx_ser.close()
        master.rx_ser.close()
        GPIO.cleanup()
```

Finalmente el dashboard en streamlit para la comunicación sería:

```python
# dashboard_rs485.py
import streamlit as st
import plotly.graph_objects as go
import plotly.express as px
from plotly.subplots import make_subplots
import pandas as pd
import time
import random
from datetime import datetime

st.set_page_config(
    page_title="Monitor RS485 Simplex/Full Duplex",
    layout="wide"
)

st.title("🔌 Monitor RS485: Simplex vs Full Duplex")

# Sidebar para configuración
st.sidebar.header("⚙️ Configuración")
mode = st.sidebar.radio("Modo de Comunicación", ["Simplex", "Full Duplex"])
update_interval = st.sidebar.slider("Intervalo (ms)", 100, 2000, 500)

# Simulador de datos
class DataSimulator:
    def __init__(self):
        self.simplex_data = []
        self.duplex_data = []
        self.start_time = time.time()
    
    def generate_simplex_data(self):
        return {
            'timestamp': datetime.now(),
            'data': random.randint(0, 1000),
            'sequence': len(self.simplex_data),
            'type': 'TX',  # Solo transmisión
            'mode': 'Simplex'
        }
    
    def generate_duplex_data(self):
        tx_data = {
            'timestamp': datetime.now(),
            'data': random.randint(0, 1000),
            'sequence': len(self.duplex_data),
            'type': 'TX',
            'mode': 'Full Duplex'
        }
        
        rx_data = {
            'timestamp': datetime.now(),
            'data': random.randint(0, 1000),
            'sequence': len(self.duplex_data),
            'type': 'RX', 
            'mode': 'Full Duplex'
        }
        
        return tx_data, rx_data

simulator = DataSimulator()

# Contenedores principales
col1, col2 = st.columns(2)

with col1:
    st.subheader("📈 Datos en Tiempo Real")
    
    # Gráfico principal
    placeholder = st.empty()

with col2:
    st.subheader("📊 Métricas de Rendimiento")
    
    metric_col1, metric_col2, metric_col3 = st.columns(3)
    
    with metric_col1:
        st.metric("Paquetes TX", "156", "+12")
    
    with metric_col2:
        st.metric("Paquetes RX", "148" if mode == "Full Duplex" else "0", 
                 "+8" if mode == "Full Duplex" else "0")
    
    with metric_col3:
        st.metric("Tasa Error", "0.8%", "-0.2%")

# Actualización en tiempo real
while True:
    # Generar nuevos datos según el modo
    if mode == "Simplex":
        new_data = simulator.generate_simplex_data()
        simulator.simplex_data.append(new_data)
        df = pd.DataFrame(simulator.simplex_data[-20:])  # Últimos 20 puntos
    else:
        tx_data, rx_data = simulator.generate_duplex_data()
        simulator.duplex_data.extend([tx_data, rx_data])
        df = pd.DataFrame(simulator.duplex_data[-40:])
    
    # Crear gráfico
    fig = make_subplots(rows=2, cols=1, 
                       subplot_titles=['Datos Transmitidos/Recibidos', 'Secuencia de Paquetes'])
    
    if mode == "Simplex":
        # Solo datos TX
        fig.add_trace(
            go.Scatter(
                x=df['timestamp'],
                y=df['data'],
                mode='lines+markers',
                name='TX Data',
                line=dict(color='blue')
            ),
            row=1, col=1
        )
    else:
        # Datos TX y RX
        tx_df = df[df['type'] == 'TX']
        rx_df = df[df['type'] == 'RX']
        
        fig.add_trace(
            go.Scatter(
                x=tx_df['timestamp'],
                y=tx_df['data'],
                mode='lines+markers',
                name='TX Data',
                line=dict(color='blue')
            ),
            row=1, col=1
        )
        
        fig.add_trace(
            go.Scatter(
                x=rx_df['timestamp'],
                y=rx_df['data'],
                mode='lines+markers',
                name='RX Data',
                line=dict(color='red')
            ),
            row=1, col=1
        )
    
    # Gráfico de secuencia
    fig.add_trace(
        go.Scatter(
            x=df['timestamp'],
            y=df['sequence'],
            mode='lines+markers',
            name='Secuencia',
            line=dict(color='green')
        ),
        row=2, col=1
    )
    
    fig.update_layout(height=600, showlegend=True)
    
    # Actualizar gráfico
    with placeholder.container():
        st.plotly_chart(fig, use_container_width=True)
        
        # Mostrar datos recientes
        st.subheader("📋 Últimos Paquetes")
        st.dataframe(df.tail(5), use_container_width=True)
    
    time.sleep(update_interval / 1000)
```

### Desarrollo de análisis y comparativa

Se tiene el siguiente código para el desarrollo de las pruebas de rendimiento:

```python

# performance_test.py
import time
import statistics

def test_simplex_performance(master, num_packets=100):
    start_time = time.time()
    successes = 0
    
    for i in range(num_packets):
        try:
            master.transmit_data(i)
            successes += 1
        except Exception as e:
            print(f"Error en paquete {i}: {e}")
        
        time.sleep(0.01)  # Pequeña pausa
    
    total_time = time.time() - start_time
    success_rate = (successes / num_packets) * 100
    
    print(f"🔹 Simplex - Paquetes: {num_packets}, Tiempo: {total_time:.2f}s")
    print(f"   Tasa éxito: {success_rate:.1f}%")
    print(f"   Throughput: {num_packets/total_time:.1f} pkt/s")

def test_duplex_performance(master, num_packets=100):
    start_time = time.time()
    tx_successes = 0
    rx_count = 0
    
    for i in range(num_packets):
        try:
            master.transmit_data(i)
            tx_successes += 1
        except Exception as e:
            print(f"Error TX en paquete {i}: {e}")
        
        # Contar recepciones
        rx_data = master.get_received_data()
        rx_count += len(rx_data)
        
        time.sleep(0.01)
    
    total_time = time.time() - start_time
    tx_success_rate = (tx_successes / num_packets) * 100
    
    print(f"🔸 Full Duplex - Paquetes: {num_packets}, Tiempo: {total_time:.2f}s")
    print(f"   TX éxito: {tx_success_rate:.1f}%")
    print(f"   RX recibidos: {rx_count}")
    print(f"   Throughput: {num_packets/total_time:.1f} pkt/s")

```

Se espera lo siguiente por parte del estudiante:

<img width="2354" height="778" alt="image" src="https://github.com/user-attachments/assets/cb16b20b-809a-42c0-8d64-a4259b308e16" />








