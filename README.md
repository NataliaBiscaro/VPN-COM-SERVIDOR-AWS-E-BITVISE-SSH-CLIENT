# VPN COM SERVIDOR AWS E BITVISE SSH CLIENT
Este repositório contém o passo a passo para configurar uma conexão VPN utilizando um servidor AWS e Bitvise SSH CLIENT.
Aqui estou assumindo que você já tem uma servidor configurado na AWS e está acessando-o utilizando um IP Elástico, que é extremamente importante para que seu IP continue estático de modo que não gere problemas quando for necessária a conexão com a VPN.

## Passo 1: Atualizar os pacotes e instalar o OpenVPN e Easy-RSA
```
sudo apt update && sudo apt upgrade -y
sudo apt install openvpn easy-rsa -y
```
## Passo 2: Configuração do Easy-RSA
```
make-cadir ~/openvpn-ca
cd ~/openvpn-ca
```
### 2.1 Edite o arquivo "vars":
  
 - Dentro do VIM, localize as seguintes variáveis e faça as alterações conforme seu ambiente ou preferências: 
```
set_var EASYRSA_REQ_COUNTRY "US"
set_var EASYRSA_REQ_PROVINCE "California"
set_var EASYRSA_REQ_CITY "San Francisco"
set_var EASYRSA_REQ_ORG "Copyleft Certificate Co"
set_var EASYRSA_REQ_EMAIL "me@example.net"
set_var EASYRSA_REQ_OU "My Organizational Unit"


# País: "US" refere-se aos Estados Unidos.
# Estado/Província: "California" é o estado.
# Cidade: "San Francisco" é a cidade.
# Organização: "Copyleft Certificate Co" é o nome fictício da organização.
# Email: "me@example.net" é um email de contato fictício.
# Unidade Organizacional: "My Organizational Unit" refere-se a um departamento específico, como vendas ou engenharia
```

> Essas são variáveis de configuração utilizadas pelo Easy-RSA quando você cria certificados e chaves. Elas definem os valores padrão que são inseridos no campo "Distinguished Name" ou DN de um certificado.
>Quando você gera um certificado, essas informações são usadas para preencher os campos padrão do DN do certificado. Embora esses campos sejam tradicionalmente usados para identificar a entidade associada ao certificado, em muitos usos práticos da VPN, esses campos podem não ser tão críticos, a menos que você esteja planejando criar uma grande infraestrutura de PKI com certificados emitidos para várias entidades e departamentos.

### 2.2 Continue no diretório ~/openvpn-ca e crie a Autoridade de Certificação (CA):

```
./easyrsa init-pki
./easyrsa build-ca nopass
```

# Passo 3: Crie um certificado e uma chave privada para o servidor

## 3.1 Construa a chave Diffie-Hellman (usada em negociações de chave TLS):
```
./easyrsa gen-dh
```
## 3.2 Gere o certificado e a chave para o servidor:
```
./easyrsa build-server-full server nopass

```
> Neste comando, "server" é apenas um nome que estou usando para identificar o certificado do servidor. Você não será solicitado a entrar com uma senha devido ao argumento nopass.

## 3.3 Mova os arquivos relevantes para o diretório de configuração do OpenVPN:
```
sudo cp ~/openvpn-ca/pki/ca.crt /etc/openvpn/
sudo cp ~/openvpn-ca/pki/dh.pem /etc/openvpn/
sudo cp ~/openvpn-ca/pki/issued/server.crt /etc/openvpn/
sudo cp ~/openvpn-ca/pki/private/server.key /etc/openvpn/
```
> Agora temos o certificado e a chave privada do servidor, juntamente com os parâmetros Diffie-Hellman e o certificado CA, no diretório de configuração do OpenVPN.

# Passo 4: Configuração do servidor OpenVPN

## 4.1 Copie o arquivo de configuração para a pasta openvpn e edite-o:
```
sudo cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf /etc/openvpn/
sudo vim /etc/openvpn/server.conf
```
- Dentro do editor VIM, faça as seguintes modificações:
  
  - Localize a linha **dh dh2048.pem** e altere-a para **dh dh.pem**.
  - Descomente (remova o ; no início) da linha **push "redirect-gateway def1 bypass-dhcp"** para que o tráfego de todos os clientes passe pela VPN.
  - Descomente as linhas push **"dhcp-option DNS 208.67.222.222"** e push **"dhcp-option DNS 208.67.220.220"** para usar os servidores DNS da OpenDNS.
  - Localize a linha ***;user nobody** e a linha **;group nogroup**. Remova o ; no início dessas linhas para descomentá-las. Isso é feito para que o servidor OpenVPN não seja executado como o superusuário por razões de segurança.
  - Encontre a linha com **cipher AES-256-CBC** e adicione **data-ciphers AES-256-GCM:AES-128-GCM:AES-256-CBC**
` abaixo dela.

### Seu arquivo server.conf deve estar similar a este:

```
#################################################
# Basic OpenVPN 2.0 configuration for server.   #
#################################################

# The TCP/UDP port on which the server will listen.
port 5000

# Use the tun device for routing rather than a tap device for bridging.
dev tun

# Protocol used by the server, either tcp or udp.
proto udp

# The server's RSA private key and certificate, and the certificate of the CA.
ca /etc/openvpn/ca.crt
cert /etc/openvpn/server.crt
key /etc/openvpn/server.key  # This file should be kept secret

# Diffie-Hellman parameters.
dh /etc/openvpn/dh.pem

# Which IP address range to use for client addresses.
server 10.8.0.0 255.255.255.0

# The server will take 10.8.0.1 for itself, the rest will be made available to clients.
# This is an official IP address range for private networks, so it should be safe to use.
ifconfig-pool-persist ipp.txt

# Use network topology for subnet-based configurations.
topology subnet

# Push routes to the client to allow it to reach other private subnets behind the server.
push "route 10.8.0.0 255.255.255.0"

# Keep-alive mechanism to determine the availability of the server.
keepalive 10 120

# For extra security, create an "HMAC firewall" to block DoS attacks and UDP port flooding.
tls-auth /etc/openvpn/ta.key 0 # This file is secret

# Cipher for data channel packets. It is a good idea to use a strong cipher.
cipher AES-256-CBC
data-ciphers AES-256-GCM:AES-128-GCM:AES-256-CBC

# Log verbosity. Can be increased if you experience issues.
verb 3

# Optional: If you want clients to reach each other, uncomment the following line.
;client-to-client

# The persist options will try to avoid accessing certain resources on restart.
persist-key
persist-tun

# Address of the local endpoint. Can be used if your server is behind a NAT.
;local (seu_endereço_IP_aqui)

# Optional: Uncomment to redirect all the client's traffic through the VPN.
;push "redirect-gateway def1 bypass-dhcp"

```
   

# Passo 5: Inicie e habilite o serviço OpenVPN.

```
sudo systemctl start openvpn@server
sudo systemctl enable openvpn@server
sudo systemctl status openvpn@server
```




#### Com isso, o servidor OpenVPN está configurado e pronto para aceitar conexões, agora vamos para a segunda parte: 
---
# Configurações do cliente:

# Passo 1: Geração de Credenciais para o Cliente:

## 1.1 Vá até o diretório openvpn-ca:

```
cd ~/openvpn-ca 
```

## 1.2 Gere a solicitação de certificado e a chave privada para o cliente:
#### Escolha um nome identificável para este cliente, no exemplo abaixo, usei "meucliente", mas você pode usar outro nome se preferir.
```
./easyrsa gen-req meucliente nopass
```
## 1.3 Assine a solicitação de certificado para o cliente:
```
./easyrsa sign-req client meucliente

```
# Passo 2: Transferência das Chaves e Certificados para o Cliente:

## 2.1 Para configurar o cliente, é necessário ter os seguintes arquivos:

 - Certificado do cliente (meucliente.crt)
 - Chave privada do cliente (meucliente.key)
 - Certificado da CA (ca.crt)
 - Chave de autenticação TLS (ta.key)
 - 
## 2.2 Os mesmos podem ser encontrados nos seguintes locais:

- ~/openvpn-ca/pki/private/meucliente.key
- ~/openvpn-ca/pki/issued/meucliente.crt
- ~/openvpn-ca/pki/ca.crt
- /etc/openvpn/ta.key

## 2.3 Agora é preciso transferir estes arquivos para a máquina do cliente. 
> #### Lembrando que estou acessando o meu servidor usando o Bitvise SSH Client, e será desta forma que farei a transferência de arquivos, é importante ressaltar tambémm que irei realizar a conexão VPN diretamente da mesma máquina que está rodando o servidor. Se este não for o seu caso, procure a maneira ideal para realizar esta transferência de arquivos para seu cliente.

## 2.4 Mova os arquivos para uma pasta temporária:

```
sudo cp ~/openvpn-ca/pki/ca.crt /tmp/
sudo cp ~/openvpn-ca/pki/issued/meucliente.crt /tmp/
sudo cp ~/openvpn-ca/pki/private/meucliente.key /tmp/
sudo cp ~/openvpn-ca/ta.key /tmp/
```
> Após a execução desses comandos, os arquivos necessários estarão na pasta /tmp.

## 2.5  Mova os arquivos para a pasta de seu cliente:

- 1. Na página inicial do Bitvise, você encontrará ao lado esquerdo o ícone "New SFTP Window", clique nele.
- 2. O lado esquerdo mostra os diretórios e arquivos no seu computador local, enquanto o lado direito mostra os diretórios e arquivos no servidor.
- 3. Navegue no painel direito (servidor) até a pasta /tmp.
- 4. Selecione os arquivos que deseja transferir.
      - ca.crt: Certificado da Autoridade de Certificação (CA).
      - meucliente.crt: Certificado do cliente.
      - meucliente.key: Chave privada do cliente.
      - ta.key: Chave para autenticação TLS.
- 5. Arraste e solte os arquivos selecionados do painel direito para o painel esquerdo para a pasta "config" do OpenVPN, geralmente localizada no caminho `C:\Program Files\OpenVPN\config`

---

- **Caso a transferência apresente erros você precisará alterar a permissão dos arquivos e então tentar mover os arquivos novamente
"**
> Alterando as permissões: 
```
sudo chmod 644 /tmp/meucliente.crt
sudo chmod 644 /tmp/meucliente.key
sudo chmod 644 /tmp/ta.key
sudo chmod 644 /tmp/ca.crt
```
---

## Passo 3: Configuração do Cliente OpenVPN:

## 3.1 Instale o cliente OpenVPN em sua máquina local:
#### No caso do Window baixe o cliente OpenVPN GUI neste link: https://openvpn.net/community-downloads/

## 3.2 Crie um arquivo de configuração para o cliente:
### Na pasta `C:\..\OpenVPN\config` adicione um arquivo .ovpn com o seguinte conteúdo:
> Este arquivo pode ser criado em um bloco de notas e depois salvo na extensão desejada. Caso posteriormente queira modificar algo no arquivo, apenas altere a extensão para ".txt" novamente e após alterar, volte para  aextensão ".ovpn" novamente.

```

client
dev tun
proto udp
remote [IP_DO_SEU_SERVIDOR] [SUA_PORTA]
resolv-retry infinite
nobind
persist-key
persist-tun
ca ca.crt
cert meucliente.crt
key meucliente.key
tls-auth ta.key 1
cipher AES-256-CBC
verb 3

```

> Você deve substituir [IP_DO_SEU_SERVIDOR] pelo endereço IP do seu servidor ( sem as chaves [ ] ) e [SUA_PORTA] pela porta que o servidor OpenVPN está escutando ( sem as chaves [ ] ). Além disso, se você tiver diferentes nomes para os arquivos .crt e .key para clientes diferentes, você terá que ajustar os nomes meucliente.crt e meucliente.key apropriadamente.

---
- Uma breve explicação de cada linha do arquivo:
     - client             --> Indica que esta é uma configuração de cliente.
     - dev tun            --> Usa a interface TUN, que é um modo de funcionamento em nível IP.
     - proto udp          --> Especifica que o protocolo de transporte utilizado será UDP.
     - remote [IP_DO_SEU_SERVIDOR] [SUA_PORTA] --> Endereço IP e porta do servidor OpenVPN ao qual o cliente tentará se conectar.
     - resolv-retry infinite --> Se a tentativa inicial de resolução de nome falhar, o cliente tentará indefinidamente resolver o nome.
     - nobind             --> Não se liga a uma porta local específica.
     - persist-key        --> Não re-leia a chave privada durante re-negociações.
     - persist-tun        --> Não fecha e reabre a interface TUN/TAP ou encerra e reinicia o daemon de escuta ao reiniciar.
     - ca ca.crt          --> Caminho para o arquivo do certificado de autoridade.
     - cert meucliente.crt --> Caminho para o certificado público do cliente.
     - key meucliente.key --> Caminho para a chave privada do cliente.
     - tls-auth ta.key 1  --> Adiciona uma camada adicional de segurança HMAC ao canal de controle TLS. O parâmetro '1' significa que esta é uma configuração de cliente.
     - cipher AES-256-CBC --> Especifica o algoritmo de cifragem a ser usado.
     - verb 3             --> Define o nível de log. '3' fornece um nível de log de saída útil para diagnóstico.

---

## Passo 4: Conecte-se ao VPN:

## 4.1 Windows:

 - 1. Execute o OpenVPN GUI como administrador.
 - 2. Clique com o botão direito do mouse no ícone do OpenVPN na bandeja do sistema.
 - 3. Selecione 'Conectar'.

# Parabéns! Você se conectou a VPN com sucesso! 

---
# Certificando que consigo acessar um documento somente se conectado a VPN:

## Passo 1: Crie um arquivo .txt

```
echo "Este eh um arquivo secreto apenas para usuarios VPN, se voce esta conseguindo ver, sinta-se especial, Jesus te ama <3." > /home/ubuntu/secreto.txt
```

## Passo 2: Navegue até onde o arquivo "secreto.txt" está localizado:

```
cd /home/ubuntu/
```

## Passo 3: Inicie um servidor HTTP simples na porta 8000 para servir o arquivo criado:

```
python3 -m http.server 8000
```

## Passo 4: Com a VPN conectada, clique no link ou cole o seguinte endereço em seu navegador: 
[http://IP_DO_SEU_SERVIDOR:8000/secreto.txt](http://IP_DO_SEU_SERVIDOR:8000/secreto.txt)
Não esqueça de trocar "IP_DO_SEU_SERVIDOR" pelo número de IP do seu servior.
Com isto, se seu cliente estiver conectado, o arquivo de texto deverá ser exibido no navegador.


---
## Observações:
  Caso algum erro ocorra, você deve verificar as regras de segurança do seu servidor AWS da seguinte maneira:
  1. No painel da AWS, vá até a seção EC2 e depois para "Security Groups".
  2. Encontre o grupo de segurança associado à sua instância.
  3. Nas regras de entrada (Inbound rules), garanta que a porta 8000 esteja aberta para o seu endereço IP ou para o intervalo de IPs da VPN.
      - Tipo: Custom TCP.
      - Protocolo: TCP.
      - Range da Porta: 8000.
      - Origem: Escolha "My IP" para permitir apenas o seu endereço IP atual ou "Anywhere" para permitir de qualquer lugar (isso não é recomendado para ambientes de produção). Se você está usando a VPN, pode precisar especificar o intervalo de IP da sua VPN, o range normalmente utilizado é'10.8.0.0/24'

---

######  Material de apoio:
https://openvpn.net/community-resources/rsa-key-management/
https://openvpn.net/community-resources/how-to/













