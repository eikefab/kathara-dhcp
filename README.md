## Integrantes

* Eike Fabrício da Silva
* João Henrique Barbosa de Fernandes Alencar

## Etapas

### Etapa 1

#### Diagrama da topologia

![Topologia da rede](topologia.png)

Resumo:
- `R1` atua como gateway da rede, faz o roteamento entre as LANs e aplica NAT/firewall para acesso externo.
- `R2` atende a LAN da esquerda.
- `R3` atende a LAN da direita.
- O servidor `dhcp` fica na LAN central e atende as três sub-redes, com relay DHCP em `R2` e `R3`.

#### Tabela de endereçamento

| Segmento | Equipamento | Interface | Endereço | Máscara | Gateway | Função |
| --- | --- | --- | --- | --- | --- | --- |
| LAN central | `R1` | `eth0` | `192.168.1.1` | `/24` | - | Gateway da LAN central |
| LAN central | `dhcp` | `eth0` | `192.168.1.254` | `/24` | `192.168.1.1` | Servidor DHCP |
| LAN central | `pc4`, `pc5` | `eth0` | DHCP | `/24` | `192.168.1.1` | Clientes |
| Enlace R1-R2 | `R1` | `eth1` | `10.0.12.1` | `/30` | - | Roteamento entre `R1` e `R2` |
| Enlace R1-R2 | `R2` | `eth0` | `10.0.12.2` | `/30` | `10.0.12.1` | Roteamento entre `R2` e `R1` |
| LAN esquerda | `R2` | `eth1` | `192.168.2.1` | `/24` | - | Gateway da LAN esquerda |
| LAN esquerda | `pc0`, `pc1` | `eth0` | DHCP | `/24` | `192.168.2.1` | Clientes |
| Enlace R1-R3 | `R1` | `eth2` | `10.0.13.1` | `/30` | - | Roteamento entre `R1` e `R3` |
| Enlace R1-R3 | `R3` | `eth0` | `10.0.13.2` | `/30` | `10.0.13.1` | Roteamento entre `R3` e `R1` |
| LAN direita | `R3` | `eth1` | `192.168.3.1` | `/24` | - | Gateway da LAN direita |
| LAN direita | `pc2`, `pc3` | `eth0` | DHCP | `/24` | `192.168.3.1` | Clientes |

#### Faixas DHCP por LAN

| LAN | Faixa DHCP | Gateway entregue | DNS entregue |
| --- | --- | --- | --- |
| Central | `192.168.1.100-192.168.1.200` | `192.168.1.1` | `8.8.8.8` |
| Esquerda | `192.168.2.100-192.168.2.200` | `192.168.2.1` | `8.8.8.8` |
| Direita | `192.168.3.100-192.168.3.200` | `192.168.3.1` | `8.8.8.8` |

#### Plano de rotas estáticas

| Equipamento | Rota | Próximo salto | Objetivo |
| --- | --- | --- | --- |
| `R1` | `192.168.2.0/24` | `10.0.12.2` | Alcançar a LAN esquerda |
| `R1` | `192.168.3.0/24` | `10.0.13.2` | Alcançar a LAN direita |
| `R2` | `default` | `10.0.12.1` | Encaminhar tráfego para outras redes e internet |
| `R3` | `default` | `10.0.13.1` | Encaminhar tráfego para outras redes e internet |
| `dhcp` | `default` | `192.168.1.1` | Alcançar redes remotas e internet |

Observação:
- O acesso à internet sai por `R1`, que aplica NAT para as redes `192.168.1.0/24`, `192.168.2.0/24`, `192.168.3.0/24`, `10.0.12.0/30` e `10.0.13.0/30`.

#### Plano de testes

1. Subir ou reiniciar o laboratório:
   `kathara lrestart`
2. Validar leases DHCP em todas as LANs:
   `kathara exec pc0 -- ip -4 -o addr show dev eth0`
   `kathara exec pc2 -- ip -4 -o addr show dev eth0`
   `kathara exec pc4 -- ip -4 -o addr show dev eth0`
3. Forçar um novo DHCP discover e observar o processo:
   `kathara exec pc0 -- dhclient -r eth0`
   `kathara exec pc0 -- dhclient -v eth0`
4. Confirmar que os relays DHCP estão ativos:
   `kathara exec r2 -- pgrep -a dhcrelay`
   `kathara exec r3 -- pgrep -a dhcrelay`
5. Testar conectividade local com o gateway de cada LAN:
   `kathara exec pc0 -- ping -w 2 192.168.2.1`
   `kathara exec pc2 -- ping -w 2 192.168.3.1`
   `kathara exec pc4 -- ping -w 2 192.168.1.1`
6. Testar roteamento entre sub-redes:
   `kathara exec pc2 -- ping -w 2 192.168.1.100`
   `kathara exec pc0 -- traceroute 192.168.3.100`
7. Testar saída para a internet:
   `kathara exec pc0 -- ping -w 2 8.8.8.8`
   `kathara exec pc4 -- ping -w 2 8.8.8.8`

### Etapa 2

Este repositório.

### Etapa 3

#### Conectividade com a Internet

O acesso externo da topologia é centralizado em `R1`. No `lab.conf`, o roteador foi configurado com `r1[bridged]="true"`, o que conecta uma interface adicional do `R1` à rede externa do hospedeiro. Na prática, isso faz do `R1` o ponto de saída da topologia para a Internet.

As demais redes usam `R1` como caminho padrão para tráfego externo:
- `dhcp` usa gateway `192.168.1.1`;
- `R2` usa rota default via `10.0.12.1`;
- `R3` usa rota default via `10.0.13.1`;
- os clientes recebem como gateway o roteador da própria LAN, e esse roteador encaminha o tráfego até `R1`.

#### SNAT e DNAT

Em [r1.startup](r1.startup), o `R1` aplica `MASQUERADE` nas redes internas:
- `10.0.12.0/30`
- `10.0.13.0/30`
- `192.168.1.0/24`
- `192.168.2.0/24`
- `192.168.3.0/24`

Esse `MASQUERADE` implementa o papel de SNAT, reescrevendo o IP de origem dos pacotes internos para o IP da interface externa do `R1`. Assim, os hosts internos conseguem sair para a Internet mesmo usando endereçamento privado.

Para esta etapa, o funcionamento necessário foi o de saída para a Internet, portanto o mecanismo efetivamente usado foi SNAT. O DNAT fica como a técnica complementar para publicar serviços internos para fora, caso seja necessário redirecionar conexões de entrada para algum host da topologia.

#### Validação

O teste de conectividade com a Internet foi realizado a partir de um cliente interno, com sucesso:

```bash
kathara exec pc0 -- ping -w 2 8.8.8.8
```

Resultado observado:
- `2 packets transmitted`
- `2 received`
- `0% packet loss`

Isso confirma que o `R1` está funcionando como gateway padrão da topologia e que o NAT de saída está operacional.

### Etapa 4

#### Testes executados

O laboratório foi iniciado com:

```bash
kathara lstart
```

#### IPs obtidos via DHCP

As verificações com `ip addr show` confirmaram leases válidos nas três LANs:

| Nó | Comando | IP obtido |
| --- | --- | --- |
| `pc0` | `kathara exec pc0 -- ip -4 addr show dev eth0` | `192.168.2.x/24` |
| `pc2` | `kathara exec pc2 -- ip -4 addr show dev eth0` | `192.168.3.x/24` |
| `pc4` | `kathara exec pc4 -- ip -4 addr show dev eth0` | `192.168.1.x/24` |

Os endereços exatos podem variar conforme a ordem das concessões, mas os clientes permanecem nas sub-redes corretas. Isso confirma que o servidor `dhcp` e os relays em `R2` e `R3` continuam distribuindo endereços corretamente.

#### Conectividade entre sub-redes

Foi validada conectividade entre redes diferentes:

```bash
kathara exec pc0 -- ping -w 2 192.168.3.100
```

Resultado observado:
- `2 packets transmitted`
- `2 received`
- `0% packet loss`

Também foi verificado o caminho percorrido entre a LAN direita e a LAN central:

```bash
kathara exec pc2 -- traceroute 192.168.1.100
```

Saltos observados:
1. `192.168.3.1` (`R3`)
2. `10.0.13.1` (`R1`)
3. `192.168.1.100` (`pc4`)

Isso confirma o roteamento entre `R3`, `R1` e a LAN central.

#### Captura DHCP com tcpdump

No lugar do Wireshark, a análise foi feita com `tcpdump`, pois estou usando WSL, e não há suporte nativo para os containers do Kathara. Para a verificação do DHCP, basta renovar o lease com `dhclient` e conferir o endereço recebido.

Fluxo simples de verificação:

```bash
kathara exec pc0 -- ip addr show dev eth0
kathara exec pc0 -- dhclient -r eth0
kathara exec pc0 -- dhclient -v eth0
kathara exec pc0 -- ip addr show dev eth0
```

Para observar os pacotes do DORA, aí sim a captura pode ser feita com `tcpdump`.

Cliente:

```bash
kathara exec pc0 -- tcpdump -n -vvv -e -s0 -i eth0 port 67 or port 68
```

Servidor:

```bash
kathara exec dhcp -- tcpdump -n -vvv -e -s0 -i eth0 port 67 or port 68
```

Explicação das flags usadas no `tcpdump`:
- `-n`: não resolve nomes, exibindo IPs e portas numericamente;
- `-vvv`: aumenta o nível de detalhamento da saída;
- `-e`: mostra o cabeçalho Ethernet, incluindo os endereços MAC;
- `-s0`: captura o pacote completo, sem truncar o conteúdo;
- `-i eth0`: define a interface de captura;
- `port 67 or port 68`: filtra apenas o tráfego DHCP, nas portas UDP do servidor e do cliente.

#### Análise do DORA

Na captura do cliente `pc1`, foi observado o ciclo completo DORA:

1. `Discover`
   `0.0.0.0.68 > 255.255.255.255.67`
2. `Offer`
   `192.168.2.1.67 > 192.168.2.100.68`
3. `Request`
   `0.0.0.0.68 > 255.255.255.255.67`
4. `Ack`
   `192.168.2.1.67 > 192.168.2.100.68`

Campos observados na captura do cliente:
- `xid`: identificador da transação DHCP, gerado de forma dinâmica a cada negociação;
- `yiaddr` (`Your-IP`): endereço oferecido ao cliente, na captura `192.168.2.100`;
- `chaddr` (`Client-Ethernet-Address`): endereço MAC do cliente;
- `siaddr` (`Server-IP`): `192.168.1.254`;
- `giaddr` (`Gateway-IP`): `192.168.2.1`.

Na captura do servidor `dhcp`, a transação apareceu com o relay em `R2`:
- origem `10.0.12.2.67`
- destino `192.168.1.254.67`
- `giaddr`: `192.168.2.1`
- `yiaddr`: endereço da LAN esquerda entregue ao cliente;
- `chaddr`: MAC do cliente que originou a requisição;
- `siaddr`: `192.168.1.254`.

Interpretação:
- o cliente transmite `Discover` e `Request` em broadcast na LAN esquerda;
- o relay em `R2` encaminha a requisição ao servidor central `dhcp`;
- o servidor responde com `Offer` e `Ack`;
- o relay devolve a resposta para a LAN do cliente.

#### Camadas envolvidas

As capturas mostraram as seguintes camadas:
- Ethernet
- IPv4
- UDP
- BOOTP/DHCP

Portas observadas:
- cliente DHCP: UDP `68`
- servidor DHCP: UDP `67`

Conclusão:
- os clientes obtêm IP por DHCP corretamente;
- há conectividade entre sub-redes;
- o fluxo DORA foi confirmado com `tcpdump` tanto do lado do cliente quanto do lado do servidor.
