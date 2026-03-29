# Lab MikroTik CHR no Proxmox — Guia de Replicação
> Autor: lucasfelz | Atualizado: Março 2026
> Ambiente: Arch Linux (PC físico) + Proxmox 8.x (PC físico) + MikroTik CHR 7.20.8

---

## Índice

1. [Visão Geral do Ambiente](#1-visão-geral-do-ambiente)
2. [Download do MikroTik CHR](#2-download-do-mikrotik-chr)
3. [Configuração da Bridge no Proxmox](#3-configuração-da-bridge-no-proxmox)
4. [Criação da VM no Proxmox](#4-criação-da-vm-no-proxmox)
5. [Configuração de Rede no MikroTik](#5-configuração-de-rede-no-mikrotik)
6. [Verificação Final](#6-verificação-final)
7. [Topologia do Lab](#7-topologia-do-lab)
8. [Problemas Conhecidos e Soluções](#8-problemas-conhecidos-e-soluções)
9. [Próximos Passos — MTCNA](#9-próximos-passos--mtcna)

---

## 1. Visão Geral do Ambiente

### Infraestrutura física

| Componente | Detalhe |
|---|---|
| Arch Linux | PC físico separado · IP: `192.168.10.101` · interface: `enp8s0` |
| Proxmox 8.x | PC físico · IP: `192.168.10.2` |
| pfSense | Gateway da rede doméstica · IP LAN: `192.168.10.1` |
| MikroTik CHR | VM no Proxmox · ID 202 · RouterOS 7.20.8 |

> **Importante:** O Arch Linux é um PC físico **separado do Proxmox**, não uma VM. Ambos estão na mesma rede doméstica `192.168.10.0/24` atrás do pfSense.

### Objetivo

Criar um lab para estudo do MikroTik CHR voltado à certificação MTCNA. O CHR tem **uma única interface** (`ether3`) conectada à `vmbr2` (rede doméstica), com IP fixo `192.168.10.200`. O acesso do Arch via WinBox é direto pela rede doméstica — sem bridges isoladas, sem rotas estáticas adicionais.

### Por que CHR e não ISO?

O MikroTik não distribui ISO para VM — apenas imagens de disco prontas (`.vmdk` / `.img`). O CHR é o RouterOS completo rodando como VM, gratuito para testes com limitação de 1 Mbps de throughput (irrelevante para lab).

---

## 2. Download do MikroTik CHR

**URL:** https://mikrotik.com/download → seção **Cloud Hosted Router (CHR)** → canal **Stable**

Versão utilizada: `chr-7.20.8.vmdk`

```bash
# No Arch — baixar e extrair
cd ~/Downloads
unzip chr-7.20.8.vmdk.zip
# Resultado: chr-7.20.8.vmdk (~45 MB)

# Transferir para o Proxmox
scp ~/Downloads/chr-7.20.8.vmdk root@192.168.10.2:/tmp/
```

---

## 3. Configuração da Bridge no Proxmox

O CHR usa a `vmbr2` — bridge com uplink físico na rede doméstica. Verificar se já existe:

```bash
cat /etc/network/interfaces | grep vmbr2
```

Se não existir, adicionar ao final do arquivo `/etc/network/interfaces`:

```
auto vmbr2
iface vmbr2 inet manual
    bridge-ports <nome_da_interface_fisica>
    bridge-stp off
    bridge-fd 0
```

> Substituir `<nome_da_interface_fisica>` pela NIC física do Proxmox destinada à rede doméstica (ex: `enp3s0`). Verificar com `ip link show`.

Aplicar:

```bash
ifreload -a
```

---

## 4. Criação da VM no Proxmox

### Criar a VM (ID 202)

```bash
qm create 202 \
  --name mikrotik-chr \
  --memory 512 \
  --cores 2 \
  --cpu host \
  --net0 virtio,bridge=vmbr2 \
  --ostype l26 \
  --machine pc-i440fx-8.1
```

### Importar o disco

```bash
qm importdisk 202 /tmp/chr-7.20.8.vmdk local-lvm
```

### Configurar disco e boot

```bash
qm set 202 --virtio0 local-lvm:vm-202-disk-0
qm set 202 --boot order=virtio0
```

### Iniciar a VM

```bash
qm start 202
```

### Verificar configuração da VM

```bash
qm config 202
```

Output esperado:

```
boot: order=virtio0
cores: 2
cpu: host
machine: pc-i440fx-8.1
memory: 512
name: mikrotik-chr
net0: virtio=BC:24:11:xx:xx:xx,bridge=vmbr2
ostype: l26
virtio0: local-lvm:vm-202-disk-0,size=128M
```

### Acessar o console

```bash
qm terminal 202
# ou pelo painel web do Proxmox: VM 202 → Console
```

### Credenciais padrão

| Campo | Valor |
|---|---|
| Usuário | `admin` |
| Senha | *(vazio — pressionar Enter)* |

O RouterOS 7.x exige definição de senha no primeiro acesso:
- Mínimo 12 caracteres
- Pelo menos: 1 maiúscula, 1 minúscula, 1 número, 1 caractere especial

---

## 5. Configuração de Rede no MikroTik

Todos os comandos são executados no console do CHR (via `qm terminal` ou diretamente no console do Proxmox).

### Verificar interfaces disponíveis

```bash
/interface print
```

Resultado esperado:

```
# NAME    TYPE      ACTUAL-MTU  MAC-ADDRESS
0 ether3  ether     1500        BC:24:11:xx:xx:xx   ← vmbr2 (rede doméstica)
1 lo      loopback  65536       00:00:00:00:00:00
```

> **Por que ether3 e não ether1?**
> O RouterOS numera interfaces sequencialmente sem reutilizar IDs removidos. Se a VM teve interfaces recriadas durante configuração, a numeração continua de onde parou. Normal, não afeta o funcionamento.

### Configurar IP fixo

```bash
/ip address add address=192.168.10.200/24 interface=ether3
```

### Configurar rota padrão

```bash
/ip route add dst-address=0.0.0.0/0 gateway=192.168.10.1 distance=1
```

### Configurar DNS

```bash
/ip dns set servers=8.8.8.8,1.1.1.1
```

### Verificar configuração completa

```bash
/ip address print
/ip route print
/ip dns print
```

Output esperado do `/ip address print`:

```
# ADDRESS              NETWORK        INTERFACE
0 192.168.10.200/24    192.168.10.0   ether3
```

Output esperado do `/ip route print`:

```
#    DST-ADDRESS       GATEWAY         DISTANCE
0 As 0.0.0.0/0        192.168.10.1    1
  DAc 192.168.10.0/24  ether3          0
```

### Testar conectividade e DNS

```bash
/ping 8.8.8.8 count=4
/resolve www.mikrotik.com
/system package update check-for-updates
```

O `check-for-updates` deve retornar a versão disponível sem erro de DNS.

### Configurações iniciais recomendadas

```bash
# Identidade
/system identity set name=CHR-Lab

# Desabilitar serviços não utilizados
/ip service disable telnet,ftp,www

# Timeout de inatividade
# ATENÇÃO: RouterOS 7.x não aceita 0
# Mínimo: 00:01:00 · Máximo: 1d00:00:00
/user set admin inactivity-timeout=1h

# Backup inicial — fazer antes de qualquer lab
/system backup save name=lab-inicial
/export file=lab-inicial-export
```

---

## 6. Verificação Final

### Do Arch

```bash
ping 192.168.10.200
# Deve responder sem Redirect Host
```

### Do CHR

```bash
/ping 8.8.8.8 count=4
/resolve www.mikrotik.com
/system package update check-for-updates
/ip address print
/ip route print
```

### WinBox no Arch (Linux)

```bash
# Baixar em https://mikrotik.com/download → Winbox → Linux
chmod +x winbox64
./winbox64
# Conectar em: 192.168.10.200
# Usuário: admin
# Senha: <definida no primeiro acesso>
```

---

## 7. Topologia do Lab

```
[Arch Linux] — PC físico separado
192.168.10.101 / enp8s0
        |
        | rede doméstica 192.168.10.0/24
        |
[pfSense] — gateway
LAN: 192.168.10.1
        |
        |
[Proxmox] — PC físico
192.168.10.2
  vmbr2 → uplink físico (rede doméstica 192.168.10.0/24)
        |
        |── net0 → vmbr2
                        |
                [MikroTik CHR — VM 202]
                RouterOS 7.20.8
                ether3: 192.168.10.200/24
                gateway: 192.168.10.1 (pfSense)
                DNS: 8.8.8.8, 1.1.1.1

Acesso WinBox do Arch:
  192.168.10.200 — direto pela rede doméstica, sem rota estática
```

---

## 8. Problemas Conhecidos e Soluções

### DNS timeout ao verificar atualizações

**Sintoma:**
```
status: ERROR: could not resolve dns name (timeout)
```

**Causa:** DNS não configurado ou rota padrão ausente.

**Solução:**
```bash
# Confirmar DNS
/ip dns set servers=8.8.8.8,1.1.1.1

# Confirmar rota padrão — deve existir linha com "As" e 0.0.0.0/0
/ip route print
# Se ausente:
/ip route add dst-address=0.0.0.0/0 gateway=192.168.10.1

# Testar
/ping 8.8.8.8 count=4
/resolve www.mikrotik.com
```

---

### WinBox desconectando e reconectando a cada ~1 minuto

**Sintoma:** Log mostra `logged out` seguido imediatamente de `logged in` em loop.

**Causa:** `inactivity-timeout` padrão é `10m`. O WinBox perde keepalive e reconecta automaticamente.

**Solução:**
```bash
/user set admin inactivity-timeout=1h

# Verificar
/user print detail
```

---

### Parâmetros que não existem no RouterOS 7.x

| Comando tentado | Erro | Observação |
|---|---|---|
| `/ip service set ssh session-timeout=0` | `expected end of command` | Parâmetro não existe |
| `/ip service set ssh timeout=0` | `expected end of command` | Parâmetro não existe |
| `/user set admin inactivity-timeout=0` | `value out of range` | Mínimo aceito: `00:01:00` |

Para ver parâmetros disponíveis em qualquer comando:
```bash
/user set admin ?
/ip service set ssh ?
```

---

### Interfaces aparecem como ether3 em vez de ether1

**Causa:** Normal. O RouterOS numera sequencialmente sem reutilizar IDs removidos. Basta usar o nome real mostrado em `/interface print`.

---

## 9. Próximos Passos — MTCNA

Seguir os módulos da apostila Opentech na ordem. **Sempre fazer backup antes de cada lab:**

```bash
/system backup save name=antes-lab-X
```

### Módulo 1 — Introdução ao RouterOS

```bash
# Identidade
/system identity set name=CHR-Lab

# Usuários
/user print detail
/user add name=ops group=full password=Senha@Lab123
/user set admin inactivity-timeout=1h
/user set ops inactivity-timeout=1h

# Serviços
/ip service print
/ip service disable telnet,ftp,www

# Backup
/system backup save name=lab-m1
/export file=lab-m1-export
/file print
```

### Módulo 2 — DHCP

> Para praticar DHCP server é necessário ter clientes na mesma rede. No setup atual o CHR está na rede doméstica — use a `ether3` como servidor DHCP apenas em ambiente de teste controlado ou adicione uma segunda interface isolada futuramente.

```bash
# Pool
/ip pool add name=pool-lab ranges=192.168.10.210-192.168.10.250

# Rede
/ip dhcp-server network add \
  address=192.168.10.0/24 \
  gateway=192.168.10.200 \
  dns-server=8.8.8.8

# Servidor
/ip dhcp-server add \
  name=dhcp-lab \
  interface=ether3 \
  address-pool=pool-lab \
  disabled=no

# Verificar
/ip dhcp-server print
/ip dhcp-server lease print
```

### Módulo 3 — Bridging

```bash
# Criar bridge
/interface bridge add name=br-lab

# Adicionar porta
/interface bridge port add bridge=br-lab interface=ether3

# Mover IP para a bridge
/ip address remove [find interface=ether3]
/ip address add address=192.168.10.200/24 interface=br-lab

# Verificar
/interface bridge print
/interface bridge port print
/ip address print
```

### Módulo 4 — Roteamento

```bash
# Ver tabela de rotas
/ip route print

# Rota estática de exemplo
/ip route add dst-address=172.16.0.0/24 gateway=192.168.10.1

# Verificar qual rota é usada
/ip route check 8.8.8.8

# Diagnóstico
/ping 8.8.8.8 count=4
/tool traceroute 8.8.8.8
```

### Módulo 6 — Firewall

```bash
# INPUT — ordem importa, drop deve ser sempre a última regra
/ip firewall filter add chain=input connection-state=established,related action=accept comment=established
/ip firewall filter add chain=input connection-state=invalid action=drop comment=invalid
/ip firewall filter add chain=input protocol=icmp action=accept comment=icmp
/ip firewall filter add chain=input src-address=192.168.10.0/24 action=accept comment=lan
/ip firewall filter add chain=input action=drop comment=drop-all

# NAT masquerade
/ip firewall nat add chain=srcnat out-interface=ether3 action=masquerade

# Verificar
/ip firewall filter print
/ip firewall nat print
```

### Módulo 7 — QoS

```bash
# Limitar cliente específico
/queue simple add name=limite-lab target=192.168.10.101 max-limit=2M/5M

# Estatísticas
/queue simple print stats
```

### Módulo 8 — Túneis

```bash
# PPPoE client
/interface pppoe-client add \
  name=pppoe-wan \
  interface=ether3 \
  user=usuario \
  password=senha \
  disabled=no

# PPTP client
/interface pptp-client add \
  name=pptp-vpn \
  connect-to=<IP_SERVIDOR> \
  user=vpnuser \
  password=vpnpass \
  disabled=no

# Sessões ativas
/ppp active print
```

### Ferramentas úteis durante o estudo

```bash
/tool torch interface=ether3           # tráfego em tempo real
/tool traceroute 8.8.8.8              # caminho até destino
/log print follow                      # log ao vivo
/log print where topics~"firewall"     # logs de firewall
/ip firewall filter print              # regras com contadores
/ip firewall connection print          # conexões ativas
/system resource print                 # CPU, RAM, uptime
/interface print stats                 # estatísticas por interface
```

---

### Referências

- Apostila Opentech MTCNA — Fábio Bersan Rocha
- MikroTik Wiki: https://wiki.mikrotik.com
- WinBox / CHR download: https://mikrotik.com/download
- Forum MikroTik: https://forum.mikrotik.com

---

*Documentação revisada em 29/03/2026 | lucasfelz*
