# Lab MikroTik CHR no Proxmox — Guia de Replicação
> Autor: lucasfelz | Atualizado: Março 2026
> Ambiente: Arch Linux (PC físico) + Proxmox 8.x (PC físico) + MikroTik CHR 7.20.8

---

## Índice

1. [Visão Geral do Ambiente](#1-visão-geral-do-ambiente)
2. [Download do MikroTik CHR](#2-download-do-mikrotik-chr)
3. [Configuração das Bridges no Proxmox](#3-configuração-das-bridges-no-proxmox)
4. [Criação da VM no Proxmox](#4-criação-da-vm-no-proxmox)
5. [Configuração de Rede no MikroTik](#5-configuração-de-rede-no-mikrotik)
6. [Verificação Final](#6-verificação-final)
7. [Topologia do Lab](#7-topologia-do-lab)
8. [Problemas Conhecidos e Soluções](#8-problemas-conhecidos-e-soluções)
9. [Próximos Passos — MTCNA](#9-próximos-passos--mtcna)
10. [Histórico de Alterações](#10-histórico-de-alterações)

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

Criar um lab para estudo do MikroTik CHR voltado à certificação MTCNA e uso no dia a dia de NOC. O CHR tem **duas interfaces**:

- `ether1` → `vmbr2` — WAN · rede doméstica · IP fixo `192.168.10.200` · acesso WinBox e internet
- `ether2` → `vmbr99` — LAN · rede isolada de lab `10.99.0.0/24` · sem uplink físico

Esse setup replica o padrão ensinado nos cursos MikroTik — internet na ether1, clientes na ether2.

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

## 3. Configuração das Bridges no Proxmox

O CHR usa duas bridges:

| Bridge | Tipo | Função |
|--------|------|--------|
| `vmbr2` | Com uplink físico | WAN — rede doméstica `192.168.10.0/24` |
| `vmbr99` | Sem uplink físico | LAN — rede isolada de lab `10.99.0.0/24` |

### Verificar se as bridges já existem

```bash
cat /etc/network/interfaces
```

### Criar vmbr2 (se não existir)

```bash
nano /etc/network/interfaces
```

```
auto vmbr2
iface vmbr2 inet manual
    bridge-ports <nome_da_interface_fisica>
    bridge-stp off
    bridge-fd 0
```

> Substituir `<nome_da_interface_fisica>` pela NIC física do Proxmox na rede doméstica. Verificar com `ip link show`.

### Criar vmbr99 (rede isolada de lab)

```
auto vmbr99
iface vmbr99 inet static
    address 10.99.0.254/24
    bridge-ports none
    bridge-stp off
    bridge-fd 0
    comment "Lab MikroTik isolado"
```

Aplicar:

```bash
ifreload -a
```

Verificar:

```bash
ip addr show vmbr2
ip addr show vmbr99
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
  --net1 virtio,bridge=vmbr99 \
  --ostype l26 \
  --machine pc-i440fx-8.1
```

> **Ordem das bridges:**
> - `--net0` → `vmbr2` (WAN) → aparece como `ether1` no CHR após renomear
> - `--net1` → `vmbr99` (LAN) → aparece como `ether2` no CHR após renomear

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
net1: virtio=BC:24:11:xx:xx:xx,bridge=vmbr99
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

Todos os comandos são executados no console do CHR (via `qm terminal` ou painel web do Proxmox).

### Verificar interfaces disponíveis

```bash
/interface print
```

As interfaces podem aparecer com numeração diferente de `ether1/ether2` dependendo do histórico da VM. Renomear para o padrão dos cursos:

```bash
/interface set 0 name=ether1
/interface set 1 name=ether2
```

Verificar:

```bash
/interface print
# ether1 → vmbr2 (WAN)
# ether2 → vmbr99 (LAN)
```

### Configurar WAN (ether1)

```bash
# IP fixo na rede doméstica
/ip address add address=192.168.10.200/24 interface=ether1

# Rota padrão via pfSense
/ip route add dst-address=0.0.0.0/0 gateway=192.168.10.1 distance=1

# DNS
/ip dns set servers=8.8.8.8,1.1.1.1
```

### Configurar LAN (ether2)

```bash
# IP fixo na rede de lab
/ip address add address=10.99.0.1/24 interface=ether2
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
0 192.168.10.200/24    192.168.10.0   ether1
1 10.99.0.1/24         10.99.0.0      ether2
```

Output esperado do `/ip route print`:

```
#    DST-ADDRESS       GATEWAY         DISTANCE
0 As 0.0.0.0/0        192.168.10.1    1
  DAc 192.168.10.0/24  ether1          0
  DAc 10.99.0.0/24     ether2          0
```

### Testar conectividade e DNS

```bash
/ping 8.8.8.8 count=4
/resolve www.mikrotik.com
/system package update check-for-updates
```

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
# Ping na WAN do CHR
ping 192.168.10.200
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
  vmbr2  → uplink físico (rede doméstica 192.168.10.0/24)
  vmbr99 → bridge isolada, sem uplink (rede de lab 10.99.0.0/24)
        |
        |── net0 → vmbr2  (WAN) ──────────────────┐
        |── net1 → vmbr99 (LAN) ──────────────────┤
                                                   |
                                       [MikroTik CHR — VM 202]
                                       RouterOS 7.20.8
                                       ether1: 192.168.10.200/24 (WAN)
                                       ether2: 10.99.0.1/24 (LAN)
                                       gateway: 192.168.10.1 (pfSense)
                                       DNS: 8.8.8.8, 1.1.1.1

Acesso WinBox do Arch:
  192.168.10.200 — direto pela rede doméstica
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
/ip dns set servers=8.8.8.8,1.1.1.1

/ip route print
# Se não houver 0.0.0.0/0:
/ip route add dst-address=0.0.0.0/0 gateway=192.168.10.1

/ping 8.8.8.8 count=4
/resolve www.mikrotik.com
```

---

### WinBox desconectando e reconectando a cada ~1 minuto

**Sintoma:** Log mostra `logged out` seguido imediatamente de `logged in` em loop.

**Causa:** `inactivity-timeout` padrão é `10m`.

**Solução:**
```bash
/user set admin inactivity-timeout=1h
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

### Interfaces aparecem com numeração inesperada (ex: ether3, ether4)

**Causa:** O RouterOS numera interfaces sequencialmente sem reutilizar IDs de interfaces removidas. Renomear conforme seção 5.

---

## 9. Próximos Passos — MTCNA

Seguir o curso na ordem dos módulos. **Sempre fazer backup antes de cada lab:**

```bash
/system backup save name=antes-lab-X
```

### Módulo 1 — Introdução ao RouterOS

```bash
/system identity set name=CHR-Lab
/user print detail
/user add name=ops group=full password=Senha@Lab123
/user set admin inactivity-timeout=1h
/user set ops inactivity-timeout=1h
/ip service print
/ip service disable telnet,ftp,www
/system backup save name=lab-m1
/export file=lab-m1-export
```

### Módulo 2 — DHCP

```bash
# Pool para a LAN
/ip pool add name=pool-lan ranges=10.99.0.100-10.99.0.200

# Rede DHCP
/ip dhcp-server network add \
  address=10.99.0.0/24 \
  gateway=10.99.0.1 \
  dns-server=8.8.8.8

# Servidor na LAN
/ip dhcp-server add \
  name=dhcp-lan \
  interface=ether2 \
  address-pool=pool-lan \
  disabled=no

/ip dhcp-server print
/ip dhcp-server lease print
```

### Módulo 3 — Bridging

```bash
/interface bridge add name=br-lan
/interface bridge port add bridge=br-lan interface=ether2
/ip address remove [find interface=ether2]
/ip address add address=10.99.0.1/24 interface=br-lan
/interface bridge print
/interface bridge port print
```

### Módulo 4 — Roteamento

```bash
/ip route print
/ip route add dst-address=172.16.0.0/24 gateway=10.99.0.254
/ip route check 8.8.8.8
/ping 8.8.8.8 count=4
/tool traceroute 8.8.8.8
```

### Módulo 6 — Firewall

```bash
# INPUT — drop deve ser sempre a última regra
/ip firewall filter add chain=input connection-state=established,related action=accept comment=established
/ip firewall filter add chain=input connection-state=invalid action=drop comment=invalid
/ip firewall filter add chain=input protocol=icmp action=accept comment=icmp
/ip firewall filter add chain=input src-address=192.168.10.0/24 action=accept comment=wan-access
/ip firewall filter add chain=input src-address=10.99.0.0/24 action=accept comment=lan-access
/ip firewall filter add chain=input action=drop comment=drop-all

# NAT — clientes da LAN acessam internet via ether1
/ip firewall nat add chain=srcnat out-interface=ether1 action=masquerade

/ip firewall filter print
/ip firewall nat print
```

### Módulo 7 — QoS

```bash
/queue simple add name=limite-lab target=10.99.0.100 max-limit=2M/5M
/queue simple print stats
```

### Módulo 8 — Túneis

```bash
# PPPoE client
/interface pppoe-client add \
  name=pppoe-wan \
  interface=ether1 \
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

/ppp active print
```

### Ferramentas úteis durante o estudo

```bash
/tool torch interface=ether1           # tráfego WAN em tempo real
/tool torch interface=ether2           # tráfego LAN em tempo real
/tool traceroute 8.8.8.8
/log print follow
/log print where topics~"firewall"
/ip firewall filter print
/ip firewall connection print
/system resource print
/interface print stats
```

---

### Referências

- Apostila Opentech MTCNA — Fábio Bersan Rocha
- Curso Udemy: Mikrotik do básico ao avançado v2026 — Vitor Mazuco
- MikroTik Wiki: https://wiki.mikrotik.com
- WinBox / CHR download: https://mikrotik.com/download
- Forum MikroTik: https://forum.mikrotik.com

---

## 10. Histórico de Alterações

### 30/03/2026 — Topologia final com duas interfaces

**Alteração:** Adicionada segunda interface à VM do CHR para replicar o setup padrão ensinado nos cursos MikroTik (internet na ether1, LAN na ether2).

| | Antes | Depois |
|---|---|---|
| Interfaces na VM | 1 (`net0 → vmbr2`) | 2 (`net0 → vmbr2`, `net1 → vmbr99`) |
| ether1 | `192.168.10.200/24` (WAN) | `192.168.10.200/24` (WAN) |
| ether2 | não existia | `10.99.0.1/24` (LAN isolada) |
| Uso no curso | limitado — sem LAN separada | completo — replica setup dos cursos |

**Comandos executados:**
```bash
# No Proxmox
qm set 202 --net1 virtio,bridge=vmbr99
qm stop 202 && qm start 202

# No CHR
/interface set 0 name=ether1
/interface set 1 name=ether2
/ip address add address=10.99.0.1/24 interface=ether2
```

---

*Documentação revisada em 30/03/2026 | lucasfelz*
