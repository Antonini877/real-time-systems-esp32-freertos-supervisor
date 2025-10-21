#  ESP32 FreeRTOS Supervisor System

Sistema multitarefa para ESP32 usando FreeRTOS. O projeto divide responsabilidades em três módulos principais: gerador de dados, receptor e supervisor com um watchdog cooperativo que reinicia o dispositivo em caso de falha de tarefas críticas.

---

## Sumário

- [Visão Geral](#visão-geral)  
- [Arquitetura](#arquitetura)  
- [Tarefas Principais](#tarefas-principais)  
- [Watchdog Cooperativo](#watchdog-cooperativo)  
- [Parâmetros Principais](#parâmetros-principais)  
- [Como Executar](#como-executar)  
- [Extensões Futuras](#extensões-futuras)

---

## Visão geral

O sistema demonstra comunicação entre tarefas com FreeRTOS (filas, event groups) e um supervisor que monitora a saúde das tarefas. Caso uma tarefa crítica pare de "alimentar" o watchdog, o supervisor força um reinício via `ESP.restart()`.

---

## Arquitetura

Gerador -> Queue (buffer de 5 ints) -> Receptor  
Supervisor monitora Gerador e Receptor (recebe feeds de tick count)

Diagrama simplificado:

                   +----------------------------------+
                   |      Supervisor (vTaskSupervisao)|
                   |    - Monitora tarefas            |
                   |    - Gerencia Watchdog           |
                   +----------------------------------+
                          ^                    ^
                          |                    |
                          |                    |
        +-----------------+                    +-----------------+
        |                                                      |
        |                                                      |
+---------------------------+          +---------------------------+
|  Gerador (vTaskGeracao)   |   --->   |  Queue (Buffer de Dados)  |   --->   |  Receptor (vTaskRecepcao) |
|  - Gera inteiros          |          |  - Capacidade: 5 itens    |          |  - Transmite e valida      |
|  - Alimenta watchdog      |          |  - Comunicação entre tasks|          |  - Reporta falhas          |
+---------------------------+          +---------------------------+          +---------------------------+
        \                                                                 /
         \---------------------------------------------------------------/
                             Comunicação com Supervisor


---

## Tarefas principais

- vTaskGeracaoDados (Gerador)
  - Gera inteiros sequenciais.
  - Envia para uma Queue (capacidade configurável).
  - Descarta valores se a fila estiver cheia.
  - Alimenta o watchdog cooperativo periodicamente.

- vTaskRecepcaoDados (Receptor)
  - Lê valores da fila e "transmite" via Serial.
  - Implementa recuperação com tentativas (3 tentativas após timeouts).
  - Em falha crítica, sinaliza instabilidade e permite reinício pelo watchdog.
  - Atualiza o supervisor sobre seu status.

- vTaskSupervisao (Supervisor)
  - Monitora os últimos feeds das tarefas.
  - Loga estado das tarefas e detecta timeout do watchdog.
  - Executa `ESP.restart()` se um feed ultrapassar `WDT_TIMEOUT_MS`.

---

## Watchdog cooperativo

Cada tarefa crítica atualiza uma variável global com o valor de `xTaskGetTickCount()` (ou equivalente). O supervisor compara o tempo atual com o último feed:

- Se tempo desde último feed > WDT_TIMEOUT_MS → log + `ESP.restart()`.
- Caso contrário → apenas log de saúde.

Observação: para produção recomenda-se usar o WDT nativo do ESP-IDF:
`esp_task_wdt_init()`, `esp_task_wdt_add()`, etc.

---

## Parâmetros principais

| Parâmetro | Descrição | Valor padrão |
|-----------|-----------:|:------------:|
| TAMANHO_FILA | Capacidade da fila de comunicação | 5 |
| TEMPO_LIMITE_RECEPCAO_MS | Timeout máximo de recepção | 5000 ms |
| WDT_TIMEOUT_MS | Timeout do watchdog cooperativo | 3000 ms |

---

## Como executar

Pré-requisitos:
- Placa ESP32 (ex: DevKit)
- Arduino IDE ou PlatformIO com Core ESP32

Passos:
1. Abra o projeto no Arduino IDE ou PlatformIO.
2. Selecione: Placa → ESP32 Dev Module, Baud → 115200.
3. Faça upload para a placa.
4. Abra o Monitor Serial (115200).

Exemplo de saída:
- Normal:
  - Gerador: enviado 1
  - Receptor: Recebido 1 -> transmitindo (simulado)
  - Gerador OK (último feed 102 ms)
  - Receptor OK (último feed 105 ms)

- Em caso de falha:
  - Receptor: FALHA CRÍTICA! max tentativas esgotado.
  - WDT: Receptor nao alimentado por 3100 ms -> REBOOT

Comando para clonar (exemplo):
```bash
git clone https://github.com/<seu-usuario>/esp32-freertos-supervisor-system.git
```

---

## Extensões futuras

- Substituir watchdog cooperativo pelo WDT real do ESP-IDF.
- Reinicialização de tarefas individuais sem reboot global.
- Registro de logs em NVS (memória não volátil).
- Monitoramento remoto via Wi‑Fi / MQTT.

---
