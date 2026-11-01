# tcp-distributed-nodes

Simulação de um sistema distribuído em Python: um coordenador TCP e quatro nós (A, B, C e D) que enviam heartbeats, mantêm uma cópia versionada do estoque de um produto e mostram como uma falha é detectada por timeout e como o nó que caiu se recupera.

Trabalho da disciplina **Plataformas e Sistemas de Alto Desempenho (AVA1)**, feito a partir da Prática 1, "Simulação de sistemas distribuídos com processos e sockets TCP". A versão original da prática tinha três nós (A, B e C); este repositório acrescenta o nó D.

## O que foi alterado

O histórico tem o código original no primeiro commit e a alteração no segundo, então o diff mostra exatamente as linhas mudadas. O nó D entrou nos três pontos onde a lista de nós aparece:

1. na validação das mensagens `REGISTRAR` e `PULSO`, em `EstadoCompartilhado.tratar`, que só aceita nomes conhecidos;
2. no laço que monta a resposta do `STATUS`;
3. no laço da função `main` que cria os processos.

O texto do passo 1 também passou de "três" para "quatro" processos. O protocolo, o timeout de 2 s e a lógica de versões não mudaram, porque já funcionam para qualquer quantidade de nós.

## Como executar

Requisitos: Python 3.9 ou superior. Não há dependências externas, só a biblioteca padrão.

```
python pratica1_tcp_processos.py
```

No Windows, se necessário, troque `python` por `py`. Tudo roda em 127.0.0.1, então o firewall precisa permitir conexões TCP locais. A demonstração leva cerca de 6 segundos.

## Saída esperada

Execução no Windows 11 com Python 3.14 (a porta e os PIDs mudam a cada execução):

```
[11:53:13] Coordenador TCP ativo em 127.0.0.1:53358
[11:53:14] Nó A: iniciado em PID=21696
[11:53:14] Nó B: iniciado em PID=22756
[11:53:14] Nó A: versão=0; réplica={'produto': 'livro', 'estoque': 10}
[11:53:14] Nó B: versão=0; réplica={'produto': 'livro', 'estoque': 10}
[11:53:14] Nó C: iniciado em PID=26160
[11:53:14] Nó D: iniciado em PID=19012
[11:53:14] Nó C: versão=0; réplica={'produto': 'livro', 'estoque': 10}
[11:53:14] Nó D: versão=0; réplica={'produto': 'livro', 'estoque': 10}
[11:53:14] 1. Estado inicial dos quatro processos: {'A': 'ATIVO', 'B': 'ATIVO', 'C': 'ATIVO', 'D': 'ATIVO'} | dados={'produto': 'livro', 'estoque': 10} (v0)
[11:53:14] Professor: estoque alterado para 7 (v1)
[11:53:14] Nó A: versão=1; réplica={'produto': 'livro', 'estoque': 7}
[11:53:14] Nó C: versão=1; réplica={'produto': 'livro', 'estoque': 7}
[11:53:14] Nó B: versão=1; réplica={'produto': 'livro', 'estoque': 7}
[11:53:14] Nó D: versão=1; réplica={'produto': 'livro', 'estoque': 7}
[11:53:15] 2. Simulando falha do nó B...
[11:53:17] 3. Detecção por timeout: {'A': 'ATIVO', 'B': 'SEM CONTATO', 'C': 'ATIVO', 'D': 'ATIVO'} | dados={'produto': 'livro', 'estoque': 7} (v1)
[11:53:18] Professor: estoque alterado para 5 durante a falha
[11:53:18] 4. Reiniciando o nó B...
[11:53:18] Nó B: iniciado em PID=19744
[11:53:18] Nó B: versão=2; réplica={'produto': 'livro', 'estoque': 5}
[11:53:18] Nó D: versão=2; réplica={'produto': 'livro', 'estoque': 5}
[11:53:18] Nó A: versão=2; réplica={'produto': 'livro', 'estoque': 5}
[11:53:18] Nó C: versão=2; réplica={'produto': 'livro', 'estoque': 5}
[11:53:19] 5. Nó B recuperou a versão mais recente: {'A': 'ATIVO', 'B': 'ATIVO', 'C': 'ATIVO', 'D': 'ATIVO'} | dados={'produto': 'livro', 'estoque': 5} (v2)
[11:53:20] Demonstração encerrada.
```

## Como funciona

- **Coordenador:** servidor TCP (`socketserver.ThreadingTCPServer`) no processo principal, com uma thread por conexão. Guarda os dados do produto, um número de versão que sobe a cada `ATUALIZAR` e o horário do último contato de cada nó. Uma trava (`threading.Lock`) impede duas alterações ao mesmo tempo.
- **Protocolo:** cada pedido abre uma conexão, envia uma linha JSON terminada em `\n` e recebe uma linha JSON de resposta. Tipos: `REGISTRAR`, `PULSO`, `ATUALIZAR` e `STATUS`.
- **Nós:** processos separados (`multiprocessing`, modo `spawn`). Cada um guarda uma réplica e a versão que conhece, e envia um pulso a cada 0,55 s. O coordenador só manda a cópia completa dos dados quando a versão do nó é diferente da dele.
- **Detecção de falha:** no `STATUS`, um nó sem pulso há 2 s ou mais aparece como `SEM CONTATO`.
- **Roteiro:** a função `main` mostra o estado inicial, muda o estoque para 7, encerra o nó B, confirma a detecção, muda o estoque para 5 com B fora do ar e reinicia B, que volta direto na versão mais recente.

## Observação

Em algumas execuções pode aparecer no encerramento um `ConnectionResetError` impresso pelo servidor. Isso acontece quando um nó é finalizado no meio de um pulso, com a conexão aberta; o atendente só trata erros de formato da mensagem. O resultado da demonstração não muda, e esse trecho é igual ao do código original.

É um laboratório didático com coordenador central: não implementa consenso distribuído nem tolera a falha do próprio coordenador.
