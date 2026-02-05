# CarlosArcila-1022443178-Prueba-T-cnica
API para gestión de vehículos, mercados/locations y reservas (con validación de disponibilidad por rango de fechas), implementada con Clean Architecture y buenas prácticas (SOLID, DRY, KISS) [web:310].

## Tech stack
- .NET (ASP.NET Core Web API) [web:310]
- Entity Framework Core + MySQL (Pomelo) en **Infrastructure**
- Swagger/OpenAPI (Swashbuckle)

## Arquitectura
La solución está organizada en 4 capas:

- `OutletRentalCars.Domain`: Entidades, value objects, enums y eventos de dominio (sin dependencias externas).
- `OutletRentalCars.Application`: Casos de uso, puertos (interfaces), validaciones y contratos; **no depende de EF Core** (DIP/Clean Architecture).  
- `OutletRentalCars.Infrastructure`: Persistencia EF Core, implementaciones de repositorios, DbContext, mapeos y configuración de data access.
- `OutletRentalCars.Api`: Controllers, DI, pipeline HTTP, Swagger y manejo global de errores con `ProblemDetails` [web:310][web:332].

## Reglas de negocio principales
- Un vehículo solo se puede reservar si está `Available`.
- El rango de fechas debe ser válido: `PickupAt < DropoffAt`.
- No se permite una reserva activa que se solape con otra para el mismo vehículo: hay overlap si `existing.PickupAt < new.DropoffAt` y `existing.DropoffAt > new.PickupAt`.
- La búsqueda de vehículos disponibles filtra por:
  - Location existente y su country.
  - Market habilitado para el país.
  - VehicleTypeCode permitido por el market.
  - Ausencia de reservas activas solapadas.

## Manejo de errores (ProblemDetails)
La API retorna errores en formato `application/problem+json` usando el estándar de Problem Details (RFC 7807) [web:332].

- `400 Bad Request`: errores de validación (p.ej. rango de fechas inválido).
- `404 Not Found`: recursos inexistentes (p.ej. vehículo no encontrado).
- `500 Internal Server Error`: errores no controlados (sin exponer stack trace en ambientes no-dev).

## Endpoints (Swagger)
Al ejecutar el proyecto, Swagger queda disponible en:

- `http://localhost:<puerto>/swagger`

Endpoints típicos:
- `GET /api/vehicles/search` (búsqueda de vehículos disponibles por location + rango)
- `POST /api/reservations` (creación de reserva)

> Nota: Los nombres exactos pueden variar según el controller; Swagger es la fuente de verdad.

## Configuración
### Variables / appsettings
Configura el connection string de MySQL en `appsettings.Development.json` (o variables de entorno), por ejemplo:

```json
{
  "ConnectionStrings": {
    "Default": "Server=localhost;Port=3306;Database=outlet_rental_cars;User=root;Password=your_password;"
  }
}
