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

| Equipamento | Rota | Proximo salto | Objetivo |
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
   `kathara exec pc0 -- sh -lc 'dhclient -r eth0; dhclient -v eth0'`
4. Confirmar que os relays DHCP estão ativos:
   `kathara exec r2 -- sh -lc 'ps -ef | grep [d]hcrelay'`
   `kathara exec r3 -- sh -lc 'ps -ef | grep [d]hcrelay'`
5. Testar conectividade local com o gateway de cada LAN:
   `kathara exec pc0 -- sh -lc 'ping -c 2 192.168.2.1'`
   `kathara exec pc2 -- sh -lc 'ping -c 2 192.168.3.1'`
   `kathara exec pc4 -- sh -lc 'ping -c 2 192.168.1.1'`
6. Testar roteamento entre sub-redes:
   `kathara exec pc2 -- sh -lc 'ping -c 2 192.168.1.100'`
   `kathara exec pc0 -- sh -lc 'traceroute 192.168.3.100 || tracepath 192.168.3.100'`
7. Testar saída para a internet:
   `kathara exec pc0 -- sh -lc 'ping -c 2 8.8.8.8'`
   `kathara exec pc4 -- sh -lc 'ping -c 2 8.8.8.8'`

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

Em [r1.startup](/r1.startup), o `R1` aplica `MASQUERADE` nas redes internas:
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
kathara exec pc0 -- sh -lc 'ping -c 2 8.8.8.8'
```

Resultado observado:
- `2 packets transmitted`
- `2 received`
- `0% packet loss`

Isso confirma que o `R1` está funcionando como gateway padrão da topologia e que o NAT de saída está operacional.

### Etapa 4
