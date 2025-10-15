## Análisis de costos del backend serverless

Se deben considerar por separado los costos de **procesamiento** y **almacenamiento**.  
En nuestra aplicación, los primeros corresponden a servicios como las funciones **AWS Lambda**, **Amazon Rekognition** y las consultas a **DynamoDB**, mientras que los segundos se asocian al uso de **S3** y al almacenamiento de datos en **DynamoDB**.

Para obtener un **costo mensual anualizado**, se calculó el promedio de usuarios activos al mes.  
Dado que existen meses de alta demanda como julio, diciembre, enero y febrero con **20.000 usuarios activos**, y el resto del año con **10.000 usuarios**, se obtiene un promedio mensual de **13.333 usuarios activos**.

### Supuestos de uso

De las consideraciones iniciales se establece que los viajes se distribuyen uniformemente a lo largo del mes, con **1 viaje por usuario**.  
En promedio, cada usuario realiza **10 posts por viaje**, y cada uno de estos presenta **5 imágenes** con un tamaño promedio de **4 MB**.

A partir de esto se analiza que:
- Cada usuario genera aproximadamente **50 imágenes al mes**, lo que implica **50 llamadas** a la función Lambda de recepción.  
- Considerando los **13.333 usuarios activos mensuales**, se obtienen **666.650 llamadas** en total.  
- Dado que la función Lambda se integra con S3, se almacenan igualmente **666.650 imágenes** en el bucket, cada una gatillando a su vez una función Lambda de procesamiento.

En total, con **666.650 imágenes** de **4 MB** cada una, se mantienen aproximadamente **2,67 TB** de objetos almacenados en S3 mensualmente.

### Escenario de mayor carga

Considerando las **desviaciones estándar** indicadas para la cantidad de posts e imágenes, el escenario más costoso se da cuando los usuarios generan **15 posts**, cada uno con **7 imágenes** de **6 MB**.  
En ese caso, habría **1.399.965 llamados** a funciones Lambda en el mes y un almacenamiento aproximado de **8,4 TB** en S3.

### Estimación de costos

Se evalúan los costos de implementar estos servicios según sus especificaciones y el uso estimado de la aplicación.  
El cálculo se realizó mediante **AWS Pricing Calculator**, que permite estimar gastos según el modelo de **pago por uso** o **reserva de recursos**.  
En este caso, los servicios se disponen mayormente en modo **pago por uso**.

#### Lambda de recepción
- **666.650 llamadas** al mes  
- **Duración promedio:** 600 ms  
- **Memoria asignada:** 128 MB  
- **Memoria efímera:** 512 MB  
- **Costo estimado:** 0,96 USD/mes  

#### Lambda de procesamiento
- **666.650 llamadas** al mes  
- **Duración promedio:** 2.000 ms  
- **Memoria asignada:** 128 MB  
- **Memoria efímera:** 512 MB  
- **Costo estimado:** 2,91 USD/mes  

Para ambas funciones Lambda se considera la duración promedio de la solicitud como un dato observado.  
Se implementan *timeouts* de 2 segundos para recepción y de 30 segundos para procesamiento, asumiendo que no se alcanzan estos límites durante la operación normal.

#### S3
El costo incluye:
- Solicitudes **PUT** para almacenar los objetos.  
- Solicitudes **GET** desde la función Lambda de procesamiento.  
- **Costo mensual de almacenamiento.**

Considerando **666.650 solicitudes GET y PUT** con objetos de **4 MB** en promedio, y **2,67 TB** almacenados el primer mes de operación, el costo asciende a **115,77 USD mensuales**.

Para simplificar el cálculo del almacenamiento, se tomó como base el costo de mantener las imágenes de un solo mes, aun cuando en la práctica el servicio crecerá con la demanda, ya que las imágenes antiguas no se eliminan para preservar la experiencia de usuario.

#### Rekognition (us-east-1)
El procesamiento de **666.650 imágenes** mediante la API de etiquetado genera un costo mensual de **666,65 USD**.

#### DynamoDB
Considerando la base de datos en modo **on-demand**:
- **Tamaño de almacenamiento:** 10 GB  
- **Tamaño promedio de elemento:** 1 KB  
- **Operaciones:** 666.650 lecturas y escrituras  
- **Costo estimado:** 2,96 USD/mes  

Se asume que todas las imágenes son procesadas y que se encuentran etiquetas con suficiente nivel de confianza para registrarlas.  
Varias etiquetas encontradas en una misma imagen se consideran como una sola escritura.

### Costo total

| Concepto | Costo mensual (USD) |
|-----------|--------------------:|
| Lambda (recepción) | 0,96 |
| Lambda (procesamiento) | 2,91 |
| S3 | 115,77 |
| Rekognition | 666,65 |
| DynamoDB | 2,96 |
| **Total mensual estimado** | **789,25** |
| **Costo inicial (infraestructura)** | **4,90** |
| **Costo anual estimado** | **9.475,90** |

### Estrategias de optimización de costos

Entre las decisiones que pueden adoptarse para reducir los costos se encuentran:

- **Evitar la subida de archivos duplicados** al bucket de S3.  
  Antes de subir una imagen, se puede calcular su hash (por ejemplo, **SHA-256**) y compararlo con los nombres de los objetos existentes, que corresponden también a hashes.  
  Si ya existe una coincidencia, se evita la carga del archivo, lo que ahorra espacio y costos.

- **Uso opcional de Rekognition.**  
  Dado su alto costo por imagen, se puede ofrecer al usuario la opción de activar o no el procesamiento automático de fotos en sus publicaciones, priorizando la experiencia sin aumentar gastos innecesarios.

**Costo mensual promedio anualizado:** ≈ **789,25 USD**

