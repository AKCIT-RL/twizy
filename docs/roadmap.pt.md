# Status & Roadmap

## O que está funcionando

### Hardware

| Componente | Status |
|------------|--------|
| Renault Twizy 80 (veículo) | Operacional — luz de serviço acesa, manutenção pendente |
| Bateria auxiliar (12V Varley) | Funcional — troca recomendada |
| Bateria de tração | Estado desconhecido — diagnóstico OBD2/CAN necessário |
| StreetDrone XCU | Operacional em `can_twizy` (PEAK USB) |
| PCAN-GPS FD | IMU funcionando via CAN; GPS não testado ao ar livre |
| Lucid Vision TRI032S-CC | Feed confirmado em ~30 FPS; lente C-mount instalada |
| Ouster LiDAR | Driver operacional; configuração de rede via dnsmasq documentada |
| Teltonika RUT950 | Validado com chip de teste; chip de produção necessário |

### Software

| Componente | Status |
|------------|--------|
| ROS2 Humble + stack Docker | Rodando no PC do veículo |
| FastDDS Discovery Server | Operacional em `twizy:11811` |
| NetBird VPN | Mesh P2P confirmado; hostname `twizy` resolve |
| Interface CAN (`can_twizy`) | Nomes persistentes via udev: `can_twizy` (PEAK USB) e `can_aux1`/`can_aux2` (PCIe FD) |
| Teleop Xbox (`direct_control`) | Pacotes válidos; **transporte DDS-sobre-VPN não entrega dados** (ver observação abaixo) |
| Dashboard web de teleoperação | Em uso em campo — 3 câmeras, LiDAR em abas, controle por teclado e painel de ajustes |
| Ponte SSH (sensores + controle) | Operacional — contorna a falha do DDS remoto; `~/twizy-ssh-bridge/start.sh` |
| Relays de compressão no veículo | Operacionais (câmeras, nuvem e panorâmicas do LiDAR); ~60–90 KB/s no total |
| Driver da câmera (arena_camera_ros2) | 3 câmeras a ~30 FPS na LAN do veículo; pela VPN só comprimido (~4,5 FPS) |
| Driver LiDAR (ouster_ros) | Nuvem e panorâmicas publicadas; exige `docker restart` após ligar (corrida de boot) |
| Simulação Gazebo | Carrega; teleop por teclado funciona |
| GitHub Pages (docs) | Publicado em [akcit-rl.github.io/twizy](https://akcit-rl.github.io/twizy/) |

---

## Problemas conhecidos

| Problema | Gravidade | Observações |
|----------|-----------|-------------|
| Compose do `main` diverge da máquina | Média | O `main` define `container_name: air_car_container`; o veículo roda o container como `twizy`. Confirme com `docker ps` antes de qualquer procedimento |
| `env.exemple` desatualizado no repositório | Média | Traz `TWIZY_CAN_PORT=can2` **tanto no `main` quanto em `feat/teleop`**, conflitando com as regras udev do próprio repositório. O valor correto é `can_twizy` |
| Dados ROS 2 não trafegam pela VPN | Alta | A descoberta funciona, o pareamento de endpoints não completa. Contornado pela ponte SSH; causa raiz não identificada |
| Veículo não se move sem marcha física | Alta | `PRND_Actual_Zs = 0` (Park/Neutral). O software não comanda marcha — o seletor precisa estar em Drive |
| Relays não sobrevivem a reinício de container | Média | Injetados via `docker exec`; falta integrá-los ao compose do veículo |
| Luz de serviço acesa | Média | Inspeção eletrônica/mecânica necessária antes de uso prolongado |
| Bateria auxiliar envelhecida | Média | +4 anos de uso; histórico de descarga profunda; suporte customizado necessário |
| GPS não testado ao ar livre | Baixa | IMU do PCAN-GPS FD funciona; coordenadas nunca validadas com sinal de satélite |
| `rqt` não funciona via VPN | Baixa | Usar script Python viewer como alternativa |
| Aterramento da tomada do lab | Alta | Tomada de 20A com aterramento correto obrigatória para carregamento |
| Chip SIM do RUT950 | Média | Chip de teste apenas — sem plano de conectividade para produção |
| Nomenclatura CAN antiga na documentação | Baixa | Antes das regras udev o barramento aparecia como `can2`; páginas que ainda citam `can0`/`can1`/`can2` precisam de revisão |

---

## Roadmap

### Curto prazo

- [ ] Alinhar o compose do veículo com o `main` (ou o contrário) — nomes de container divergentes
- [ ] Corrigir `TWIZY_CAN_PORT` para `can_twizy` no `main` e nas branches, e revisar as branches paradas
- [ ] Levar a documentação de inicialização automática também para o README do repositório de hardware
- [ ] Integrar os relays de compressão ao compose do veículo (hoje são temporários)
- [ ] Tornar o driver Ouster resiliente à corrida de inicialização (evitar `docker restart` manual)
- [ ] Investigar a causa raiz da falha do DDS remoto, ou consolidar a ponte SSH como solução definitiva

- [ ] Trocar bateria auxiliar e fabricar suporte de fixação
- [ ] Adquirir plano de dados para o RUT950
- [ ] Fazer diagnóstico OBD2 para avaliar estado da bateria de tração
- [ ] Testar GPS ao ar livre e validar coordenadas do PCAN-GPS FD
- [ ] Investigar códigos de falha da luz de serviço

### Médio prazo

- [ ] Documentar configuração multi-câmera (segunda unidade Lucid Vision)
- [ ] Validar calibração espacial LiDAR + câmera (extrínseca)
- [ ] Integrar dados de IMU na stack de navegação ROS2
- [ ] Testar latência da teleoperação via 4G (não apenas WiFi/LAN)
- [ ] Substituir tomada do laboratório por tomada de 20A aterrada

### Simulação

Portar a stack de hardware para um simulador é um entregável do projeto. O levantamento feito em
duas máquinas (uma com NVIDIA de 4 GB, outra com AMD de 12 GB) chegou a uma conclusão que muda o
planejamento.

**O LiDAR simulado do AWSIM exige NVIDIA — não é questão de VRAM.** A biblioteca
`libRobotecGPULidar.so` linka diretamente contra `libcuda`, `libnvidia-ml` e `libnvoptix`. Em GPU
AMD o carregamento falha e o LiDAR é desabilitado; em NVIDIA de 4 GB a VRAM estoura. O resto do
AWSIM (ambiente 3D, física, tráfego, câmera, GNSS, IMU) funciona nos dois casos.

Sem a nuvem de pontos o `e2e_simulator` não fecha o ciclo: a localização por NDT depende dela e a
inicialização só por GNSS não sustenta o modo autônomo.

**Consequência maior:** a percepção neural do Autoware (`lidar_centerpoint`, detectores de câmera)
exige CUDA/TensorRT e não tem backend ROCm. Mesmo com LiDAR simulado, a percepção não roda sem
NVIDIA — o único caminho sem CUDA é o `euclidean_cluster`, clustering geométrico clássico. Investir
em simulação de sensores não desbloqueia percepção; o gargalo será a GPU.

Situação de cada opção:

| Opção | Situação |
|---|---|
| **Autoware Planning Simulator** | **Em uso** — ciclo autônomo completo, sem sensores raytraced |
| `scenario_simulator_v2` (TIER IV) | **A avaliar** — ativo, nativo no Humble, simula LiDAR e detecções por raycast geométrico sem GPU, permite regressão por OpenSCENARIO |
| AWSIM | Parcial — serve como ambiente 3D e para validar o contrato do vehicle_interface, sem LiDAR |
| AWSIM-Labs | **Descartado** — arquivado em maio/2026 e usa a mesma biblioteca CUDA |
| ZLUDA (CUDA→AMD) | Descartado — suporte a OptiX removido |
| Isaac Sim / Omniverse | Descartado por ora — exige NVIDIA RTX com RT cores |
| CARLA | Não priorizado — 16 GB de download, sem relato confirmado na GPU disponível |

!!! note "O Planning Simulator não exercita o vehicle_interface"
    O `simple_planning_simulator` **ocupa o lugar** do vehicle_interface: consome
    `/control/command/control_cmd` e publica `/vehicle/status/*` diretamente. Ele mostra qual
    contrato o nó precisa implementar, mas não testa o nó em si. Para isso, o caminho é `vcan0`
    com o DBC do StreetDrone em loopback.

- [ ] Avaliar o `scenario_simulator_v2` como caminho para regressão automatizada
- [ ] Ancorar as TFs do URDF nos pontos de fixação de fábrica do chassi, os mesmos usados pelos
      suportes reais de câmera e LiDAR
- [ ] Escrever o `vehicle_interface` do Twizy: traduzir `autoware_control_msgs/Control` para o
      `DirectControl`/CAN já mapeado — **não depende de simulador de sensores**

### Longo prazo

- [ ] Integrar LiDAR + câmera em pipeline de percepção
- [ ] Implementar frenagem autônoma / desvio de obstáculos
- [x] Adicionar dashboard de monitoramento remoto — ver [Dashboard Web](teleoperation/dashboard.md)
- [ ] Teste de navegação autônoma ao ar livre
- [ ] Documentar versão do firmware do XCU StreetDrone e procedimento de atualização
