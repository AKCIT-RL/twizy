# Inicialização Automática no Veículo

O computador de bordo sobe a stack sozinho ao ligar (para ligar o veículo, veja
[Ligar o Veículo](startup.md)). **Não é preciso rodar `docker compose up`
manualmente** — os comandos manuais que aparecem nas outras páginas servem para depuração, não
para a operação normal.

Ao ligar o veículo, quatro serviços systemd entram em ação numa ordem definida, e regras udev
garantem nomes fixos para as interfaces CAN.

## Ordem de inicialização

```
network-online.target
        │
        ├─> dnsmasq-twizy.service          (DHCP da rede do LiDAR)
        ├─> docker-sock-permission.service (acesso ao socket do Docker)
        ├─> air-twizy-gige-tuning.service  (buffers e rotas das câmeras GigE)
        │
        └─> air-twizy-hardware.service ──> docker compose up -d
                                             ├─ discovery_server
                                             ├─ air_twizy_camera
                                             ├─ ouster_lidar
                                             └─ twizy  (baixo nível / CAN)
```

## Os serviços

| Serviço | Função |
|---|---|
| `air-twizy-hardware` | Sobe a stack Docker (`docker compose up -d`) a partir de `/home/air/tmp_simoes/air_twizy_hardware`. É `oneshot` com `RemainAfterExit=yes`, então aparece como *active (exited)* mesmo com tudo rodando — isso é normal, não é falha |
| `dnsmasq-twizy` | Servidor DHCP para o LiDAR em `enp11s0`, faixa `10.5.5.50–100` |
| `air-twizy-gige-tuning` | Ajusta buffers de rede do kernel e as rotas link-local das câmeras GigE; também configura `10.5.5.1/24` na interface do LiDAR |
| `docker-sock-permission` | Garante permissão de acesso ao socket do Docker |

Comandos úteis:

```bash
systemctl status air-twizy-hardware      # estado da stack
systemctl status dnsmasq-twizy air-twizy-gige-tuning
journalctl -u air-twizy-hardware -b      # log desde o último boot
```

## Nomes persistentes das interfaces CAN

As interfaces CAN **não** usam `can0`/`can1`/`can2`: a numeração do kernel varia conforme a ordem
de detecção, o que já causou confusão sobre qual barramento era qual. A regra
`/etc/udev/rules.d/90-twizy-can-names.rules` fixa os nomes pelo hardware:

| Nome fixo | Hardware | Uso |
|---|---|---|
| `can_twizy` | Adaptador PEAK **USB** | Barramento do veículo — é o que fala com o XCU StreetDrone |
| `can_aux1` | PEAK **PCIe** FD, canal 0 | Auxiliar |
| `can_aux2` | PEAK **PCIe** FD, canal 1 | Auxiliar |

Sempre use os nomes fixos em scripts e configurações. Se um nome não aparecer, verifique se o
adaptador está conectado na mesma porta USB de sempre — a regra do `can_twizy` casa também o
caminho físico da porta.

## Divergência entre o compose e a máquina

O serviço sobe o compose de `/home/air/tmp_simoes/air_twizy_hardware` — um diretório de trabalho
pessoal, não um clone canônico. É daí que vem a divergência a seguir.

O `docker-compose.yml` no `main` do repositório de hardware define o serviço do baixo nível como
`car`, com `container_name: air_car_container`. A máquina do veículo, porém, roda esse container
com o nome **`twizy`** — sinal de que ela está numa revisão diferente do compose.

Ao seguir qualquer procedimento, confirme os nomes reais antes:

```bash
docker ps --format '{{.Names}}\t{{.Status}}'
```

Os nomes observados na máquina são `twizy`, `ouster_lidar`, `air_twizy_camera` e
`discovery_server`. Alinhar a máquina ao `main` (ou o contrário) continua pendente.

Vale a mesma atenção com a variável `TWIZY_CAN_PORT`: o `.env` deve apontar para o nome udev
(`can_twizy`), não para `can0`/`can2`. O repositório traz `install_can_udev_rules.sh` para
instalar as regras.

## O que ainda não é automático

Dois pontos exigem intervenção manual depois do boot:

- **Driver do LiDAR.** Existe uma corrida entre a subida do driver Ouster e a configuração da
  interface de rede. Quando o LiDAR não publicar, um `docker restart ouster_lidar` resolve.
- **Relays de compressão.** Não fazem parte do compose; são injetados via `docker exec` e não
  sobrevivem a um reinício de container. Veja [Dashboard Web](../teleoperation/dashboard.md).
