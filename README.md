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

## Decisiones técnicas tomadas
- **Clean Architecture**: Domain + Application no dependen de infraestructura; Infrastructure implementa acceso a datos (MySQL/Mongo) y la API orquesta HTTP/DI [file:1].
- **DI correcto**: Controllers delgados, casos de uso y servicios resueltos por interfaces (DIP).
- **SOLID/DRY/KISS**: responsabilidades separadas (controllers vs casos de uso vs persistencia), validaciones consistentes y evento in-memory para no introducir complejidad de mensajería (KISS) [file:1].
- **CQRS ligero**: Query para búsqueda y Command para crear reserva, sin frameworks pesados.

## Estructura del proyecto
- `src/OutletRentalCars.Domain`: entidades y enums (ej. `VehicleStatus`, `Reservation`, `ReservationStatus`) [file:222].
- `src/OutletRentalCars.Application`: casos de uso, comandos/queries (ej. `CreateReservationCommand`), contratos (puertos) y eventos (dispatcher/handlers) [file:221][file:218][file:219].
- `src/OutletRentalCars.Infrastructure`: EF Core MySQL + repos/servicios + integración con Mongo, y handler del evento.
- `src/OutletRentalCars.Api`: Controllers, Swagger y pipeline HTTP.

## Tutorial: correr la app y las BD (local)

### 1) Requisitos
- .NET 7 o superior (este repo compila en net10.0).
- Docker + Docker Compose.
- Puertos libres: MySQL `3306`, MongoDB `27017` [file:278].

### 2) Levantar MySQL y MongoDB (Docker)
Desde la raíz del repo:

```bash
docker compose up -d
docker ps
