
flowchart LR
    %% Stili dei nodi
    classDef isr fill:#ffe5e5,stroke:#c0392b,stroke-width:2px,color:#000
    classDef task fill:#e8f4f8,stroke:#2980b9,stroke-width:2px,color:#000
    classDef ipc fill:#fef9e7,stroke:#d4ac0d,stroke-width:2px,stroke-dasharray: 4 4,color:#000
    classDef hw fill:#eaeded,stroke:#566573,stroke-width:1.5px,color:#000

    %% Hardware esterno / Core M0+
    subgraph HW ["Hardware & Radio Core (Cortex-M0+)"]
        direction TB
        M0["Cortex-M0+ BLE Stack<br/>(Mailbox IPCC)"]:::hw
        IMU["ICM-20602 (SPI)<br/>Pin INT1 @ 100 Hz"]:::hw
        MODEM["Quectel BG95-M5<br/>(UART 1.8V via Level Shifter)"]:::hw
    end

    %% Interrupt Service Routines (ISR)
    subgraph ISR ["Interrupt Service Routines (ISR - Cortissime)"]
        direction TB
        ISR_BLE(["IPCC_RX_ISR<br/>(Priorità NVIC Alta)"]):::isr
        ISR_IMU(["EXTI_INT1_ISR<br/>(Data Ready IMU)"]):::isr
        ISR_DMA(["SPI_DMA_RxCplt_ISR<br/>(Fine lettura 6-assi)"]):::isr
        ISR_UART(["UART_IDLE_DMA_ISR<br/>(Fine riga AT / URC)"]):::isr
    end

    %% Oggetti IPC FreeRTOS
    subgraph IPC ["Primitive IPC & Sincronizzazione FreeRTOS"]
        direction TB
        SEM_BLE{{"Binary Semaphore<br/>xBleEventSem"}}:::ipc
        NOTIF_IMU{{"Task Notification<br/>ulTaskNotifyTake"}}:::ipc
        Q_TEL[("xQueue: TelemetryQueue<br/>(Struct: GPS, IMU, Stato)")]:::ipc
        SB_AT[("xStreamBuffer: AtRxBuffer<br/>(Byte grezzi UART BG95)")]:::ipc
        EG_SYS{{"EventGroup: SysFlags<br/>NET_OK | CRASH | IGN_ON"}}:::ipc
        MTX_CFG{{"Mutex: DeviceStateMutex<br/>(Config & Parametri) "}}:::ipc
    end

    %% Task Applicativi (Cortex-M4)
    subgraph TASKS ["Task Applicativi FreeRTOS (Cortex-M4)"]
        direction TB
        T_BLE["BleHostTask<br/>Priorità: High (4)<br/>Stack: 512 W"]:::task
        T_IMU["ImuSensorTask<br/>Priorità: AboveNormal (3)<br/>Stack: 256 W"]:::task
        T_MODEM["ModemFsmTask<br/>Priorità: Normal (2)<br/>Stack: 1024 W"]:::task
        T_PWR["Idle / PowerMgr Hook<br/>Priorità: Idle (0)<br/>Tickless STOP2 Mode"]:::task
    end

    %% Collegamenti Hardware -> ISR
    M0 --> ISR_BLE
    IMU --> ISR_IMU
    IMU -. "DMA Transfer" .-> ISR_DMA
    MODEM -. "DMA Circular" .-> ISR_UART

    %% Collegamenti ISR -> IPC
    ISR_BLE -- "xSemaphoreGiveFromISR" --> SEM_BLE
    ISR_IMU -- "Avvia SPI DMA" --> IMU
    ISR_DMA -- "vTaskNotifyGiveFromISR" --> NOTIF_IMU
    ISR_UART -- "xStreamBufferSendFromISR" --> SB_AT

    %% Collegamenti IPC <-> Task
    SEM_BLE --> T_BLE
    NOTIF_IMU --> T_IMU

    T_IMU -- "xQueueSend (Eventi/Log)" --> Q_TEL
    T_IMU -- "xEventGroupSetBits (Urto)" --> EG_SYS

    Q_TEL -- "xQueueReceive" --> T_MODEM
    SB_AT -- "xStreamBufferReceive" --> T_MODEM
    EG_SYS <--> T_MODEM

    T_BLE <--> MTX_CFG
    T_MODEM <--> MTX_CFG
    T_BLE <--> EG_SYS
