# Ligar o Veículo

Sequência completa para tirar o Twizy do repouso e deixá-lo pronto para receber comandos por
software. **A ordem importa**: inverter os passos é a causa mais comum de luz vermelha no painel e
recusa de entrar em modo autônomo.

Antes de começar, leia [Segurança na Operação](safety.md).

## Painel StreetDrone

Além dos controles originais do Renault Twizy, o veículo tem um painel próprio:

| # | Controle | Função |
|---|---|---|
| 1 | Luz vermelha | Falha e estado do CAN |
| 2 | Luz verde | Modo manual |
| 3 | Luz âmbar | Modo autônomo |
| 4 | Botão de emergência | Corta a alimentação dos atuadores drive-by-wire |
| 5 | Chave seletora de modo (preta) | Arma o veículo para modo autônomo |
| 6 | Chave de energia do XCU (prateada) | Liga o XCU |
| 7 | Chave mestra dos atuadores | Isola a energia dos atuadores; sem ela o XCU ainda lê sensores, mas não atua |
| 8 | Chave de energia auxiliar 12 V | Alimenta o computador de bordo, sensores, hubs USB e a tela frontal |

!!! danger "As chaves 6 e 8 drenam a bateria"
    Deixe a **chave do XCU** e a **chave de energia auxiliar** desligadas sempre que a ignição do
    Twizy estiver desligada. Esquecê-las ligadas descarrega a bateria de 12 V — que já tem
    histórico de descarga profunda neste veículo.

## Sequência de partida

1. **Estacione em superfície plana e nivelada.** Deixe o **freio de mão desengatado**, a direção na
   posição central e **não pise** no freio nem no acelerador.

2. **Coloque o Twizy em modo "Go".** Segure a chave principal do Twizy por cerca de um segundo no
   fim do curso antes de soltar. O ícone **Go** deve aparecer no painel.

3. **Confirme as chaves 7 e 8.** Chave mestra dos atuadores encaixada e energia auxiliar 12 V
   ligada.

4. **Ligue o XCU** pela chave prateada do lado direito do painel (chave 6). As três luzes acendem
   por cerca de **15 segundos** de inicialização. Terminou quando resta apenas a **luz verde
   sólida** — modo manual.

Neste ponto o veículo está ligado e o computador de bordo sobe a stack sozinho — veja
[Inicialização Automática](autostart.md). Para teleoperar, siga para o
[Dashboard Web](../teleoperation/dashboard.md).

## Entrar em modo autônomo

Só faça isso com o piloto de segurança no banco e a área livre.

5. **Gire a chave seletora preta para a direita.** A luz âmbar começa a piscar devagar — é o setup
   do modo autônomo.

6. **Dentro de 5 segundos** o software precisa enviar o pedido CAN de torque e esterço autônomos.
   É o que o dashboard faz ao publicar em `/direct_control_cmd`. Se a luz voltar ao verde, o prazo
   expirou e o carro seguiu em manual — repita o passo 5.

7. **Sucesso** quando a luz âmbar passa a piscar **mais rápido**. O veículo está em modo autônomo.

!!! warning "A marcha continua sendo física"
    Modo autônomo armado não move o carro sozinho: o seletor precisa estar em **Drive**. O software
    não comanda marcha (`PRND_Actual_Zs`).

## Leitura das luzes

| Sinal | Significado |
|---|---|
| Todas as luzes acesas | Inicializando — aguarde |
| Âmbar piscando devagar | Preparando modo autônomo |
| Âmbar piscando rápido | Modo autônomo **ativo** |
| Âmbar muito rápido e depois verde fixo | Modo autônomo cancelado |
| Vermelho piscando com verde fixo | Sem conexão CAN, em modo manual |
| Verde fixo | Modo manual |
| Vermelho fixo | XCU em estado seguro (erro) |
| Vermelho fixo + verde fixo | XCU em estado seguro (erro) e modo manual |

## Sair do modo autônomo

Qualquer um destes desarma o modo autônomo:

- Girar a chave seletora para a **esquerda** (manual).
- Zerar os bits de requisição de torque e esterço pelo CAN.
- Apertar o **botão vermelho** do painel, que corta a energia dos controladores drive-by-wire.
- **O motorista assumir o controle** aplicando uma pequena força no volante, no freio ou no
  acelerador.
- Remover a chave mestra dos atuadores.
- Falha detectada em sensores ou **perda de comunicação CAN** — o desarme é automático.

## Quando algo não funciona

**Luz vermelha fixa e não entra em autônomo.** O XCU detectou erro e bloqueia o drive-by-wire. Na
prática quase sempre é erro de sequência na partida:

- O carro está ligado no carregador? Isso aciona erro no XCU.
- A chave principal do Twizy foi ligada **antes** da chave prateada do StreetDrone? Essa ordem é
  obrigatória.
- A chave seletora preta está para a **esquerda** (desligada)? O carro precisa ser iniciado em modo
  manual.

**Luz vermelha piscando.** Não há conexão CAN completa entre o XCU e o CAN do cliente. Verifique se
os quadros estão sendo transmitidos e se o cálculo de CRC está correto — o leitor PEAK CAN que
acompanha o veículo serve para inspecionar isso.

---

*Procedimento conforme o manual do usuário do StreetDrone Renault Twizy (documento do fabricante,
de circulação restrita à equipe).*
