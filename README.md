# Estação meteorológica — firmware MicroPython

**Vídeo técnico:** https://youtu.be/aV6OuZGKMI8

Repositório do Artefato 3 (código e documentação técnica aberta). Os arquivos de código são o MicroPython do ESP32: `boot.py`, `main.py` e `ssd1306.py`.

## Guia de instalação

A placa precisa do firmware MicroPython e da biblioteca `umqtt.simple`. O módulo `dht` já vem no MicroPython.

1. Copie `wifi_config.example.py` para `wifi_config.py`, coloque o nome e a senha da rede e grave esse arquivo só no ESP32.
2. Grave na raiz da placa: `boot.py`, `main.py`, `ssd1306.py` e `wifi_config.py`.
3. Reinicie. No console aparecem o IP, os endereços I2C e, a cada envio aceito, o JSON.

No Thonny, escolha o interpretador MicroPython (ESP32) e use *Salvar como → Dispositivo MicroPython*.

No computador, com a porta da placa no lugar de `COM3`:

```powershell
python -m pip install mpremote
python -m mpremote connect COM3 mip install umqtt.simple
python -m mpremote connect COM3 fs cp boot.py :boot.py + fs cp main.py :main.py + fs cp ssd1306.py :ssd1306.py + fs cp wifi_config.py :wifi_config.py + reset
```

Se `umqtt.simple` já estiver na placa, o comando `mip install` pode ser pulado.

## Esquemático eletrônico

![Esquema da montagem física](docs/esquema-eletronico.png)

| Peça | Ligação | Uso no firmware |
|---|---|---|
| DHT22 | 3,3 V, GND, dados no GPIO 14 | Temperatura em °C e umidade em % |
| BMP280 | 3,3 V, GND, SDA no GPIO 21, SCL no GPIO 22 | Pressão em hPa; tenta os endereços 0x76 e 0x77 |
| MQ-135 | alimentação do módulo e AO no GPIO 34 | Leitura ADC de 0 a 4095, atenuação 11 dB |
| Luz digital | VCC, GND e DO no GPIO 35 | Estado `Claro` ou `Escuro` |
| OLED SSD1306 128×64 | mesmo I2C, endereço 0x3C | Quatro telas locais; não entra no JSON |

A placa é alimentada pela USB. O trilho de 3,3 V do DevKit alimenta a lógica. O esquema registrado na bancada mostra o LDR analógico e o pluviômetro fora desta montagem, e o MQ-135 sem divisor de tensão. Os rótulos do ar usam limites fixos do ADC: abaixo de 1200 é `Ar Limpo`, abaixo de 2500 é `Ar Moderado`, e o restante é `Ar Poluido`. Esse rótulo é relativo; o firmware não converte o ADC em ppm.

`LUZ_INVERTIDA` em `main.py` está `True`. Se o texto do OLED inverter claro e escuro, troque para `False`.

## Fotos do protótipo

Bancada fotografada em 17/11/2026.

![Protoboard com ESP32, MQ-135, sensor de luz e DHT22](docs/prototipo-bancada.jpg)

DHT22, na caixa branca, e BMP280, na placa roxa.

![DHT22 e BMP280](docs/prototipo-sensores.jpg)

## Interfaces

No OLED, a tela muda a cada 3 s: temperatura e umidade, pressão, qualidade do ar, luminosidade. Sem o display, a leitura e a publicação continuam.

O GIF reúne quatro capturas do celular em 17/11/2026, às 19:47, no painel que mostra o JSON publicado por este firmware (`estacao-meteorologica-esp32.web.app`): indicadores, gráfico de temperatura e umidade, pressão e mapa com a distância vetorial.

![Painel no celular em 17/11/2026](docs/interface-painel.gif)

## Operação

O laço usa `time.ticks_ms()`:

- lê os sensores quando passam mais de 2 s;
- monta o JSON e tenta publicar quando passam mais de 5 s;
- troca a tela do OLED quando passam mais de 3 s;
- se o MQTT caiu, tenta o Wi-Fi e o broker de novo após 10 s.

Há uma pausa de 100 ms no fim da volta para o laço não girar sem intervalo.

Publicação MQTT: broker `broker.hivemq.com`, porta `1883`, tópico `universidade/sensores/estacao`, sem usuário nem senha. O broker é público. A leitura aceita também segue para o Firebase Realtime Database do projeto, nos caminhos `estacao/atual` e `estacao/historico`.

Formato da mensagem. Os seis primeiros valores são os que o painel mostrou na leitura das 19:47:17 de 17/11/2026. O `timestamp` sai em UTC depois que o NTP (`pool.ntp.org`) responde.

```json
{
  "dht22_temp": 30.2,
  "dht22_umi": 65.1,
  "pressao_hpa": 928.8,
  "mq135_valor": 1228,
  "mq135_status": "Ar Moderado",
  "luminosidade_estado": "Claro",
  "timestamp": "AAAA-MM-DDTHH:MM:SSZ"
}
```

A publicação só ocorre com leitura válida do BMP280 e com estas conferências:

- temperatura entre −40 °C e 80 °C;
- umidade entre 0% e 100%;
- pressão entre 300 hPa e 1100 hPa;
- ADC do MQ-135 entre 0 e 4095, com rótulo igual ao calculado pelos limites;
- luz só `Claro` ou `Escuro`;
- temperatura e umidade zeradas ao mesmo tempo são recusadas;
- salto acima de 5 °C ou de 20 pontos de umidade entra só se a leitura seguinte confirmar o novo valor.

## Arquivos gravados na placa

| Arquivo | Função |
|---|---|
| `boot.py` | Sobe com a energia e passa o controle ao `main.py` |
| `main.py` | Sensores, OLED, conferência, MQTT e Firebase |
| `ssd1306.py` | Driver do display |
| `wifi_config.py` | Redes Wi-Fi, criado a partir do exemplo e mantido só na placa |
