---
layout: single
title: "DigitalGearKnob v1 — Cambio con IMU"
date: 2026-10-15
categories: [hardware]
github_repo: https://github.com/Scuffy-sh/DigitalGearKnob
header:
  teaser: https://raw.githubusercontent.com/Scuffy-sh/DigitalGearKnob/main/images/knob-open.jpeg
summary: "Prototipo v1: detección de marcha con IMU BNO055 montado en la palanca, calibración por marcha guardada en NVS y limitación de deriva por ángulo de montaje."
excerpt: "Prototipo v1 con IMU BNO055 montado en la palanca de cambios. La marcha se detecta por orientación; la calibración por marcha persiste en NVS. Queda la deriva por ángulo de montaje como limitación conocida."
---

# DigitalGearKnob v1 — Cambio con IMU

## Resumen

DigitalGearKnob es un proyecto de electrónica embarcada para mostrar y leer la marcha seleccionada en el cambio de un vehículo. La **v1** es un prototipo con **IMU BNO055** montado directamente en la palanca de cambios: la marcha se infiere a partir de la orientación y se calibra por posición (marcha) guardando los valores en NVS.

## Características principales

- **IMU BNO055**: sensor de orientación montado en la palanca de cambios.
- **Detección por orientación**: la marcha se identifica según el ángulo/orientación del conjunto.
- **Calibración por marcha**: para cada marcha se guarda un offset/valor de referencia en NVS; se cargan al arrancar y permiten reprocesar contra ese punto de referencia.
- **Persistencia en NVS**: los datos de calibración se almacenan en memoria no volátil para conservar la calibración entre encendidos.
- **Estado legado**: el código bajo `bno/` queda **read-only** en el repositorio (mantiene la línea histórica del prototipo IMU).

## Limitaciones conocidas

- **Deriva por ángulo de montaje (mounting-angle drift)**: el acople mecánico/geométrico entre palanca e IMU introduce una deriva del ángulo de montaje que afecta la precisión de la detección entre re-montajes o tras vibraciones. Es la principal limitación del enfoque v1.
- **Calibración manual por marcha**: requiere recorrer las marchas para establecer los puntos de referencia.
- **Dependiente de orientación**: sensible a cambios de montaje respecto al chasis.

## Evolución

La **v2** evoluciona el enfoque hacia lectura CAN/TWAI con un estimador RPM÷velocidad más robusto, interfaz LVGL y arquitectura dual (NimBLE + app Flutter). Ver entrada **[DigitalGearKnob v2](/projects/2026-10-15-digital-gear-knob-v2.html)**.

## Enlaces

- [Repositorio GitHub](https://github.com/Scuffy-sh/DigitalGearKnob)
- [DigitalGearKnob v2](/projects/2026-10-15-digital-gear-knob-v2.html)