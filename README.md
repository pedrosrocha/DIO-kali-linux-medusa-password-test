# Teste de Bruteforce (Kali + Medusa)

Visão geral

Este repositório descreve como eu validei, em um ambiente controlado e autorizado, a resistência de um software proprietário (desenvolvido em Flask) a ataques de força-bruta via formulário web. O teste foi realizado localmente usando uma máquina virtual com Kali Linux e a ferramenta Medusa. O objetivo foi didático: aprender a operar ferramentas de pentest em ambiente isolado e documentar o processo.

>Importante (ética e legal): todos os testes foram realizados somente contra a aplicação que eu mesmo desenvolvi e controlo (https://github.com/pedrosrocha/Testlink-clone
). Não execute ataques contra sistemas de terceiros sem autorização explícita e documentada.

## Repositório testado

*  App web (alvo do teste): Testlink-clone (Flask).
Repositório: https://github.com/pedrosrocha/Testlink-clone

(rodei a aplicação localmente)


## Ferramentas e ambiente
*   Kali Linux (máquina virtual): ambiente de ataque/controlado.

* Medusa: ferramenta de brute-force remoto (módulo web-form).

* Arquivos usados:

   - users.txt: lista de usuários a serem testados.

    - password.txt: lista de senhas a serem testadas.

    - output.txt: saída do medusa (capturada para análise).

A máquina com Kali foi conectada à mesma rede local da máquina onde a aplicação Flask estava rodando.


## Como pesquisei usuários e senhas mais comuns

Para criar listas representativas de credenciais populares, adotei a seguinte abordagem:

1. Fontes públicas de listas de senhas: consultei listas utilizadas em estudos e conjuntos de wordlists amplamente divulgados na comunidade.

2. Filtragem para o escopo do exercício: extraí as entradas mais frequentes (top 200), preferindo senhas simples e muito usadas (ex.: `123456`, `password`, `admin`, `etc`.).

3. Usuários: considerei nomes de usuário comuns (admin, root, test, user, guest) e variações óbvias (ex.: `admin`, `administrator`, `testuser`, `editor`).

4. Tamanho das listas: mantive users.txt e password.txt suficientemente curtas para testes rápidos em laboratório, mas com variedade suficiente para validar o processo.

## Comando Medusa utilizado
Executei o Medusa com o módulo web-form usando exatamente o comando abaixo (rodado no Kali):

```bash
    medusa -M web-form -h 192.168.18.16 -U users.txt -P password.txt -n 5000 -t 6 \
    -m FORM:"/login" \
    -m DENY-SIGNAL:"Invalid" \
    -m FORM-DATA:"post?username=&password=&submit=True" > output.txt
```


**Parâmetros chave**

 - `-M web-form` → usa o módulo de formulário web.

 - `-h 192.168.18.16` → endereço IP do servidor Flask na rede local.

 - `-U users.txt` e `-P password.txt` → listas de nomes de usuário e senhas.

 - `-n 5000` → porta para conectar ao software.

 - `-t 6` → threads simultâneas (controle de paralelismo).

 - `-m FORM:"/login"` → endpoint do formulário.

 - `-m FORM-DATA:"post?username=&password=&submit=True"` → template do corpo do POST (campo username, password e submit).

 - `-m DENY-SIGNAL:"Invalid"` → string que identifica falha de login na resposta (página retornada quando a combinação usuário/senha é inválida contém Invalid).

## Tratamento especial: resposta HTTP 302 (redirect)

Minha aplicação Flask (o Testlink-clone) responde com HTTP 302 (redirect) quando um login é bem-sucedido (redirecionando o usuário para a área autenticada). Por padrão o Medusa não interpreta 302 como uma “resposta válida de sucesso” para o módulo web-form — ele valida sucesso/erro baseado no conteúdo da página.

 - Configurei `-m DENY-SIGNAL:"Invalid"` para que o Medusa trate respostas contendo a string "Invalid" como **falha**.

 - Como o fluxo de sucesso da minha aplicação não contém "Invalid" e, em vez disso, retorna um redirect (302), a ausência da string "Invalid" na resposta foi utilizada como indicador implícito de sucesso.

 - Em outras palavras, inverti a checagem: se a resposta NÃO contiver "Invalid", isso é tratado como um possível sucesso e merece análise manual.

>Nota técnica: nem todas as ferramentas interpretam 302 como sucesso para web-form brute-force. Quando a aplicação usa redirect em logins válidos, é comum usar um DENY-SIGNAL (string identificadora de falha) ou um ALLOW-SIGNAL (se aplicável) para distinguir resultados. No meu caso, DENY-SIGNAL foi suficiente e eu tratei ausências da string como candidatos a sucesso.


### Como analisei os resultados (output.txt)

Após rodar o medusa, analisei output.txt para encontrar tentativas que não continham a string de falha (Invalid) — essas entradas são candidatas a logins válidos (ou, ao menos, respostas diferentes que exigem investigação manual).

Exemplo simples de filtragem (executado no Kali):


```bash
    # Exibe linhas que não contenham a palavra "Invalid"
    grep -v "Invalid" output.txt | less
```

Em seguida, validei manualmente as combinações encontradas (tentando o login no navegador ou via curl) para confirmar que o redirect 302 realmente significava autenticação bem-sucedida e que o usuário/senha identificados eram válidos.

### Configuração da VM (Kali) e rede local
Resumo das etapas que usei para preparar o ambiente de teste:

1. Máquina alvo (Flask):

 - Rodei a aplicação Flask no host com IP 192.168.18.16.

 - Assegurei que a rota /login estava ativa e que não havia firewalls bloqueando o acesso do Kali.

2. Máquina atacante (Kali VM):

 - Instalei o medusa (apt install medusa quando necessário).

 - Configurei adaptador de rede da VM para Bridged (ou NAT com portas apropriadas) de forma que a VM pudesse alcançar 192.168.18.16.

 - Posicionei users.txt, password.txt na VM.

3. Teste controlado:

 - Fiz testes de conectividade (curl) antes de executar ataques automatizados.






### Resultados e validação

 - Identifiquei combinações candidatas (linhas em output.txt sem o DENY-SIGNAL).

 - Validei manualmente que, ao usar essas credenciais, a aplicação respondia com 302 e redirecionava para a área autenticada — confirmando sucesso do login.

 - Documentei as credenciais válidas no relatório do curso (apenas para fins de auditoria interna / aprendizado).

### Recomendações gerais (segurança)

 - Autenticação segura: evitar usar redirecionamentos sem controle de conteúdo de resposta. Sempre considerar retornar um corpo de resposta que permita distinção segura por ferramentas de verificação.

 - Bloqueio de tentativas: implementar mecanismos de mitigação (limite de tentativas, captchas, bloqueio por IP) para reduzir risco de força-brute.

 - Logs e alertas: registrar tentativas repetidas de login e gerar alertas para administradores.

 - Senhas fortes: forçar políticas de senha e incentivação ao uso de MFA.

## Conclusão

Este README documenta o processo utilizado para testar (de modo autorizado e controlado) a resistência de uma aplicação Flask a ataques por força-bruta usando Medusa em uma VM Kali. O ponto técnico mais relevante deste teste foi a necessidade de contornar a interpretação de HTTP 302 como “sucesso” pelo Medusa, usando DENY-SIGNAL e analisando a ausência da string de falha para identificar candidatos a credenciais válidas.





