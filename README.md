# melipilla-iot-apr-9-Medición-de-humedad-de-suelo-y-riego-automatico
Un sensor de humedad de suelo mide en tiempo real. Si la humedad cae por debajo de un umbral (según cultivo), se activa una electroválvula o bomba para regar hasta alcanzar el rango deseado, con Raspberry Pi para medir humedad de suelo, automatizar el riego y que además adapte la estrategia según el cultivo (umbral, método y ventana horaria)



# 1) Qué vas a medir y por qué

* **Variable clave:** Humedad volumétrica del suelo (VWC, % o m³/m³).
* **Decisión de riego:** Regar cuando el suelo baje de un **umbral** entre *capacidad de campo (θfc)* y *punto de marchitez (θwp)*, según el **porcentaje de agotamiento permisible (p)** del cultivo.
* **Fórmulas útiles (zona radicular Zr en m):**

  * **TAW (mm)** = 1000·(θfc − θwp)·Zr
  * **RAW (mm)** = p · TAW (agua que puedes “consumir” antes de regar)

> Regar cuando la humedad medida corresponda a una **depleción ≈ RAW**.

# 2) Arquitectura recomendada (dos variantes)

## A) “Sencilla y robusta” (invernadero/parcelas medianas)

* **Controlador:** ESP32 o PLC compacto (Si deseas Wi-Fi y dashboard sencillo → ESP32).
* **Sensores de humedad:** Capacitivos tipo 0–3 V o SDI-12 (p. ej., Decagon/METER EC-5/10HS) — evita resistivos baratos por drift.
* **Actuación:** Electroválvulas 24 VAC (solenoide) + **relé de estado sólido** (SSR) o módulo triac para cada zona.
* **Alimentación:** 220 VAC → fuente 24 VAC para válvulas + 5/12 V DC para electrónica (fuente con margen 30%).
* **Hidráulica:** Filtro, regulador de presión, caudalímetro en cabecera (opcional pero útil), válvulas por sector.
* **Comunicaciones:** Wi-Fi local; datalog en microSD o en la nube (InfluxDB/ThingsBoard opcional).

## B) “Escalable y a campo abierto” (lotes grandes)

* **Nodos** LoRa/LoRaWAN a batería con 2–3 sondas por sector.
* **Gateway** LoRaWAN + servidor (ChirpStack/The Things Stack).
* **Control central** (PLC/RTU) acciona bombas/válvulas según datos del servidor.
* **Ventajas:** Bajo consumo, gran alcance; **Contras:** mayor complejidad inicial.

# 3) Componentes (BOM orientativo)

* 1× **Controlador** (ESP32 DevKit C o PLC)
* 2–3× **Sensores de humedad por zona** (a distintas profundidades: 1/3 y 2/3 de Zr)
* n× **Electroválvulas 24 VAC** (una por sector)
* n× **SSR/triac** para 24 VAC, con optoaislamiento
* 1× **Fuente 24 VAC** (≥ VA total de solenoides) + 1× **Fuente 5/12 V DC**
* **Caja IP65**, prensaestopas, conectores impermeables, tubo corrugado
* (Opcional) **Caudalímetro** (pulsos) y **presostato** para protección de bomba
* (Opcional) **Pluviómetro** y **sensor de temperatura** (heladas)

# 4) Colocación y calibración de sondas

* **Por zona:** mínimo 2 sondas; 3 si el suelo es heterogéneo.
* **Profundidad:** una a 1/3 de Zr y otra a 2/3 de Zr (p. ej., Zr=30 cm → 10 cm y 20 cm).
* **Ubicación** entre goteros/emisores, a media distancia de la planta para evitar “picos” de agua directa.
* **Calibración rápida:**

  1. Riega a capacidad de campo; anota lectura → aproxima **θfc**.
  2. Deja secar hasta observar leve estrés o fin del riego por varios días; anota → aproxima **θwp** (mejor si tienes valores de laboratorio).
  3. Ajusta curva sensor→VWC si el fabricante lo permite.
* **Promediado:** usa promedio o mediana de sondas por zona para decidir.

# 5) Umbrales prácticos por tipo de cultivo (referencia)

* **Hortícolas de hoja (p≈0,25–0,35):** mantener 70–80% de TAW disponible → umbral cercano a 20–30% depleción.
* **Frutales (p≈0,35–0,5 según estado fenológico):** 40–50% de TAW.
* **Berries (sensibles):** 25–35% de TAW.

> Empieza conservador (regar “antes”) y ajusta con datos de rendimiento y calidad.

# 6) Lógica de control (histeresis + ventanas de riego)

**Condiciones para regar una zona:**

1. Humedad promedio **< Umbral\_ON** (ej., 30% TAW restante o VWC equivalente).
2. **Ventana horaria** permitida (p. ej., 05:00–09:00 y 19:00–22:00).
3. **Interlocks OK:** presión OK, caudal dentro de rango, no llueve, sin alarma de depósito vacío.

**Condiciones para parar:**

* Humedad **> Umbral\_OFF** (umbral\_ON + 2–4% VWC de histéresis),
* o tiempo máximo de riego alcanzado (failsafe),
* o caudal anómalo (fuga/rotura).

# 7) Cálculo rápido de tiempo de riego

1. **Agua a reponer (mm)** ≈ **RAW consumido** (o hasta volver a θfc).
2. **Lámina neta (mm)** = Agua a reponer.
3. **Lámina bruta (mm)** = Lámina neta / **Eficiencia** (p. ej., 0,9 goteo).
4. **Tiempo (h)** = Lámina bruta (mm) × **Área (m²)** / **Caudal total (L/h)**

> 1 mm = 1 L/m²

**Ejemplo:** Parcela 200 m², goteo 600 L/h total, reponer 6 mm, eficiencia 0,9

* Bruta = 6/0,9 = 6,67 mm → 6,67 L/m²
* Volumen = 6,67×200 = 1334 L
* Tiempo = 1334/600 ≈ **2,22 h**


# 8) Seguridad y fiabilidad

* **Separación eléctrica:** 24 VAC de válvulas aislada de la lógica (optos SSR).
* **Sellado IP65** y alivio de tensión en cables de sensores.
* **Failsafes:** tiempo máx. por riego, paro por falta de presión o caudal cero, watchdog en el microcontrolador.
* **Mantenimiento:** limpiar filtro, test de válvulas mensual, recalibrar sondas por temporada.

# 9) Integraciones “pro”

* **Lluvia/pronóstico:** si hay lluvia prevista > X mm, **posponer**.
* **Heladas:** si T° < umbral, riego antiheladas (si diseño lo permite).
* **Dashboard:** Grafana/ThingsBoard para ver VWC, caudal, aperturas, alarmas.
* **Alertas:** Telegram/WhatsApp via webhook cuando un sector no alcanza θfc dentro del tiempo esperado (posible obturación).



