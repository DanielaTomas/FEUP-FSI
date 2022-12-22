# LOGBOOK 11

### Lab Environment Setup

Completamos todos os pontos presentes no guião deste lab, isto é, DNS Setup e Container Setup and Commands.

## Task 1

Por padrão, OpenSSL usa o arquivo de configuração /usr/lib/ssl/openssl.cnf. Como precisamos fazer alterações neste arquivo, vamos copiá-lo no nosso diretório atual e instruir o OpenSSL a usar essa cópia.
A seção [CA default] do arquivo de configuração mostra a configuração padrão que precisamos preparar. Precisamos criar vários subdiretórios. Descomentamos a linha de "unique_subject" para permitir a criação de certificações com o mesmo assunto.

![](https://i.imgur.com/uSz61iW.png)

Para o arquivo index.txt, criamos um arquivo vazio. Para o arquivo serial, colocamos um único número em formato de string. Depois de definir o arquivo de configuração openssl.cnf, conseguimos criar e emitir certificados.

![](https://i.imgur.com/XZdlwez.png)

Executamos o comando ` openssl req -x509 -newkey rsa:4096 -sha256 -days 3650 -keyout ca.key -out ca.crt` para gerar o certificado *self-assigned* para a CA:

![](https://i.imgur.com/nQLp3mi.png)

![](https://i.imgur.com/kLwxXqq.png)

Utilizamos os seguintes comandos para examinar o conteúdo decodificado do certificado X509 e a chave RSA (-text significa decodificar o conteúdo em texto simples; -noout significa não imprimir a versão codificada):
```
openssl x509 -in ca.crt -text -noout
openssl rsa  -in ca.key -text -noout
```

[Ficheiro com certificado de chave pública- ca.crt](ca_crt.txt)
[Ficheiro com chave privada da CA](ca_key.txt)

**Questions:**
**1) What part of the certificate indicates this is a CA’s certificate?!**

![](https://i.imgur.com/W2VrYgF.png)

"CA:TRUE" indica que é um certificado da CA.

**2) What part of the certificate indicates this is a self-signed certificate?**

![](https://i.imgur.com/mCuW61B.png)

Quando o X509v3 Subject Key Identifier é igual ao X509v3 Authority Key Identifier, estamos perante um certificado *self-assigned*.

**3) In the RSA algorithm, we have a public exponent e, a private exponent d, a modulus n, and two secret numbers p and q, such that n = pq. Please identify the values for these elements in your certificate and key files.**

public exponent - e:
![](https://i.imgur.com/5imGwQZ.png)

private exponent - d:
![](https://i.imgur.com/pF6bzBb.png)

modulus - n: 
![](https://i.imgur.com/pyFOMD5.png)

secret number 1 - p:
![](https://i.imgur.com/WTwTXP3.png)

secret number 2 - q:
![](https://i.imgur.com/CfzsAnb.png)

## Task 2

Com o seguinte comando geramos um CSR(Certificate Signing Request) para www.bank32.com (utilizamos o nome de servidor presente no guião):
```
openssl req -newkey rsa:2048 -sha256 \
            -keyout server.key   -out server.csr \
            -subj "/CN=www.bank32.com/O=Bank32 Inc./C=US" \
            -passout pass:dees
```

![](https://i.imgur.com/r4NdR4o.png)

Com os seguintes comandos visualizamos o conteúdo decodificado do CSR e dos arquivos de chave privada:
```
openssl req -in server.csr -text -noout
openssl rsa -in server.key -text -noout
```

Adicionamos dois nomes alternativos ao pedido de assinatura de certificado porque são necessários para as tarefas seguintes. Esta etapa foi incluída no *printscreen* colocado acima.

## Task 3

O seguinte comando transforma o pedido de assinatura de certificado (server.csr) num certificado X509 (server.crt), urilizando ca.crt e ca.key da CA:
```
openssl ca -config myCA_openssl.cnf -policy policy_anything \
           -md sha256 -days 3650 \
           -in server.csr -out server.crt -batch \
           -cert ca.crt -keyfile ca.key
```

![](https://i.imgur.com/KJj72w9.png)

Por motivos de segurança, a configuração padrão em openssl.cnf não permite que o comando "openssl ca" copie o campo de extensão do pedido para o certificado final. Para habilitarmos isso, descomentamos a seguinte linha na nossa cópia do arquivo de configuração:

![](https://i.imgur.com/sUGRkwo.png)

Depois de assinarmos o certificado, utilizamos o comando `openssl x509 -in server.crt -text -noout` para imprimir o conteúdo decodificado do certificado e concluímos que os nomes alternativos estão incluídos.

![](https://i.imgur.com/5rZXejp.png)

[server.crt](server_crt.txt)

## Task 4

Através do exemplo do guião, configuramos o nosso próprio site HTTPS.

Demos build, executamos o nosso container Docker e posto isto executamos os seguintes comandos:
```
# a2enmod ssl                 // Enable the SSL module
# a2ensite bank32_apache_ssl  // Enable the sites described in this file
```
O primeiro comando servem para ativar o módulo Apache ssl e o segundo para ativar o site. Estes comandos também são executados no processo de built do container Docker.

O servidor Apache não é iniciado automaticamente no container, porque é preciso introduzir a password para desbloquear a chave privada. Executamos, no container, o comando `# service apache2 start` para iniciar o servidor. Os comandos `# service apache2 stop` e `# service apache2 restart` servem para parar e para reiniciar o servidor, respetivamente.

![](https://i.imgur.com/DzeLIFX.png)

Quando o servidor Apache é iniciado, precisa carregar a chave privada para cada site HTTPS. A nossa chave privada é encriptada, logo, o servidor vai pedir a password para desencriptá-la. Dentro do container, a password utilizada é dees (para este exemplo bank32).

![](https://i.imgur.com/c7mnXXe.png)

Não conseguimos aceder ao site, porque precisamos de carregar o certificado para o firefox. Carregamos o certificado como é indicado no guião e assim conseguimos aceder com o certificado.

![](https://i.imgur.com/89RpjQk.png)

![](https://i.imgur.com/XDGcv2I.png)

## Task 5
O objetivo desta tarefa é mostrar como PKI pode impedir ataques Man-In-The-Middle.

- Selecionamos www.fnac.com como o nosso target website.

- Utilizamos o mesmo certificado do nosso próprio servidor da tarefa anterior, tal como indicado no guião.

![](https://i.imgur.com/j2w5NZz.png)

- Alteramos o ficheiro etc/hosts e o bank32_apache_ssl.conf para quando acedermos ao site www.fnac.com, sermos redirecionados para o nosso servidor malicioso.

![](https://i.imgur.com/QGpENTm.png)
![](https://i.imgur.com/KytT4DC.png)

- Este é o resultado da tentativa de aceder ao site www.fnac.com que resultou num warning por causa do certificado que geramos antes para o www.bank32.com.

![](https://i.imgur.com/t9qkMZs.png)


## Task 6

Nesta tarefa geramos um novo certificado porque quando uma CA é comprometida, nós próprios podemos assinar certificados e fazer-nos passar por outros *sites*.

![](https://i.imgur.com/PR3pvpp.png)

![](https://i.imgur.com/Zj241qU.png)

Assim, já é possível aceder ao site sem que apareça um warning:

![](https://i.imgur.com/caWMZDF.png)

## CTFs

### Desafio 1

Depois de nos conectarmos ao servidor, verificámos que este envia sempre uma flag cifrada com RSA.

![](https://i.imgur.com/nByZ7BF.png)

Para decifrar a flag, usámos o seguinte script:

```python=
from binascii import hexlify, unhexlify
from sympy import *

p = nextprime(2**512) # next prime 2**512
q = nextprime(2**513) # next prime 2**513
n = p*q
e = 0x10001 # a constant -> expoente público
d = pow(e,-1,((p-1)*(q-1))) # a number such that d*e % ((p-1)*(q-1)) = 1 -> expoente privado

enc_flag = b"00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000004c8ac15da60342876d0e4da23691144ce8ff26ee9886dea91a64c8dea863e3653499857f5ebb01d8b29a8b70c2bafb7feae0c4026235f1091695156112ddb317f0a1271e0348119f7d1a770b8d6f05fcc6030acd42ecd1c7f6974cdd28c4c566882c781ca5b53754b83599330feec670f3b22553bc735f78f594bad5f8365023"

def enc(x):
	int_x = int.from_bytes(x, "big")
	y = pow(int_x,e,n)
	return hexlify(y.to_bytes(256, 'big'))

def dec(y):
	int_y = int.from_bytes(unhexlify(y), "big")
	x = pow(int_y,d,n)
	return x.to_bytes(256, 'big')

y = dec(enc_flag)
print(y.decode())

```
Primeiro, usámos a função *nextprime* do python para encontrar os números primos *p* e *q* secretos.  Em seguida, sabendo o expoente público *e* (0x10001) e usando os valores de *p* e *q*, o expoente privado *d* foi calculado usando a fórmula ``` d = (1/e) mod ((p-1)*(q-1))) ```. A variável *enc_flag* contém a mensagem que queremos decifrar que, neste caso, é a flag enviada pelo servidor. Quando corremos o programa, a flag aparece decifrada:

![](https://i.imgur.com/JVp6Pi2.png)

### Desafio 2
