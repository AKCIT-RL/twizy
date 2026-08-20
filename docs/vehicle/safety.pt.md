# Segurança na Operação

Leia esta página antes de qualquer teste com o veículo em movimento. O controle por software é
real: com o veículo armado e em Drive, um comando de torque acelera o carro de verdade.

!!! danger "Regra inegociável"
    **Nunca opere o veículo sem um piloto de segurança no banco do motorista**, com a mão próxima
    à chave de modo e condições de assumir a direção imediatamente.

## Quem tem a palavra final: a chave de modo

A autonomia depende de um **handshake** entre o piloto de segurança e o veículo. Está no código do
SD-VehicleInterface e funciona assim:

| Sinal | Significado |
|---|---|
| `AutomationArmed_B` | verdadeiro quando **o piloto de segurança gira a chave de modo** para autônomo |
| `AutomationGranted_B` | verdadeiro quando o veículo **concede** o controle solicitado |

O software só pede controle autônomo (`RequestAutonomousControl`) enquanto `AutomationArmed_B` for
verdadeiro. Se o piloto desarmar a chave, a interface passa a enviar quadros CAN **zerados**
(`ResetControlCanData`) e os comandos de torque e esterço deixam de ter efeito.

Em outras palavras: **desarmar a chave de modo é o caminho confiável para retomar o controle**, e
não depende da rede, do computador de bordo nem do operador remoto.

O manual do fabricante confirma que o piloto também retoma o controle **aplicando uma pequena força
no volante, no freio ou no acelerador** — o XCU cancela o modo autônomo por conta própria. Outras
formas de desarme estão em [Ligar o Veículo](startup.md).

## Antes de operar

- [ ] Veículo ligado conforme [Ligar o Veículo](startup.md), com luz verde sólida no painel
- [ ] Piloto de segurança no veículo, ciente de que o teste vai começar
- [ ] Área livre, sem pessoas na trajetória
- [ ] Marcha física em **Drive** — o software não comanda marcha (veja [Operação](operation.md))
- [ ] Limites de aceleração conferidos no painel de ajustes do
      [dashboard](../teleoperation/dashboard.md); comece baixo e suba aos poucos
- [ ] Comunicação combinada entre operador e piloto (rádio, telefone ou viva-voz aberto)

## Durante a operação

- O **Espaço** no dashboard aplica freio de emergência por software (torque negativo, não inércia).
  É o primeiro recurso do operador remoto, mas **não substitui** o piloto de segurança.
- O operador deve anunciar cada comando antes de executá-lo.
- Ao menor sinal de comportamento inesperado, o piloto desarma a chave de modo. Discussão depois.

## Perda de comunicação

O manual do fabricante confirma que **a perda de comunicação CAN desativa o modo autônomo
automaticamente**, assim como qualquer falha detectada em sensores. O XCU implementa essa proteção
no próprio hardware, independente do software.

!!! warning "Isso cobre o CAN, não a VPN"
    A proteção do XCU vale para o barramento CAN dentro do veículo. Se o que cair for o **enlace
    remoto** (VPN ou ponte SSH), o computador de bordo continua ligado e o último comando publicado
    pode permanecer válido — não há watchdog documentado nesse trecho do caminho.

    Trate a queda de enlace como situação em que **o piloto de segurança assume imediatamente**.
    Medir esse comportamento com o veículo suspenso continua sendo um item pendente —
    veja [Status & Roadmap](../roadmap.md).

## Encerrar a operação

1. Zere os comandos: solte todas as teclas e confirme torque em zero
   (`ros2 topic echo /direct_control_cmd` deve mostrar `torque_setpoint: 0.0`).
2. O piloto de segurança **desarma a chave de modo**.
3. Marcha física em **Park**, freio de estacionamento aplicado.
4. Derrube a ponte de teleoperação no PC do operador: `~/twizy-ssh-bridge/stop.sh`.
5. Desligue a ignição do veículo.

!!! warning "Aguarde antes de deixar o veículo parado por muito tempo"
    Após desligar a ignição, o sistema pode continuar drenando a bateria de tração por cerca de
    **35 minutos**. Considere isso ao encerrar a operação, sobretudo se o veículo for ficar parado
    vários dias — a bateria auxiliar já tem histórico de descarga profunda.

## O que nunca fazer

- **Nunca envie comandos de aceleração por scripts ou `curl`** para "testar a cadeia" com o veículo
  armado. Valide sempre por leitura: `ros2 topic echo /direct_control_cmd`.
- Nunca opere sozinho, mesmo para um teste rápido.
- Nunca assuma que o veículo está desarmado sem confirmar a posição da chave de modo.
