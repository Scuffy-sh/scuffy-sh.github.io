---
layout: single
title: "DigitalGearKnob v2 — Cambio por CAN"
date: 2026-10-15
categories: [hardware]
github_repo: https://github.com/Scuffy-sh/DigitalGearKnob
header:
  teaser: https://raw.githubusercontent.com/Scuffy-sh/DigitalGearKnob/main/images/t-display-s3-amoled.jpg
summary: "Prototipo v2: lectura directa del bus CAN en modo escucha (TWAI listen-only a 500 kbps) con ESP32-S3 T-Display S3 AMOLED y transceptor SN65HVD230; estimador de marcha RPM÷velocidad con histéresis y debounce, UI LVGL y arquitectura BLE + app Flutter."
excerpt: "Prototipo v2 que lee el bus CAN del vehículo en modo solo-escucha (TWAI listen-only, 500 kbps) con ESP32-S3 T-Display S3 AMOLED y transceptor SN65HVD230. La marcha se estima con RPM÷velocidad (histéresis, debounce 3×200 ms, reglas N/parado/stale) y se muestra en UI LVGL; BLE NimBLE + app Flutter scuffy para estado en vivo."
---

# DigitalGearKnob v2 — Cambio por CAN

## Resumen

DigitalGearKnob es un proyecto de electrónica embarcada para mostrar y leer la marcha seleccionada en el cambio de un vehículo. La **v2** abandona el enfoque IMU de la v1 y pasa a **lectura directa del bus CAN** del vehículo: un **ESP32-S3 LilyGO T-Display S3 (AMOLED 1.43")** con transceptor **Waveshare SN65HVD230** escucha el tráfico CAN, estima la marcha activa y la muestra en pantalla, con estado en vivo vía BLE y una app Flutter complementaria.

## Hardware

- **ESP32-S3 LilyGO T-Display S3 (AMOLED 1.43")**: placa con pantalla AMOLED integrada, base de la UI LVGL.
- **Transceptor Waveshare SN65HVD230**: interfaz física CAN del ESP32-S3 hacia el bus del vehículo.
- **Sin path de transmisión**: ver seguridad CAN más abajo — el hardware y el firmware están pensados solo para escuchar.

## Seguridad CAN: TWAI listen-only a 500 kbps

- **TWAI (driver CAN del ESP32) en modo escucha a 500 kbps**, la velocidad típica del bus.
- **Modo listen-only**: el controlador está en modo escucha, **sin path de transmisión**; el nodo nunca inicia tramas ni domina el bus — **nunca puede perturbar el bus CAN** del vehículo.
- Esto es el requisito central de seguridad de la v2: el dispositivo observa, no participa.

## Estimador de marcha (RPM ÷ velocidad)

- El estimador calcula el **ratio RPM ÷ velocidad** a partir de los frames CAN y lo compara contra una **tabla por marcha** (ratio teórico de cada relación).
- **Histéresis**: la comparación contra la tabla usa histéresis para evitar saltos entre marchas vecinas en las zonas de transición.
- **Reglas de estado**:
  - **N / parado**: vehículo detenido o sin condiciones válidas → se muestra **N**.
  - **Stale**: si no llegan frames durante **500 ms** → se asume **N** (dato viejo, no se arriesga una marcha inventada).
- **Debounce 3×200 ms**: un cambio de marcha se confirma recién tras **3 detecciones consecutivas × 200 ms**.

## Interfaz: UI LVGL 8.3.11

- **LVGL 8.3.11** sobre el AMOLED del T-Display S3.
- **Animación de arco** para representar el cambio de marcha en pantalla.

## Conectividad: BLE + app Flutter

- **BLE NimBLE "SCUFFY"**: servicio/perfil BLE con nombre **SCUFFY**; publica el estado en vivo (marcha actual, etc.).
- **App Flutter "scuffy"**: aplicación móvil que consume el BLE para mostrar:
  - **estado en vivo** de la marcha,
  - **sniff CAN** (vista del tráfico observado),
  - **modo debug**,
  - **theming** de la interfaz.

## Desarrollo y calidad

- **Licencia**: MIT.
- **Tests Unity host-side**: tests unitarios Unity corriendo en host (sin hardware), para la lógica del estimador y el resto de la firmware.
- **Checklists bench/in-car**: verificación manual en banco y en el vehículo.

## Relación con v1

La v1 detectaba la marcha por orientación (IMU BNO055 en la palanca) con sus limitaciones de deriva y calibración manual; la v2 elimina esa dependencia leyendo el bus CAN en modo escucha. Ver entrada **[DigitalGearKnob v1](/projects/2026-10-15-digital-gear-knob-v1.html)**.

## Enlaces

- [Repositorio GitHub](https://github.com/Scuffy-sh/DigitalGearKnob)
- [DigitalGearKnob v1](/projects/2026-10-15-digital-gear-knob-v1.html)
