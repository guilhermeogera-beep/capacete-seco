# Capacete Seco Pro

Secador de capacete com gerador de ozônio, controlado por ESP32-C3 Mini via PWA instalável.

---

## Como funciona

O ESP32 opera em **modo híbrido**:

- **Configurado** (uso normal): conecta no Wi-Fi de casa. O celular fica na mesma rede, **com internet**, e acessa o secador em `http://secador.local`.
- **Não configurado / fora de casa**: se não encontrar a rede salva, abre o próprio Wi-Fi `SecadorCapacete` (modo AP) em `http://192.168.4.1`.

### Primeira configuração
1. Instale o PWA no celular.
2. Conecte no Wi-Fi `SecadorCapacete` (sem senha de internet).
3. Abra o app → **Configurar Secador** (ou acesse `http://192.168.4.1/wifi`).
4. Informe o nome e a senha do seu Wi-Fi de casa e salve. O ESP32 reinicia e conecta na rede.

### Uso no dia a dia
1. Mantenha o celular no Wi-Fi de casa (com internet).
2. Abra o app → **Conectar ao Secador** (vai para `http://secador.local`).
3. Selecione o ciclo e inicie. O ESP32 controla ventilador e ozônio pelos relês.
4. Um contador regressivo mostra o tempo restante; a página recarrega sozinha ao terminar.

---

## Hardware

### Lista de componentes

| Componente | Descrição |
|---|---|
| ESP32-C3 Mini | Microcontrolador principal (Wi-Fi AP + servidor HTTP) |
| Módulo relê 2 canais | Aciona ventilador e gerador de ozônio (lógica invertida: LOW = ON) |
| Ventilador 12V | Circula ar quente dentro do capacete |
| Gerador de ozônio | Purifica o interior do capacete |
| Fonte de alimentação | Alimenta ESP32 e os periféricos 12V |

### Pinagem

| Função | GPIO |
|---|---|
| Ventilador (Relê 1) | GPIO 4 |
| Gerador de ozônio (Relê 2) | GPIO 2 |

> O módulo relê usa **lógica invertida**: `LOW` liga o dispositivo, `HIGH` desliga.

---

## Estrutura do projeto

```
pwa secador 3/
├── index.html           # PWA — tela de instalação e conexão
├── manifest.json        # Manifesto do PWA (ícone, nome, display)
├── service-worker.js    # Cache offline da tela de instalação
├── icons/
│   ├── icon-192.png     # Ícone do app (192×192)
│   └── icon-512.png     # Ícone do app (512×512)
└── secador_Capacete/
    └── secador_Capacete.ino  # Firmware do ESP32-C3 Mini
```

---

## Ciclos de operação

### Ciclo 1 — Ventilador
Ventilador ligado continuamente pelo tempo selecionado (1–12 h).

### Ciclo 2 — Purificar
Ozônio pulsado durante o tempo selecionado:
- **Bloco de 10 min**: ozônio 30 s ligado / 5 s desligado (ventilador substitui nos 5 s)
- **Pausa de 3 min** entre blocos (tudo desligado)
- Repete até atingir o tempo total

### Ciclo 3 — Alternando
Alterna entre ventilação e ozônio enquanto o tempo total não esgota:
- **30 min** de ventilador contínuo
- **10 min** de ozônio pulsado (mesmo padrão 30 s/5 s do Ciclo 2)
- **Pausa de 3 min**
- Repete

### Ciclo 4 — Personalizado
Configura separadamente o tempo de ventilação e de ozônio:
- **Fase 1**: ventilador contínuo por H:MM (0:00–12:59). Se 0:00, pula direto para Fase 2.
- **Fase 2**: ozônio pulsado por H:MM total (0:00–5:59), dividido em blocos de 10 min com pausas de 3 min.

---

## Rede Wi-Fi

### Modo STA (conectado na rede de casa — uso normal)
| Parâmetro | Valor |
|---|---|
| Endereço | `http://secador.local` (mDNS) |
| IP | atribuído pelo roteador (DHCP) |
| Porta | 80 |

### Modo AP (configuração / sem rede de casa)
| Parâmetro | Valor |
|---|---|
| SSID | `SecadorCapacete` |
| Senha | `12345678` |
| IP do ESP32 | `192.168.4.1` |
| Porta | 80 |

> **mDNS (`secador.local`)** funciona nativamente no iOS (Safari) e no Android moderno (Chrome). Em alguns Android antigos o `.local` pode não resolver — nesse caso, descubra o IP do secador na página **Configurar Wi-Fi** (que mostra o IP atual) e acesse por ele, ou reserve um IP fixo no roteador.

---

## API HTTP do ESP32

### `GET /`
Retorna a interface de controle completa (HTML embutido no firmware).

### `GET /wifi`
Página para configurar o Wi-Fi de casa (SSID e senha).

### `GET /savewifi`
Salva as credenciais na EEPROM e reinicia o ESP32. Parâmetros: `s` (SSID), `p` (senha).

### `GET /set`
Configura e inicia um ciclo. Parâmetros:

| Parâmetro | Tipo | Intervalo válido | Descrição |
|---|---|---|---|
| `c` | int | 1–4 | Número do ciclo |
| `t` | int | 1–12 | Duração em horas (ciclos 1–3) |
| `v4h` | int | 0–12 | Ciclo 4: horas de ventilação |
| `v4m` | int | 0–59 | Ciclo 4: minutos de ventilação |
| `o4h` | int | 0–5 | Ciclo 4: horas de ozônio |
| `o4m` | int | 0–59 | Ciclo 4: minutos de ozônio |

Exemplo: `http://192.168.4.1/set?c=1&t=3` inicia o Ciclo 1 por 3 horas.

Após receber os parâmetros, o ESP32 redireciona para `/` com status 303.

---

## Mapa de memória EEPROM

| Endereço | Variável | Intervalo |
|---|---|---|
| 0 | Ciclo atual | 1–4 |
| 1 | Horas configuradas (ciclos 1–3) | 1–12 |
| 2 | Ciclo 4: horas de ventilação | 0–12 |
| 3 | Ciclo 4: minutos de ventilação | 0–59 |
| 4 | Ciclo 4: horas de ozônio | 0–5 |
| 5 | Ciclo 4: minutos de ozônio | 0–59 |
| 100 | Flag de Wi-Fi salvo (`0xA5` = sim) | — |
| 101–132 | SSID do Wi-Fi de casa | texto |
| 133–196 | Senha do Wi-Fi de casa | texto |

Valores fora dos intervalos válidos são corrigidos automaticamente na inicialização. EEPROM suporta ~100.000 ciclos de escrita.

---

## Comportamento na inicialização

O ESP32 inicia rodando automaticamente o último ciclo salvo na EEPROM ao ser ligado na tomada, sem necessidade de abrir a interface.

---

## Compilando e gravando o firmware

### Dependências (Arduino IDE)

- Board: **ESP32-C3** via `esp32` by Espressif (≥ 3.x)
- Bibliotecas inclusas no SDK (não precisa instalar nada): `WiFi`, `WebServer`, `ESPmDNS`, `EEPROM`, `esp_task_wdt`

### Configurações no Arduino IDE

| Campo | Valor |
|---|---|
| Board | ESP32C3 Dev Module |
| USB CDC On Boot | Enabled |
| Flash Size | 4MB |
| Partition Scheme | Default 4MB |

### Passos

1. Abra `secador_Capacete/secador_Capacete.ino` no Arduino IDE.
2. Selecione a porta COM do ESP32-C3.
3. Clique em **Upload**.

---

## Hospedando o PWA

O PWA (`index.html`, `manifest.json`, `service-worker.js`, `icons/`) precisa ser servido via **HTTPS** para que a instalação funcione em Android/Chrome.

Opções recomendadas:
- **GitHub Pages**: suba os arquivos em um repositório e ative Pages. Configure `start_url` e `scope` no `manifest.json` com o caminho correto (ex: `/capacete-seco/`).
- **Qualquer servidor HTTPS**: copie os arquivos para a raiz ou subpasta.

> O Service Worker só funciona em HTTPS (ou `localhost` para desenvolvimento).

---

## Limitações conhecidas

- A senha do **modo AP de configuração** (`12345678`) é fixa no firmware. Vale apenas durante a configuração inicial; no uso normal o secador fica na rede de casa.
- A senha do Wi-Fi de casa é guardada na EEPROM em texto puro. Para reconfigurar de uma rede para outra, basta abrir **Configurar Wi-Fi** novamente.
- `millis()` faz rollover após ~49,7 dias, mas a aritmética com `unsigned long` trata isso corretamente.
- O contador regressivo na interface é calculado no lado do cliente (JavaScript) e pode divergir alguns segundos do ESP32 se a página ficar aberta por muito tempo. Um reload manual sincroniza.
