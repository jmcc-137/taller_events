## Actividad

Haciendo uso de las siguientes tablas para la base de datos de `pizza` realice los siguientes ejercicios de `Events`  centrados en el uso de **ON COMPLETION PRESERVE** y **ON COMPLETION NOT PRESERVE** :

```sql

CREATE TABLE IF NOT EXISTS resumen_ventas (
fecha       DATE      PRIMARY KEY,
total_pedidos INT,
total_ingresos DECIMAL(12,2),
creado_en DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS ingrediente (
  id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  nombre VARCHAR(100) NOT NULL,
  stock INT NOT NULL DEFAULT 0
);

CREATE TABLE IF NOT EXISTS alerta_stock (
  id              INT AUTO_INCREMENT PRIMARY KEY,
  ingrediente_id  INT UNSIGNED NOT NULL,
  stock_actual    INT NOT NULL,
  fecha_alerta    DATETIME NOT NULL,
  creado_en DATETIME DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (ingrediente_id) REFERENCES ingrediente(id)
);


CREATE TABLE IF NOT EXISTS pedidos (
  id INT AUTO_INCREMENT PRIMARY KEY,
  total DECIMAL(10,2) NOT NULL,
  fecha_pedido DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

1. Resumen Diario Único : crear un evento que genere un resumen de ventas **una sola vez** al finalizar el día de ayer y luego se elimine automáticamente llamado `ev_resumen_diario_unico`.
```sql
DROP IF EXISTS ev_resumen_diario_unico

CREATE EVENT IF NOT EXISTS ev_resumen_diario_unico
ON SCHEDULE AT CURRENT_DATE + INTERVAL 1 DAY
ON COMPLETION NOT PRESERVE
DO
INSERT INTO resumen_ventas (fecha, total_pedidos,total_ingresos)
SELECT
    CURDATE() - INTERVAL 1 DAY,
    COUNT(*) AS total_pedidos,
    SUM(total) AS total_ingresos
FROM pedidos
WHERE DATE(fecha_pedido) = CURDATE() - INTERVAL 1 DAY;
```

2. Resumen Semanal Recurrente: cada lunes a las 01:00 AM, generar el total de pedidos e ingresos de la semana pasada, **manteniendo** el evento para que siga ejecutándose cada semana llamado `ev_resumen_semanal`.

```sql
DROP EVENT IF EXISTS ev_resumen_semanal;

CREATE EVENT IF NOT EXISTS ev_resumen_semanal
ON SCHEDULE EVERY 1 WEEK
STARTS (CURRENT_DATE + INTERVAL (8 - DAYOFWEEK(CURRENT_DATE)) DAY + INTERVAL 1 HOUR)
ON COMPLETION PRESERVE
DO
INSERT INTO resumen_ventas (fecha, total_pedidos, total_ingresos)
SELECT 
    CURDATE() - INTERVAL (DAYOFWEEK(CURDATE()) + 5) DAY,
    COUNT(*) AS total_pedidos,
    SUM(total) AS total_ingresos
FROM PEDIDOS
WHERE fecha_pedido BETWEEN CURDATE() - INTERVAL 7 DAY AND CURDATE();
```
3. Alerta de Stock Bajo Única: en un futuro arranque del sistema (requerimiento del sistema), generar una única pasada de alertas (`alerta_stock`) de ingredientes con stock < 5, y luego autodestruir el evento.
```sql
DROP EVENT IF EXISTS ev_alerta_stock_unico;

CREATE EVENT IF NOT EXISTS ev_alerta_stock_unico
ON SCHEDULE AT CURRENT_TIMESTAMP + INTERVAL 10 SECOND
ON COMPLETION NOT PRESERVE
DO
INSERT INTO alerta_stock (ingrediente_id, stock_actual, fecha_alerta)
SELECT id, stock, now()
FROM ingrediente
WHERE stock < 5;
```

4. Monitoreo Continuo de Stock: cada 30 minutos, revisar ingredientes con stock < 10 e insertar alertas en `alerta_stock`, **dejando** el evento activo para siempre llamado `ev_monitor_stock_bajo`.
```sql
DROP EVENT IF EXISTS ev_monitor_stock_bajo;


CREATE EVENT IF NOT EXISTS ev_alerta_stock_unico
ON SCHEDULE AT CURRENT_TIMESTAMP + INTERVAL 10 SECOND
ON COMPLETION NOT PRESERVE
DO
INSERT INTO alerta_stock (ingrediente_id, stock_actual, fecha_alerta)
SELECT id, stock, now()
FROM ingrediente
WHERE stock < 5;
```


5. Limpieza de Resúmenes Antiguos: una sola vez, eliminar de `resumen_ventas` los registros con fecha anterior a hace 365 días y luego borrar el evento llamado `ev_purgar_resumen_antiguo`.

```sql
DROP EVENT IF EXISTS ev_purgar_resumen_antiguo

CREATE EVENT IF NOT EXISTS ev_purgar_resumen_antiguo
ON SCHEDULE AT CURRENT_TIMESTAMP + INTERVAL 14 SECOND
ON COMPLETION NOT PRESERVE
DO
DELETE FROM resumen_ventas
WHERE fecha < CURDATE() - INTERVAL 3 DAY;

```