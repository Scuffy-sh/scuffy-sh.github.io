---
layout: single
title: "IEEE-CIS Fraud Detection: Sistema de Detección de Fraude"
date: 2026-04-07
tags: [python, pandas, scikit-learn, LightGBM, XGBoost, feature engineering, streamlit]
github_repo: https://github.com/Scuffy-sh/IEEE-CIS-Fraud-Detection
excerpt: "Sistema de machine learning para detectar transacciones fraudulentas en tiempo real con AUC-ROC de 0.924."
---

# 🔒 IEEE-CIS Fraud Detection: Sistema de Detección de Fraude

## 📋 Resumen Ejecutivo

Sistema de machine learning para predecir transacciones fraudulentas en la plataforma de pagos de Vesta Corporation. El modelo genera predicciones probabilísticas para detectar fraude en tiempo real, con un AUC-ROC de 0.924 en validación.

**Stack tecnológico**: Python, pandas, scikit-learn, LightGBM, XGBoost, Streamlit  
**Duración del proyecto**: 3 días  
**Repositorio**: [GitHub]({{ page.github_repo }})

---

## 🎯 Problema de Negocio

La detección de fraude en transacciones online presenta desafíos únicos debido al desbalance de clases (~3.5% de fraudes), la alta dimensionalidad de los datos (400+ features) y la necesidad de tiempos de respuesta rápidos.

**Métrica principal**: AUC-ROC (mayor es mejor)  
**Target**: Probabilidad de fraude para cada transacción

---

## 📊 Dataset

| Origen | Descripción | Registros |
|--------|-------------|-----------|
| Kaggle | Competición IEEE-CIS Fraud Detection | 590,540 |
| train_transaction.csv | Transacciones de entrenamiento | 590,540 |
| train_identity.csv | Información de identidad | 144,233 |
| test_transaction.csv | Transacciones de test | 506,691 |
| test_identity.csv | Identidades de test | 141,481 |

**Rango temporal**: Datos temporales con gap entre train y test  
**Features utilizadas**: ~300 (V columns, C columns, D columns, cards, addr, etc.)

---

## ⚙️ Pipeline de Machine Learning

### 1. Feature Engineering

| Feature | Descripción | Tipo |
|---------|-------------|------|
| Merge | Unión de transactions + identity por TransactionID | Join |
| Label Encoding | Codificación de categorías de baja cardinalidad | Categórico |
| Target Encoding | Encoding con CV para evitar leakage | Numérico |
| Imputation | Fillna con valores por defecto (-999, 'missing') | Numérico/Cat |
| Drop Columns | Eliminación de columnas con >80% missing | Reducción |

### 2. Modelado

**Modelo principal**: LightGBM (Baseline v1.0)

```
Parámetros optimizados:
- num_leaves: 255
- learning_rate: 0.05
- min_child_samples: 50
- subsample: 0.8
- colsample_bytree: 0.8
- scale_pos_weight: 27.46 (para desbalance)
- early_stopping_rounds: 100
```

**Modelo ensemble**: LightGBM + XGBoost (v1.1)

**Estrategia de validación**: Time-based split (80% train, 20% validation)

### 3. Arquitectura

```
Input (300 features)
    ↓
Preprocesamiento (FraudPreprocessor)
    ↓
LightGBM / XGBoost
    ↓
Predicción de probabilidad
    ↓
Threshold (0.5) → Binary prediction
```

---

## 📈 Resultados

### Métricas de Validación

| Métrica | Baseline v1.0 | Ensemble v1.1 |
|---------|----------------|---------------|
| AUC-ROC | 0.9240 | 0.9161 |
| PR-AUC | 0.5878 | 0.5582 |
| Best Iteration | 185 | 1000 |

### Comparación con Ganador Kaggle

| Métrica | Tu Proyecto | 1er Lugar | Diferencia |
|---------|-------------|-----------|------------|
| AUC-ROC (val) | 0.924 | 0.9459 | -2.3% |

---

## 🔒 Validación de Data Leakage

Se verificó exhaustivamente que no exista data leakage:

- ✅ Time-based split (validation más reciente que training)
- ✅ Target encoding con CV (5 folds)
- ✅ No se usa información futura en features

---

## 💡 Limitaciones

1. **Sin UID features**: No se implementó el sistema de Unique ID del ganador
2. **Sin CatBoost**: Solo LightGBM + XGBoost
3. **Sin postprocessing**: No se usa average por cliente UID
4. **Features limitadas**: No se incluyen aggregaciones por grupo

---

## 🚀 Mejoras Futuras (del Ganador)

1. **Feature engineering avanzado**:
   - Crear UID (card1 + addr1 + D1) para identificar clientes
   - Agregaciones por grupo (mean, std de TransactionAmt por UID)
   - Features de agregación: TransactionAmt_UID_mean, D9_UID_mean, etc.

2. **Modelo ensemble completo**:
   - Añadir CatBoost al ensemble
   - Postprocessing: reemplazar predicciones con promedio del cliente

3. **Validación más robusta**:
   - GroupKFold con meses como grupos
   - GPU acceleration (RAPIDS)

---

## 🛠️ Tech Stack

- **Lenguaje**: Python 3.13
- **Data**: pandas, numpy
- **ML**: scikit-learn, LightGBM, XGBoost
- **Dashboard**: Streamlit
- **Testing**: pytest (77 tests passing)

---

## 📁 Estructura del Proyecto

```
IEEE-CIS-Fraud-Detection/
├── src/
│   ├── eda/                    # Análisis exploratorio
│   │   ├── analyze.py
│   │   └── visualize.py
│   ├── features/               # Feature engineering
│   │   ├── merge.py
│   │   ├── encode.py
│   │   ├── impute.py
│   │   └── preprocess.py
│   ├── models/                 # Modelos
│   │   ├── train.py
│   │   ├── ensemble.py
│   │   └── inference.py
│   ├── pipeline/               # Pipeline completo
│   │   └── inference.py
│   └── utils/                  # Utilidades
│       ├── config.py
│       └── logging.py
├── tests/                      # Tests (77 passing)
├── streamlit_app.py           # Dashboard web
├── config.yaml                # Configuración
├── CHANGELOG.md               # Registro de cambios
├── COMPARACION_KAGGLE.md       # Comparativa con ganador
└── PROJECT_SUMMARY.md         # Este documento
```

---

## 📊 Dashboard

El proyecto incluye un dashboard Streamlit con:

- 📈 **Métricas**: AUC-ROC, PR-AUC, iteraciones óptimas, distribución de probabilidades
- 🔮 **Simulador**: Predicción interactiva con preguntas simples
- 📜 **Modelos**: Comparación de versiones (v1.0 vs v1.1)
- 📁 **Análisis**: Análisis de lotes con gráficos de riesgo

---

## 🔗 Enlaces

- **Repositorio**: [GitHub]({{ page.github_repo }})
- **Dashboard**: `streamlit run streamlit_app.py`
- **Tests**: `pytest -v`
