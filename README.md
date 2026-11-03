# RA-3.2_SamuelSabogal

## 1. Descripción General y Objetivo
Este proyecto implementa un **Gemelo Digital (Digital Twin)** para el monitoreo, control y supervisión del nivel de líquido en un tanque industrial. El sistema integra lógica de control programada bajo el estándar IEC 61131-3 (Diagrama Ladder en **CODESYS**), una interfaz humano-máquina (**HMI local**), un puente de integración **Software-in-the-Loop (Python)** y un panel de telemetría y mando remoto en la nube (**Ubidots IoT**).

---

## 2. Arquitectura del Sistema
El sistema se divide en tres capas sincronizadas en tiempo real:

1. **Capa de Control y Simulación Local (CODESYS V3.5 SP22 Patch 3):** Ejecuta el PLC virtual (`CODESYS Control Win V3 x64`), procesando las señales de los sensores de nivel (`B1`, `B2`, `B3`) para activar los actuadores e indicadores luminosos (`H1` a `H5`), reflejados en una HMI local (`Visualization`).
2. **Capa de Integración / Middleware (Python `tanque.py`):** Script de enlace que emula la pasarela de comunicación hacia la nube mediante peticiones HTTP REST (`POST` y `GET`) hacia los endpoints de la API de Ubidots STEM.
3. **Capa de Supervisión Cloud (Ubidots):** Dashboard interactivo que grafica el porcentaje de llenado del tanque en tiempo real, alerta condiciones críticas o fallos de sensores y permite accionar el mando remoto (`control_remoto`).

---

## 3. Lógica de Control y Tabla de Verdad (CODESYS)

### Variables del Sistema (`PLC_PRG`)
* **Entradas (Sensores de Nivel):**
  * `B1` (`BOOL`): Sensor de nivel bajo (33%).
  * `B2` (`BOOL`): Sensor de nivel medio (66%).
  * `B3` (`BOOL`): Sensor de nivel alto / lleno (100%).
* **Salidas (Indicadores y Alarmas):**
  * `H1` (`BOOL`): Indicador de **Nivel Medio**.
  * `H2` (`BOOL`): Indicador de **Nivel Bajo**.
  * `H3` (`BOOL`): Indicador de **Tanque Lleno / Alarma de Tope**.
  * `H4` (`BOOL`): Indicador de **Tanque Vacío**.
  * `H5` (`BOOL`): **Alarma de Fallo en Sensores** (incoherencia física si se activa un sensor superior sin haberse activado el inferior).

### Tabla de Estados yEcuaciones Lógicas

| Estado del Tanque | Nivel (%) | `B3` (Alto) | `B2` (Medio) | `B1` (Bajo) | Salida Activa | Ecuación Lógica (Ladder) |
| :--- | :---: | :---: | :---: | :---: | :---: | :--- |
| **Vacío** | 0% | 0 | 0 | 0 | **`H4`** | `/B3 AND /B2 AND /B1` |
| **Nivel Bajo** | 33% | 0 | 0 | 1 | **`H2`** | `/B3 AND /B2 AND B1` |
| **Nivel Medio** | 66% | 0 | 1 | 1 | **`H1`** | `/B3 AND B2 AND B1` |
| **Tanque Lleno** | 100% | 1 | 1 | 1 | **`H3`** | `B3 AND B2 AND B1` |
| **Error de Sensores** | N/A | 1 | 0 | X | **`H5`** | `(B3 AND /B2) OR (B2 AND /B1)` |
| **Error de Sensores** | N/A | X | 1 | 0 | **`H5`** | `(B3 AND /B2) OR (B2 AND /B1)` |

### Diagrama de Contactos (Ladder `PLC_PRG`)
<img width="1122" height="705" alt="image" src="https://github.com/user-attachments/assets/a306ce0c-3e07-487e-90b5-fd8a3f65d877" />


---

## 4. Interfaz HMI Local (CODESYS Visualization)
Se diseñó una interfaz gráfica directamente en el entorno **Visualization** de CODESYS para operar el gemelo digital de manera local. Cuenta con:
* **3 Interruptores (Switches):** Vinculados a `PLC_PRG.B1`, `PLC_PRG.B2` y `PLC_PRG.B3` para simular la activación progresiva de los sensores del tanque.
* **5 Lámparas Indicadoras (LEDs):** Vinculadas a `PLC_PRG.H1` hasta `PLC_PRG.H5` para visualizar el estado activo y las alarmas.
* **Representación Visual del Tanque:** Bloques dinámicos que cambian de color a medida que el nivel de agua sube desde el estado vacío hasta el 100%.
<img width="1072" height="586" alt="image" src="https://github.com/user-attachments/assets/7bfcde2f-f0b6-4a71-9c7e-36f9bef2ea5f" />


---

## 5. Puente de Integración IoT (`tanque.py`)
Para enlazar la lógica validada en CODESYS con la nube de Ubidots, el script `tanque.py` estructura un payload JSON con el estado de los sensores, las salidas lógicas y el porcentaje de nivel, transmitiéndolo mediante el protocolo HTTP REST:

* **Endpoint:** `https://industrial.api.ubidots.com/api/v1.6/devices/<DEVICE_LABEL>`
* **Transmisión (Uplink):** Envía las variables de estado (`nivel`, `b1`, `b2`, `b3`, `h1`..`h5`) en cada cambio de nivel.
* **Recepción (Downlink):** Consulta en tiempo real la variable `control_remoto` desde Ubidots para ejecutar comandos desde el Dashboard en la nube hacia el sistema local.
---

## 6. Supervisión en la Nube y Validación (Dashboard Ubidots)
El Dashboard en **Ubidots** refleja de forma sincronizada el comportamiento del gemelo digital en todos sus escenarios operativos:

1. **Monitoreo de Nivel:** Widget de tanque que transiciona entre 0%, 33%, 66% y 100%.
2. **Panel de Alarmas:** Indicadores visuales para estado vacío (`H4`), lleno (`H3`) y fallo de instrumentación (`H5`).

<img width="1917" height="1078" alt="image" src="https://github.com/user-attachments/assets/2e50ee3c-cc26-469a-84cf-e2d424bbb352" />


## 7. Limitaciones
Para esta entrega del ra, no conte con un microcontrolador, por ende la entrega fisica solo cuenta como un representacion fisica de como seria el montaje fisico, entiendo que esto puede acarrear la perdida de puntos, pero preferi hacer el script anteriormente mencionado para hacer la entrega y no quedar con la nota en 0, apesar de no contar con la conexion del microcontrolador con el codesys, con este trabajo pude entender la conexion con ubidots para la entrega del proyecto final, igualmente reitero que la entrega de este ra es de muy bajo nivel teniendo en cuenta la intancia en la que nos encontramos, pero yo entiendo que la forma en la que se resolvio el problema y la forma en la que se ve el flujo del sistema quedo claro, muchas gracias.

---

## 8. Conclusiones
Se pudo hacer un acercamiento a como seria el comportamiento del gemelo digital del tanque, contando con una visualizacion del tanque tanto como en el codesys como en el ubidots, haciendo asi que se tenga un control preciso del nivel del tanque, al no tener montaje fisico ni la conexion como se quisiera, no se puede tener en cuenta este sistema como un sistema funcional, pero si como una muy buena base para el futuro montaje del sistema
