# Projeto de Infraestrutura: Integração do OPNsense com Active Directory e SSL VPN

## 1. Visão Geral do Projeto

Este projeto documenta a implementação de uma infraestrutura de rede segura utilizando o **OPNsense** como firewall/gateway central e o **Microsoft Active Directory (AD)** como controlador de domínio e provedor de identidade (IdP). O objetivo principal é estabelecer um ambiente corporativo simulado com gerenciamento centralizado de usuários, resolução de nomes híbrida, segurança de perímetro com IDS/IPS e acesso remoto seguro via OpenVPN com autenticação baseada em grupos do AD.

## 2. Topologia de Rede e Tecnologias

- **Firewall/Gateway:** OPNsense 24.x
- **Controlador de Domínio:** Windows Server 2022 (Active Directory Domain Services)
- **Segmentação de Rede:**
  - **LAN:** 172.16.0.0/24 (Gateway: 172.16.0.254)
  - **VPN Roadwarrior:** 10.0.8.0/24
- **Serviços Críticos:** DNS Unbound, Servidor DHCP (AD), OpenVPN, IDS/IPS (Suricata).

## 3. Implementação do Firewall OPNsense

### 3.1. Instalação Inicial e Configuração (Assistente)

A instalação seguiu um padrão BSD *hardened*. Após a inicialização inicial, as seguintes configurações foram aplicadas através do Assistente:

- **Hostname/Domínio:** Definido como `fw-core.mydomain.com`. O uso de um FQDN é essencial para a emissão correta de certificados SSL.
- **DNS Externo:** Configurado com o IP `1.1.1.1` (Cloudflare) para resolução upstream.
- **Interface WAN (em0):** Configurada via DHCP para receber o IP do provedor/hypervisor.
- **Interface LAN (em1):** Definida com IP estático **172.16.0.254/24**.
  - *Nota Técnica:* O Servidor DHCP do OPNsense foi explicitamente **desativado** nesta interface para evitar conflitos (Split-Brain) com o serviço DHCP do AD.
- **Otimização da Pilha:** O IPv6 foi desativado em `Interfaces > WAN > IPv6 Configuration Type: None` para reduzir a superfície de ataque e simplificar o roteamento interno.

### 3.2. Configuração de DNS e Visibilidade

Para garantir o monitoramento interno e a resolução de nomes:

1.  **Encaminhamento de Consultas:** O OPNsense foi configurado para encaminhar consultas específicas para o resolvedor local em execução no host (192.168.0.4), garantindo a resolução recursiva.
2.  **Logs:** O log detalhado foi ativado em `Reporting > Unbound DNS` para auditoria de solicitações de domínios maliciosos.

## 4. Configuração do Active Directory (AD DS e AD CS)

O Controlador de Domínio atua como a "fonte da verdade" para a rede.

1.  **Ajustes de Rede (DC):**
    - Endereço IP: 172.16.0.1
    - Gateway: 172.16.0.254 (aponta para o OPNsense).
2.  **Servidor DHCP:** Configurado para a LAN, definindo o IP do OPNsense (172.16.0.254) como o **Roteador (Gateway)** para os clientes.
3.  **Encaminhador DNS:** No console DNS do Windows, configurado um *Forwarder* para o IP do OPNsense. Isso garante que as consultas de domínios externos passem pelo filtro de segurança do firewall.
4.  **AD CS (Active Directory Certificate Services):** Instalado e configurado para permitir LDAPS (LDAP sobre SSL). Embora o laboratório utilize a porta 389 para testes iniciais, a infraestrutura está preparada para a migração para a porta 636 (Produção).

## 5. Integração OPNsense + Active Directory (LDAP)

Para permitir que o firewall "enxergue" os usuários do domínio, configuramos um servidor de autenticação externo:

- **Caminho:** `System > Access > Servers`
- **Nome Descritivo:** `AD_Corp`
- **Tipo:** LDAP
- **Hostname:** 172.16.0.1 (IP do DC)
- **Porta:** 389 (TCP Padrão)
- **Credenciais de Bind:**
  - DN do Usuário: `CN=opnsense,OU=Service Accounts,DC=mydomain,DC=com`
  - *Nota:* Uma conta de usuário comum sem privilégios administrativos, seguindo o **Princípio do Menor Privilégio**.
- **Base de Busca (DN):** `DC=mydomain,DC=com`
- **Opções de Sincronização:** Ativado `Synchronize groups` para importar grupos de segurança do AD para o OPNsense, facilitando o controle de acesso à VPN.
- **Propriedades de Leitura:** Ativado.

## 6. Implementação de VPN de Acesso Remoto (OpenVPN)

A solução de VPN foi projetada para oferecer segurança robusta e facilidade de conexão.

### 6.1. Infraestrutura de Chaves Públicas (PKI)

1.  **Autoridade de Certificação (CA):** Criada a `CA_FW` internamente no OPNsense.
2.  **Certificado de Servidor:** Criado o `CERT-SSLVPN-SERVER`, assinado pela `CA_FW`.
3.  **Certificado de Cliente:** Criado o `CERT-SSLVPN-CLIENT` para autenticação baseada em certificado (Fator Duplo: Certificado + credenciais do AD).
4.  **Chave Estática TLS:** Gerada a chave `SSLVPN` para criptografar o canal de controle e proteger contra ataques DoS.

### 6.2. Configuração da Instância OpenVPN

- **Protocolo/Porta:** UDP 1194 (Melhor desempenho para túneis).
- **Modo:** TUN (Camada 3 - Roteado).
- **Rede do Túnel (Rede Virtual IPv4):** 10.0.8.0/24.
- **Autenticação:** Selecionado o servidor `AD_Corp`.
- **RBAC (Controle de Acesso Baseado em Função):** Opção `Enforce local group` definida para o grupo `SSLVPN`. Apenas usuários do AD pertencentes a este grupo podem fechar a conexão.
- **DNS Push:** Configurado para enviar o domínio `mydomain.com` e o IP `10.0.8.1` para os clientes, garantindo que os usuários remotos resolvam nomes internos do AD.

### 6.3. Chave TLS Estática

Para aumentar a segurança contra negação de serviço (DoS) e varreduras de porta, uma chave TLS estática foi implementada.

### 6.4. Regras de Firewall

Regras específicas foram criadas para permitir o tráfego:

1.  **WAN:** Permitir tráfego na porta UDP 1194 para o IP da interface WAN.
2.  **LAN:** Permitir DNS (porta 53) para o DC e o firewall, bem como navegação web.
3.  **Interface OpenVPN:** Permitir tráfego da interface `OpenVPN net` para os recursos necessários da LAN.

## 7. Segurança Avançada e Monitoramento

### 7.1. IDS/IPS (Suricata)

Ativado o sistema de detecção e prevenção de intrusões (Suricata) nas interfaces de borda. Isso permite a inspeção profunda de pacotes (DPI) para detectar malware, exploits e varreduras de rede.

### 7.2. Exportação de Cliente e Testes

Para o provisionamento de usuários:

1.  Utilizado o módulo **Client Export**.
2.  Definido o Hostname como o endereço IP público/DNS dinâmico do laboratório.
3.  Adicionadas diretivas personalizadas ao arquivo `.ovpn`:
    ```bash
    ping 10
    ping-restart 60
    dhcp-option ADAPTER_DOMAIN_SUFFIX mydomain.com
    ```
    **Validação:** O teste foi realizado a partir de uma rede externa (VM isolada). O usuário autenticou-se usando credenciais do AD, recebeu um IP da rede 10.0.8.0/24 e conseguiu pingar e acessar serviços no domínio `mydomain.com` com sucesso, validando a integração.

## 8. Validação e Resultados

O teste final envolveu a conexão de um cliente fora da rede local usando credenciais do AD.

- **Status:** Conectado com sucesso.
- **IP Recebido:** 10.0.8.x.
- **Resolução de Nomes:** O cliente foi capaz de resolver nomes internos através do DNS push configurado.

## 9. Conclusão

Este projeto demonstra a viabilidade do uso de ferramentas de código aberto de alto desempenho (OPNsense) integradas com ecossistemas Microsoft Windows. A implementação garante uma postura de segurança sólida, com logs de DNS detalhados, proteção perimetral ativa e acesso remoto escalável e seguro, alinhado com as melhores práticas de arquitetura de redes corporativas.

**Documentado por:** [0xK4z]
**Tecnologias:** OPNsense, Active Directory, OpenVPN, LDAP, PKI.
