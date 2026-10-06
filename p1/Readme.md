
# Aspiradora de gama baja con Autómata de Estados Finito reactivo
Esta primera práctica implementa un autómata de estados finitos (FSM) que controla el comportamiento de una aspiradora de gama baja que usa sus sensores para evitar obstaculos.

El objetivo de esta practica es que la aspiradora aspire el mayor área de la casa posible, lo recorre en espiral, retrocede cuando ve obstaculos y avanza a una nueva zona en la que no hay obstaculos cerca

La demo de la práctica la podeis encontrar en el siguiente enlace(el video esta en x2.5 ya que su duracion era muy larga): [Ver video de la demo](https://drive.google.com/file/d/1RwMHjFkjsX_8wFNqYEUQfJMU7Nit3guZ/view?usp=sharing)

## Estructura general del sistema

El programa sigue una máquina de estados finitos (FSM) con los siguientes estados:
  - TURNING: Estado inicial del robot y en el cual realiza las espirales
  - BACKWARD: Estado al que pasa tras encontrar un objeto con el laser, consiste en poner una velocidad negativa(para retroceder) hasta que el obstaculo este lo suficientemente lejos
  - AVOIDING_OBSTACLE: Una vez ha retrocedido lo suficiente, gira hacia el lado donde la media de obstáculos está más lejos, es decir, girará a la derecha si los obstáculos están más lejos a ese lado y viceversa
  - FORWARD: Este es el último estado al que pasa y lo hace cuando ha girado lo suficiente. Consiste en ir hacia delante 5 segundos; durante este tiempo no debe encontrar ningún obstáculo y, si lo hace, volverá al estado BACKWARD. De no haber objeto, comenzará a hacer espirales nuevamente

## Lógica del autómata
El corazón del sistema es el bucle principal que actualiza las velocidades según el estado actual del robot:
```python

while True:
    if state == FORWARD:
        ...
    elif state == BACKWARD:
        ...
    elif state == TURNING:
        ...
    elif state == AVOIDING_OBSTACLE:
        ...
    
    HAL.setV(vel_v)
    HAL.setW(vel_w)
    Frequency.tick()

```
Cada ciclo:
  - Lee los datos del láser.
  - Evalúa colisiones.
  - Actualiza el estado del robot.
  - Publica las velocidades lineal y angular (HAL.setV, HAL.setW).

## Detección de obstáculos:

### parse_laser_data
El robot usa un sensor láser de 180º para detectar obstáculos en su entorno. La función parse_laser_data convierte los datos en coordenadas polares y cartesianas:

```python

def parse_laser_data(laser_data):
    laser_polar = []  # Laser data in polar coordinates (dist, angle)
    laser_xy = []  # Laser data in cartesian coordinates (x, y)
    for i in range(180):
        # i contains the index of the laser ray, which starts at the robot's right
        # The laser has a resolution of 1 ray / degree
        #
        #                (i=90)
        #                 ^
        #                 |x
        #             y   |
        # (i=180)    <----R      (i=0)

        # Extract the distance at index i
        dist = laser_data.values[i]
        # The final angle is centered (zeroed) at the front of the robot.
        angle = math.radians(i - 90)
        laser_polar += [(dist, angle)]
        # Compute x, y coordinates from distance and angle
        x = dist * math.cos(angle)
        y = dist * math.sin(angle)
        laser_xy += [(x, y)]
    return laser_polar, laser_xy

````

### collides()

Con la funcion anterior, collides() determina si hay un obstáculo demasiado cerca:
````python

def collides():
    laser_data = HAL.getLaserData()

    if len(laser_data.values) > 0:
        laser_polar, _ = parse_laser_data(laser_data)

        # Divide los datos del laser en sus respectivos segmentos, derecha, izquierda y al frente
        left = laser_polar[120:180]
        front = laser_polar[60:120]
        right = laser_polar[0:60]
        
        # Calcula la media de la distancia en cada segmento
        avg_left = np.mean([dist for dist, angle in left])
        #avg_front = np.mean([dist for dist, angle in front])
        avg_right = np.mean([dist for dist, angle in right])

        min_front = min(dist for dist, angle in front)

        if min_front < DIST_OBJ or avg_left < DIST_OBJ or avg_right < DIST_OBJ:
            return True
        else:
            return False
        
    else:
        return False

````
Pese a que con la informacion que conoce podria devolver el lado mas vacio o el mas ocupado, decidi separar esa parte en otra función llamada analyze_laser_data().

## AVOIDING_OBSTACLE
Este comportamiento me generó grandes conflictos porque no sabía si usar ángulos aleatorios o tiempos aleatorios para que el algoritmo tuviera aleatoriedad. Al final me decidí por usar tiempos aleatorios ya que simplificaban muchísimo el algoritmo. Tras esto se me ocurrió que, en vez de ponerlo a girar a un lado aleatorio en un tiempo aleatorio, analizara qué lado estuviera más vacío para girar hacia allí, de forma que la probabilidad de chocarse contra otro obstáculo fuera menor.

Al principio, mi código evaluaba hacia qué lado girar continuamente mientras el robot ya estaba girando. Esto provocaba que los datos del láser cambiaran constantemente, haciendo que el algoritmo cambiara de decisión en cada iteración y el robot se quedara atascado "temblando" sobre sí mismo.

La solución fue tomar la decisión de la dirección **una sola vez** justo antes de empezar a girar, guardar esa velocidad en una variable (`turn_speed`), y aplicarla de manera constante durante todo el tiempo aleatorio:

```python
# 1. Al salir del estado BACKWARD, decidimos la dirección UNA VEZ y la guardamos:
if direction == "RIGHT":
    turn_speed = -1
else:
    turn_speed = 1  # Si es LEFT o FORWARD, giramos a la izquierda

````


