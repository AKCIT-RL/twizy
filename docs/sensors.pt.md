# Sensores

Visão geral de tudo que está instalado no veículo hoje. Cada sensor tem uma página própria com a
configuração detalhada — aqui ficam só o essencial e os tópicos publicados.

## Resumo

| Sensor | Quantidade | Interface | Detalhes |
|---|---|---|---|
| Câmeras Lucid Vision Triton | 3 | GigE Vision (Ethernet) | [Câmera](camera/index.md) |
| LiDAR Ouster OS1-128 | 1 | Ethernet dedicada (`enp11s0`, `10.5.5.0/24`) | [LiDAR](lidar/index.md) |
| PCAN-GPS FD (GNSS + IMU) | 1 | CAN | [Hardware / Sensores](hardware/sensors.md) |
| Telemetria do veículo | — | CAN (`can_twizy`) | [Interface do Veículo](vehicle/interface.md) |

---

## Câmeras — Lucid Vision Triton

Três unidades TRI032S-CC (Sony IMX265, global shutter, 2048×1536, ~35 FPS), montadas num suporte
impresso em 3D no teto, cobrindo esquerda, centro e direita. O conjunto central usa lente
olho-de-peixe.

Publicam em `/camera/top_left/image_raw`, `/camera/top_front/image_raw` e
`/camera/top_right/image_raw`.

!!! note "Pela VPN, só comprimido"
    O dado bruto de uma câmera exige cerca de 800 Mbps, muito acima do que a VPN entrega. Um relay
    no veículo comprime para JPEG antes de transmitir — veja
    [Dashboard Web](teleoperation/dashboard.md).

## LiDAR — Ouster OS1-128

Sensor de 128 canais em suporte próprio no rack superior. Fica numa rede Ethernet dedicada, servida
por DHCP pelo `dnsmasq-twizy`, e é alcançável assim que a máquina inicia.

Publica a nuvem em `/ouster/points`, a IMU interna em `/ouster/imu` e quatro panorâmicas 360°
(`range`, `signal`, `nearir`, `reflec`), que são o que alimenta as abas do dashboard.

!!! warning "Corrida de inicialização"
    O driver pode subir antes de a interface de rede estar pronta. Quando não publicar,
    `docker restart ouster_lidar` resolve — veja [Inicialização Automática](vehicle/autostart.md).

## GNSS e IMU — PCAN-GPS FD

Módulo da PEAK System ligado ao barramento CAN, com receptor GNSS u-blox MAX-7W (GPS, GLONASS,
QZSS, SBAS), acelerômetro e magnetômetro Bosch BMC050 e giroscópio ST L3GD20.

A IMU está funcionando e publica em `/sd_imu_raw`; a posição sai em `/sd_current_GPS`.

!!! note "GPS ainda não validado ao ar livre"
    As coordenadas nunca foram conferidas com sinal de satélite em céu aberto — a IMU é a parte já
    confirmada. Consta no [Roadmap](roadmap.md).

## Telemetria do veículo

Não é um sensor separado, mas é dessa via que vêm os dados de estado do carro, lidos do barramento
CAN pelo SD-VehicleInterface: velocidade atual (`/current_velocity`, `/sd_current_twist`), ângulo
de esterço e estado da automação (`/sd_control`).

É também por aqui que se confirma a marcha (`PRND_Actual_Zs`) — informação decisiva, já que o
veículo não se move fora de Drive. Veja [Segurança](vehicle/safety.md).

---

## O que ainda não está instalado

O sidebar do dashboard mostra indicadores para **RADAR** e **Joystick** que não correspondem a
hardware presente no veículo — são resquícios do layout original da interface.
