# CarlosArcila-1022443178-Prueba-Tecnica
# Outlet Rental Cars — Vehicle Search API

Web API para buscar vehículos disponibles y crear reservas, aplicando reglas de negocio del dominio de renta de vehículos y alineada con Clean Architecture + SOLID/DRY/KISS [file:1].

## Descripción de la solución
La API expone:
- Un **endpoint de búsqueda** (Query) para retornar vehículos disponibles dado: localidad de recogida, localidad de devolución, fecha/hora de recogida y fecha/hora de devolución [file:1].
- Un **endpoint de creación de reserva** (Command) que valida disponibilidad y genera un evento de dominio `VehicleReservedEvent` manejado internamente in-memory (sin bus externo) [file:1].

Persistencia:
- **MySQL** para información transaccional: vehículos y reservas [file:1].
- **MongoDB** para información de catálogo/configuración: mercados, tipos de vehículo, etc. [file:1].

## Reglas de negocio (según la prueba)
La búsqueda de vehículos disponibles aplica estas reglas [file:1]:
- Disponibilidad por localidad: el vehículo debe estar disponible en la localidad de recogida; la localidad de devolución puede ser distinta [file:1].
- Disponibilidad por mercado: solo retornar vehículos habilitados para el país al que pertenece una localidad (catálogo/config) [file:1].
- Rango de fechas: no retornar vehículos con una **reserva activa** que se cruce con el rango solicitado [file:1].
- Estado del vehículo: solo retornar vehículos con estado **Disponible** [file:1].

Overlap (cruce de reservas):
- Existe cruce si `existing.PickupAt < requested.DropoffAt` y `existing.DropoffAt > requested.PickupAt` (misma regla usada para bloquear disponibilidad).

## Decisiones técnicas tomadas (explicadas)

- Clean Architecture para mantener el dominio limpio.
  La prueba pedía explícitamente Clean Architecture y buenas prácticas, pero además es una decisión que se paga sola cuando el proyecto crece: el dominio y los casos de uso quedan aislados de frameworks y detalles de infraestructura [file:1]. En este tipo de retos es común “colar” EF Core en la capa Application (por ejemplo usando extensiones async de EF o devolviendo `IQueryable`), y eso termina generando acoplamientos, fricción con versiones y comportamientos inesperados; por eso la intención fue que Application hable el lenguaje del negocio (interfaces/puertos + casos de uso) y que Infrastructure resuelva el “cómo” (EF Core/MySQL, Mongo) [file:1]. Este enfoque ayuda a cumplir SOLID (DIP/SRP) y deja la solución lista para evolucionar sin reescribirlo todo.

- MySQL para transacciones y MongoDB para catálogo/configuración (modelo híbrido a propósito).
  Se eligió MySQL para la parte transaccional porque vehículos y reservas necesitan consultas eficientes, índices, y un modelo consistente (ej. validar solapes y estados), lo cual encaja natural en SQL [file:1][file:62]. MongoDB se reservó para catálogo/config (markets, tipos habilitados, etc.) porque esa información suele ser más “documental”, cambia distinto y requiere flexibilidad de esquema sin forzar migraciones frecuentes [file:1]. Separar “transacción” vs “config” también hace que la búsqueda sea más clara: primero determinas qué está habilitado por mercado, luego filtras disponibilidad real en reservas/vehículos.

- CQRS “ligero”: Query para buscar, Command para reservar (sin sobre-ingeniería).
  La prueba pedía un Query de búsqueda y un Command de reserva, así que se implementó esa separación sin meter librerías pesadas (KISS) [file:1]. La búsqueda no muta estado y la reserva sí; mantenerlos separados hace el código más fácil de razonar, probar y depurar cuando aparece un caso borde (por ejemplo fechas inválidas o solapes).

- Evento de dominio in-memory al reservar (separación real sin bus externo).
  Al crear una reserva se genera un evento de dominio (`VehicleReservedEvent`) como lo pide la prueba [file:1]. En vez de integrar mensajería (Kafka/Rabbit/Outbox), el manejo se hizo in-memory porque para el alcance del reto es suficiente, reduce complejidad y aun así deja el diseño preparado: cualquier reacción secundaria (auditoría, notificación, logging) puede engancharse al evento sin contaminar el caso de uso principal (SRP/KISS) [file:1].

- Reproducibilidad local con Docker + seed (pensado para el evaluador).
  La prueba exige que se ejecute local y permite usar data seed, así que la decisión fue documentar un flujo reproducible para levantar MySQL/Mongo y cargar datos base [file:1]. Esto evita el clásico “en mi máquina funciona” y facilita validar rápido el endpoint de búsqueda con casos que demuestren reglas del dominio (por ejemplo una reserva activa seed que bloquea un vehículo en un rango) [file:63][file:278].

- Tratamiento de errores como parte del contrato HTTP (no como “crash”).
  Varias validaciones (ej. `PickupAt < DropoffAt`) son errores esperados por input del cliente; por eso se prioriza devolver 400/404 con un mensaje claro en vez de propagar excepciones como 500 [file:1]. Esto mejora experiencia en Swagger/consumidores y hace que la API sea más predecible.

## Estructura del proyecto
- `src/OutletRentalCars.Domain`: entidades y enums (ej. `VehicleStatus`, `Reservation`, `ReservationStatus`).
- `src/OutletRentalCars.Application`: casos de uso, comandos/queries (ej. `CreateReservationCommand`), contratos (puertos) y eventos (dispatcher/handlers)
- `src/OutletRentalCars.Infrastructure`: EF Core MySQL + repos/servicios + integración con Mongo, y handler del evento.
- `src/OutletRentalCars.Api`: Controllers, Swagger y pipeline HTTP.

## Tutorial: correr la app y las BD (local)

### 1) Requisitos
- .NET 7 o superior (este repo compila en net10.0).
- Docker + Docker Compose.
- Puertos libres: MySQL `3306`, MongoDB `27017`.

### 2) Levantar MySQL y MongoDB (Docker)
Desde la raíz del repo:

```bash
docker compose up -d
docker ps
