# 🛡️ Proyecto de Portafolio: Detección y Análisis de Vectores de Fraude Financiero

## 📌 1. Preguntar (Ask)
### Contexto del Proyecto
En el sector financiero, el fraude no solo representa pérdidas monetarias masivas, sino también una brecha crítica en la seguridad de la infraestructura y la confianza de los clientes. Este proyecto analiza un conjunto de datos con más de 6 millones de transacciones móviles simuladas para identificar patrones de comportamiento fraudulento y proponer medidas de mitigación de ciberseguridad.

### Objetivos de Negocio y Seguridad
*   *Identificar el Vector de Ataque:* Determinar a través de qué operaciones específicas los atacantes logran evadir los controles del sistema y extraer los fondos.
*   *Analizar el Patrón Temporal:* Descubrir si los ataques ocurren de forma manual o mediante procesos automatizados (scripts/bots) a lo largo de las 24 horas.
*   *Proponer Recomendaciones de Ciberseguridad (SIEM/MFA):* Diseñar reglas proactivas de mitigación basadas en datos empíricos para proteger el sistema de transacciones.

---

## 💾 2. Preparar (Prepare)
*   *Origen de los Datos:* El dataset utilizado es el Synthetic Financial Datasets for Fraud Detection, alojado en Kaggle.
*   *Características del Dataset:* Contiene $6.362.620$ de registros de transacciones financieras con variables como tipo de transacción (type), monto (amount), balance de origen/destino y una etiqueta binaria de fraude (isFraud).
*   *Almacenamiento y Gobernanza:* Siguiendo las mejores prácticas de gobernanza de datos y cumplimiento con el *RGPD (Reglamento General de Protección de Datos), los datos se alojaron en una base de datos externa en **Google BigQuery* bajo la multi-región *EU (Unión Europea)*.

---

## 🧹 3. Procesar (Process)
El análisis se ejecutó directamente en la consola de *Google BigQuery*. Durante esta fase, se realizaron tareas de:
*   Verificación de tipos de datos automáticos (esquema autodetectado).
*   Manejo y filtrado de transacciones inválidas.
*   Uso de operaciones aritméticas modulares (MOD) para aislar el factor temporal de la variable step.

---

## 📊 4. Analizar (Analyze)

### Consulta 1: Diagnóstico General del Impacto del Fraude
Objetivo: Medir la severidad y el ticket promedio del fraude detectado.

```sql
SELECT 
  COUNT(*) AS total_transacciones,
  SUM(CASE WHEN isFraud = 1 THEN 1 ELSE 0 END) AS total_fraudes,
  ROUND((SUM(CASE WHEN isFraud = 1 THEN 1 ELSE 0 END) / COUNT(*)) * 100, 4) AS porcentaje_fraude,
  ROUND(SUM(CASE WHEN isFraud = 1 THEN amount ELSE 0 END), 2) AS dinero_total_fraudulento,
  ROUND(AVG(CASE WHEN isFraud = 1 THEN amount ELSE 0 END), 2) AS promedio_por_fraude
FROM 
  daimielcloud1er.Ciberseguridad_Fraude.transacciones_financieras;

Hallazgo Clave:
Aunque el fraude solo representa el 0.1291% de las transacciones globales, el monto promedio sustraído por ataque es extremadamente alto: $1.467.967,30. Esto demuestra que los atacantes no realizan pequeños robos de manera masiva, sino ataques quirúrgicos de alto impacto dirigidos a cuentas con fondos elevados.

---

### Consulta 2: Identificación del Vector de Ataque (Tipos de Transacción)
Objetivo: Aislar en qué operaciones ocurre el fraude y cómo mueven los fondos.

```sql
SELECT 
  type AS tipo_transaccion,
  COUNT(*) AS total_transacciones,
  SUM(isFraud) AS cantidad_fraudes,
  ROUND(SUM(CASE WHEN isFraud = 1 THEN amount ELSE 0 END), 2) AS dinero_total_fraudulento,
  ROUND(AVG(CASE WHEN isFraud = 1 THEN amount ELSE 0 END), 2) AS promedio_por_fraude
FROM 
  daimielcloud1er.Ciberseguridad_Fraude.transacciones_financieras
GROUP BY 
  tipo_transaccion
ORDER BY 
  dinero_total_fraudulento DESC;

Hallazgo Clave (Vector de Ataque):
El fraude ocurre exclusivamente en transacciones de tipo TRANSFER y CASH_OUT. Esto revela el Modus Operandi de los ciberdelincuentes: comprometer las cuentas para realizar transferencias inmediatas a cuentas "puente" y, al mismo tiempo, realizar retiros en efectivo (CASH_OUT) para romper el rastreo digital del dinero.

---

### Consulta 3: Patrones Temporales y Detección de Botnets
Objetivo: Analizar la distribución de los ataques por horas.

SELECT 
  MOD(step, 24) AS hora_del_dia,
  COUNT(*) AS total_transacciones,
  SUM(isFraud) AS transacciones_fraudulentas,
  ROUND((SUM(isFraud) / COUNT(*)) * 100, 4) AS tasa_de_fraude_por_hora
FROM 
  daimielcloud1er.Ciberseguridad_Fraude.transacciones_financieras
GROUP BY 
  hora_del_dia
ORDER BY 
  tasa_de_fraude_por_hora DESC;

Hallazgo Clave (Automatización):
A las 5:00 AM y 4:00 AM, la tasa de transacciones fraudulentas se dispara exponencialmente a un 22.3% y 22% respectivamente. Sin embargo, el volumen total de fraudes se mantiene casi constante (un promedio de ~340 ataques por hora). Esto confirma el uso de scripts automatizados (bots) que operan de manera constante las 24 horas del día. El pico de riesgo de la madrugada se debe simplemente a que disminuye la actividad de los usuarios legítimos (humanos), convirtiéndola en la ventana horaria más crítica para el sistema.

