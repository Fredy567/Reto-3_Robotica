# capytown_G[n] — RC-0 · Primer arranque y calibración

Robótica (08079) · Universidad ESAN · 2026-2 · Proyecto CapyTown · Robot Yahboom MicroROS-Pi5

| Dato | Valor |
|---|---|
| Sección | S-003 |
| Variantes | A1 Torre (calibración) · A2 Escalón (validación) |
| ROS_DOMAIN_ID | 20 |
| r (`wheel_radius`) final | **0.0305 m** |
| b_eff (`wheel_separation`) final | **0.215 m** |
| Tag | `v1.0-rc0` |

## Integrantes y roles

| Integrante | Rol | Responsabilidad en RC-0 |
|---|---|---|
| Fredy Alonso Toribio Sotelo| Integración | Bringup, ROS_DOMAIN_ID, rosbag2, bags, repositorio y tag |
| Walter Alexander Yucra Leyva | Control | Calibración de r y b_eff; `wheel_params.yaml` |
| Madeline Trisha Chavez Cervantes | Planificación | Secuencia de comandos de prueba (recta y giros) |
| Nilser Maier Villalobos Sernaque | Percepción | `plot_odom.py`: lectura de bags y gráficas |

## Resultados

| Prueba | Variante | Error | Criterio |
|---|---|---|---|
| Recta 1.0 m (promedio de 3 corridas, r = 0.0305 m) | A1 | 0.33 % | ≤ 2 % ✔ |
| Giro 360° antihorario (b_eff = 0.215 m) | A1 | 0.28 % | < 1 % ✔ |
| Recta 1 de validación | A2 | 0.67 % | — |
| Recta 2 de validación | A2 | 0.03 % | — |
| Giro 360° horario de validación | A2 | 5.56 % | — |

La calibración de r se sostuvo en la segunda variante; la de b_eff no, porque depende del patinaje lateral, del piso y del sentido de giro. El análisis completo está en `informe.pdf`.

## Contenido del repositorio

```
capytown_G[n]/
├── README.md
├── informe.pdf                  informe de 1 página
├── setup_log.txt                Parte A: nodos, topics, frecuencias, rosbag2
├── linea_recta.csv              Parte B: corridas de la recta
├── calibration_log.csv          Parte C: giros de calibración de b_eff
├── validacion.csv               Parte D: validación en la variante A2
├── config/
│   └── wheel_params.yaml        r y b_eff finales
├── bags/
│   ├── rc0_recta/               recta de calibración (A1)
│   ├── rc0_giro/                giro de calibración (A1)
│   ├── rc0_val_recta/           recta 1 de validación (A2)
│   ├── rc0_val_recta_2/         recta 2 de validación (A2)
│   └── rc0_val_giro/            giro de validación (A2)
├── plots/                       gráficas x-y y θ(t) de cada bag
├── capytown_esan/
│   ├── odom_corrector.py        /odom_raw → /odom con r y b_eff calibrados
│   ├── calibrate_beff.py        giro de 360° y cálculo de b_eff
│   ├── plot_odom.py             gráficas de los bags (Parte E)
│   ├── error_odom.py
│   ├── teleop_key.py
│   ├── run_square.py
│   └── fake_robot.py
├── launch/
│   ├── bringup.launch.py
│   └── sim.launch.py
├── resource/
├── package.xml
├── setup.py
└── setup.cfg
```

## Cómo funciona

La placa ESP32 publica `/odom_raw` con valores de fábrica (r0 = 0.0325 m, b0 = 0.24 m). El nodo `odom_corrector` corrige cada paso de esa odometría y publica `/odom`:

- distancia: `ds · r / r0`
- giro: `dθ · (r / r0) · (b0 / b_eff)`

r se calibró primero porque también afecta al giro.

## Uso

Dentro del contenedor `ros2_dev`, en cada terminal:

```bash
source /opt/ros/humble/setup.bash
source /root/yahboomcar_ws/install/setup.bash
export ROS_DOMAIN_ID=20
```

Compilar y lanzar (el bringup se lanza una sola vez):

```bash
cd /root/yahboomcar_ws
colcon build --packages-select capytown_esan --symlink-install
ros2 launch capytown_esan bringup.launch.py
```

Recta de 1 m (grabar en otra terminal con `ros2 bag record /odom /odom_raw /cmd_vel /tf -o <nombre>`):

```bash
ros2 topic pub --rate 10 --times 100 /cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.10}}"; ros2 topic pub --rate 10 --times 5 /cmd_vel geometry_msgs/msg/Twist "{}"
ros2 run capytown_esan error_odom <nombre>
```

Calibración de b_eff (el mismo valor en el nodo y en el script):

```bash
ros2 param set /odom_corrector wheel_separation 0.215
ros2 run capytown_esan calibrate_beff --ros-args -p b_eff:=0.215 -p target_angle_deg:=360.0 -p angular_vel:=0.5
```

Gráficas (lee el formato sqlite3 o mcap desde `metadata.yaml`):

```bash
python3 capytown_esan/plot_odom.py bags/rc0_recta bags/rc0_giro bags/rc0_val_recta bags/rc0_val_recta_2 bags/rc0_val_giro -o plots
```

## Problemas encontrados

- **Bringup duplicado:** dos publicadores en `/odom` (`ros2 topic info /odom`). Solución: lanzar el bringup una sola vez.
- **`ros2 topic pub --once` se perdía:** el freno y el reset no siempre llegaban. Solución: enviar 5 mensajes.
- **Variación entre corridas de ±3 %:** r se estimó con el promedio de varias corridas, no con una sola.
- **b_eff en el nodo y en el script:** el robot gira según el valor del nodo; el script solo calcula. Los dos deben coincidir.
