## Reporte de bugs del sistema

### Como usuario viajero

[![Captura-de-pantalla-2026-04-08-a-la(s)-6-34-42-p-m.png](https://i.postimg.cc/SKz7sRt9/Captura-de-pantalla-2026-04-08-a-la(s)-6-34-42-p-m.png)](https://postimg.cc/Y4MmDr72)

> 1. En los cards de habitaciones deberia de aparecer la ciudad y el pais de cada una

[![Captura-de-pantalla-2026-04-08-a-la(s)-6-33-41-p-m.png](https://i.postimg.cc/Nfg44gLx/Captura-de-pantalla-2026-04-08-a-la(s)-6-33-41-p-m.png)](https://postimg.cc/WtHgpcGD)

> 2. En la busqueda de ciudad solo busca al colocar el nombre de la ciudad completo

[![Captura-de-pantalla-2026-04-08-a-la(s)-6-37-05-p-m.png](https://i.postimg.cc/7PX7qSYW/Captura-de-pantalla-2026-04-08-a-la(s)-6-37-05-p-m.png)](https://postimg.cc/qghgLhKs)

> 3. No se esta viendo el qr cuando ya se genera la confirmacion despues de hacer el pago

### Como usuario Administrador

- Modulo de reservas
[![Captura-de-pantalla-2026-04-08-a-la(s)-6-38-29-p-m.png](https://i.postimg.cc/Jnxy4Wm5/Captura-de-pantalla-2026-04-08-a-la(s)-6-38-29-p-m.png)](https://postimg.cc/5YH9nTXX)

> 1. En el panel de admin en el modulo de reservas no se esta visualizando el nombre del cliente ni correo, cuando el usuario viajera reserva la habitacion desde el lado publico

- Modulo de reservas
[![Captura-de-pantalla-2026-04-08-a-la(s)-6-40-05-p-m.png](https://i.postimg.cc/Hx6x1443/Captura-de-pantalla-2026-04-08-a-la(s)-6-40-05-p-m.png)](https://postimg.cc/7GTDgzRT)

> 2. En el crear reserva deberia dejarme seleciona la habitacion, no debe ingresar uuid de la habitacion

- Modulo de reservas
[![Captura-de-pantalla-2026-04-08-a-la(s)-6-41-18-p-m.png](https://i.postimg.cc/HxvphWdw/Captura-de-pantalla-2026-04-08-a-la(s)-6-41-18-p-m.png)](https://postimg.cc/47tkKgKy)

> 3. Las validaciones al crear una reserva deberian tener mejor visualizacion se pierden en el fondo

- Moudulo de clientes
[![Captura-de-pantalla-2026-04-08-a-la(s)-6-44-58-p-m.png](https://i.postimg.cc/tT8ggRwx/Captura-de-pantalla-2026-04-08-a-la(s)-6-44-58-p-m.png)](https://postimg.cc/JDc8681r)

> 1. Los clientes se crean bien desde el modulo directamente pero no se autoregistran al momento que se hace una reserva como usuario viajero ni cuando creas un reservacion desde el modulo de reservaciones

- Modulo de habitaciones
[![Captura-de-pantalla-2026-04-08-a-la(s)-6-49-41-p-m.png](https://i.postimg.cc/7hMyyxk4/Captura-de-pantalla-2026-04-08-a-la(s)-6-49-41-p-m.png)](https://postimg.cc/8f5YRgcX)

> 1. Deberia dejarme selecionar el hotel no escribir el uuid para crearlo
> 2. Deberia colcoar opciones para selecionar el Ala de la habitacion y no dejar escribir cualquier cosa

### Notas agregadas

- Las validaciones no se pueden visulizar bien en los modulos de clientes, reservas ni habitaciones, para ver un ejemlpo fijate en el numeral 3 del modulo de reservaciones
- Falta realizar la funcionad en donde el viajero llega con su qr y el administrador escanea el qr y puede confirmar la reservacion
- Cuando el viajero llega y le da el codigo el administrador puede ingresar el codigo y veriricar que si esta hech la reserva

