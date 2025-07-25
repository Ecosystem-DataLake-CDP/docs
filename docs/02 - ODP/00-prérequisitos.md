# 00 • Pré-requisitos

Este documento descreve todos os requisitos mínimos — de infraestrutura, software e acesso — para que a instalação da OpenSource Data Platform (ODP) 1.2.4.0 seja feita em qualquer ambiente Oracle Cloud Infrastructure (OCI) ou bare metal compatível.

Nos tópicos 01, 02, 03 e 04 abaixo, temos algumas informações pertinentes de revisão e conhecimento geral sobre o processo, portanto, é interessante que o leitor confirme que suas máquinas têm capacidade de hardware suficiente e que caso seja de seu interesse, entre nas documentações oficiais da Clemlab e da OCI para mais detalhes.

## 1. Infra-estrutura mínima

Abaixo está uma listagem de recapitulação do que é feito na documentação referente à OCI, portanto, verifique está de acordo com estes padrões.

| Papel do nó | OCPU mín. | Memória mín. | SO homologado |
|-------------|-----------|--------------|---------------|
| master      | 1 OCPU    | 6 GiB       | Ubuntu 22.04 LTS / Oracle Linux 8 |
| node × 3  | 1 OCPU    |  6 GiB       | Ubuntu 22.04 LTS / Oracle Linux 8 |

Obs: Recomendo o uso do Ubuntu pois os próximos passos foram realizados numa máquina Ubuntu.

> Substitua as quantidades conforme sua carga de trabalho. No OCI, o shape de referência é **VM.Standard.E4.Flex** com 2-4 OCPU e 8-16 GiB de RAM.

### Armazenamento

* Disco do SO: 50 GB (SSD ou Block Volume)
* Dados Hadoop/Hive: mínimo 100 GB por nó (Block Volume ou armazenamento local)

## 2. Rede e DNS

| Item                              | Valor genérico                       |
|-----------------------------------|--------------------------------------|
| VCN CIDR                          | `<CIDR_VCN>` (ex.: 10.0.0.0/16)      |
| Subnet pública CIDR               | `<CIDR_PUBLICA>` (ex.: 10.0.1.0/24)  |
| Subnet privada CIDR (dados)       | `<CIDR_PRIVADA>` (ex.: 10.0.2.0/24)  |
| Security List / NSG – portas IN   | 22/tcp, 8080/tcp, 8440-8441/tcp      |
| Security List – portas OUT        | 80/tcp, 443/tcp (ou mirror local)    |
| DNS interno                       | `<dominio_local>` (ex.: clemlab.local) |

## 3. Acesso à Internet ou repositório offline

A instalação pode ser feita de duas formas:

1. **Online:** permitindo saída HTTP/HTTPS (80/443) para `archive.clemlab.com`.
2. **Offline:** baixando previamente:
   * `repos-ambari.tar.gz`
   * `repos.tar.gz`
   * `repos-utils.tar.gz`

Em cenários offline, copie os arquivos para `/opt/odp-repo/` de **todos** os nós e aponte o APT/YUM para o caminho local. Neste documento, iremos consumir o repositório fonte da Clemlab, inicialmente.

## 4. Usuários, chaves e permissões

* Usuário padrão do SO com privilégio sudo (`ubuntu`).
* Chave SSH RSA/ECDSA carregada no OCI **ou** distribuída manualmente (`~/.ssh/id_rsa`).
* Fuso horário configurado e serviço NTP ativo (`chrony` ou `systemd-timesyncd`).

## 5. Dependências do sistema operacional

5. Sistema Operacional – Dependências e Ajustes Finos

A instalação do ODP implica requisitos adicionais além da simples presença do Java e de utilitários básicos. As seções abaixo compilam todos os pontos listados em *Operating-System Requirements* do guia Clemlab (versão 1.2.4.0) e boas práticas específicas para Hadoop.

> Todos os comandos devem ser executados em **todos** os nós (master e workers) com privilégio sudo.
> Onde aparecer `<valor>`, substitua pelo parâmetro que se aplica ao seu ambiente.

### 5.1 Pacotes de sistema
No Ubuntu 22.04 LTS, antes de instalar qualquer pacote, realize uma atualização geral do sistema operacional, em todas as máquinas:
```bash
sudo apt update -y && sudo apt upgrade -y
sudo reboot
```

```bash
sudo apt update && sudo apt install -y openjdk-8-jdk-headless python3 curl wget gnupg lsb-release net-tools libaio1 openssh-server openssh-client chrony
```

### 5.2 Ajustes de kernel e memória

Crie o arquivo `/etc/sysctl.d/99-odp.conf` em todos os nós com o conteúdo abaixo — ele aplica os parâmetros recomendados pelo guia Clemlab para o Hadoop Stack 1.2.4.0:

### 5.3 Desativar Transparent Huge Pages (THP)
```bash
cat <<'EOF' | sudo tee /etc/sysctl.d/99-odp.conf
```
Insira o texto abaixo no arquivo que foi criado:
```text
# ——— Desempenho e estabilidade para Hadoop / Spark ———
vm.swappiness                 = 1
vm.overcommit_memory          = 1
vm.max_map_count              = 262144

# Buffers de rede
net.core.somaxconn            = 1024
net.core.netdev_max_backlog   = 5000
net.ipv4.tcp_tw_reuse         = 1
net.ipv4.ip_local_port_range  = 10240 65535

# Desativa IPv6 (opcional)
net.ipv6.conf.all.disable_ipv6     = 1
net.ipv6.conf.default.disable_ipv6 = 1
```
Digite EOF ao final, para que entenda "End Of File".

```bash
sudo sysctl --system
```


### 5.3.2 Definir limites de arquivos e processos
Ainda em cada nó, crie o arquivo de limites:
```bash
sudo tee /etc/security/limits.d/99-odp.conf <<'EOF'
```
Insira no arquivo as linhas abaixo:
```text
* soft  nofile 65536
* hard  nofile 65536
* soft  nproc  65536
* hard  nproc  65536
```
Para garantir que o módulo pam_limtes está habilitado para novas sessões, dê o seguinte comando:
```bash
grep -q pam_limits.so /etc/pam.d/common-session || \
echo "session required pam_limits.so" | sudo tee -a /etc/pam.d/common-session
```

## 6. Sincronização de tempo
```bash
sudo apt update
sudo apt install -y chrony
sudo systemctl enable --now chrony
sudo systemctl status chronyd
chronyc sources -v # verificação
```
## 7. Portas e firewall

Em cada instância:

Ubuntu 22.04 (UFW/iptables):

```bash
sudo ufw allow 8080/tcp # Ambari Web
sudo ufw allow 8440:8441/tcp # Ambari Agent ↔ Server
sudo ufw reload
```
## 8. Definição do Hostname em Cada Máquina

Para garantir que cada nó do cluster seja identificado corretamente na rede interna, defina o hostname de cada máquina conforme a função que ela desempenha. Execute o comando correspondente **em cada nó**:

- No master:
```bash
sudo hostnamectl set-hostname master.clemlab.local
```

- No node1:
```bash
sudo hostnamectl set-hostname node1.clemlab.local
```

- No node2:
```bash
sudo hostnamectl set-hostname node2.clemlab.local
```

- No node3:
```bash
sudo hostnamectl set-hostname node3.clemlab.local
```
Após isso, por garantia, reinicialize as máquinas, para que as definições de hostname certamente estejam em vigor.

## 9. Configuração de DNS Interno (/etc/hosts)

Garanta que todos os nós resolvam corretamente os nomes internos do cluster.
Edite o arquivo `/etc/hosts` em **todos os nós** e adicione (ajuste os IPs conforme sua rede):
Exemplo com IP's privados quaisquer:

```bash
10.0.0.10  master.clemlab.local master
10.0.0.11  node1.clemlab.local  node1
10.0.0.12  node2.clemlab.local  node2
10.0.0.13  node3.clemlab.local  node3
```
Após isso, verifique em cada nó:
```bash
hostname -f
ping master
ping node1
ping node2
ping node3
```

## 10. Acesso SSH sem Senha (Password-less SSH)

Para que o Ambari Server possa instalar automaticamente os agentes e orquestrar comandos em todos os nós, é imprescindível habilitar acesso SSH sem senha a partir do nó **master** para cada **node**.

Detalho ainda que caso o procedimento abaixo não funcione adequadamente, recomendo que vá em [Problemas conhecidos](./XX-problemas-conhecidos.md), e verifique a configuração de SSH alternativa, mas com ações manuais em cada nó.

> No laboratório utilizamos o usuário padrão `ubuntu`, já pertencente ao grupo *sudo*.
> Caso utilize outro usuário, certifique-se de conceder sudo *NOPASSWD* em `/etc/sudoers`.

### 10.1 Verificar se o OpenSSH está instalado
```bash
sudo apt install -y openssh-server openssh-client
```

### 10.2 Gerar o par de chaves no nó *master*
```bash
ssh-keygen -b 4096 -t rsa -C "odp-cluster" -N "" -f ~/.ssh/id_rsa
```
* Aceite o caminho padrão `~/.ssh/id_rsa`.
* A opção `-N ""` cria a chave sem passphrase, requisito para automação.

### 10.3 Copiar a chave pública manualmente para cada nó

Copie o conteúdo da saída (linha única que começa com `ssh-rsa`).
Na sua máquina Master, pegue sua chave pública com o seguinte comando:

```bash
cat ~/.ssh/id_rsa.pub
```
### 10.4 Ajuste das permissões dos arquivos de chave SSH

Após copiar a chave pública do master para cada nó, **em cada nó** (master, node1, node2, node3), execute:

```bash
echo "<your-public-key>" >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chown ubuntu:ubuntu ~/.ssh ~/.ssh/authorized_keys
```
---

## 11. Configuração do iptables (Firewall)

Para que o Ambari possa se comunicar corretamente durante a configuração e operação do cluster, é fundamental garantir que as portas necessárias estejam abertas em todas as máquinas (master, node1, node2, node3).

### 11.1 Opção rápida: Desabilitar temporariamente o firewall

Execute em **todos os nós**:

```bash
sudo ufw disable
sudo iptables -X
sudo iptables -t nat -F
sudo iptables -t nat -X
sudo iptables -t mangle -F
sudo iptables -t mangle -X
sudo iptables -P INPUT ACCEPT
sudo iptables -P FORWARD ACCEPT
sudo iptables -P OUTPUT ACCEPT
```
No Ubuntu, mesmo com o firewall desativado, ainda é necessário que seja realizada uma validação para que as portas das máquinas, todas as máquinas, estejam liberadas, portanto, realize os comandos abaixo também:

```bash
sudo ufw allow 22/tcp # SSH
sudo ufw allow 8080/tcp # Ambari Web
sudo ufw allow 8440/tcp # Ambari Agent
sudo ufw allow 8441/tcp # Ambari Agent
sudo ufw reload

sudo iptables -I INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -I INPUT -p tcp --dport 8080 -j ACCEPT
sudo iptables -I INPUT -p tcp --dport 8440 -j ACCEPT
sudo iptables -I INPUT -p tcp --dport 8441 -j ACCEPT
sudo netfilter-persistent save # ou sudo service iptables save
```
Por fim, apenas por precaução, realize o (`sudo ufw disable`) para certificar-se de que o firewall está desativado, este atrapalha a instalação do Ambari e ODP e as configurações de segurança foram realizadas na OCI.

---
Após cumprir todos os itens acima, prossiga para `02 - ODP/01-configuracao-repositorio.md`, onde será configurado o repositório e iniciada a instalação do Ambari.





