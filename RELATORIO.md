# Relatório do Laboratório 2 - Chamadas de Sistema

---

## Exercício 1a - Observação printf() vs 1b - write()

### Comandos executados:

```bash
strace -e write ./ex1a_printf
strace -e write ./ex1b_write
```

### Análise

**1. Quantas syscalls write() cada programa gerou?**

- ex1a_printf: 8 syscalls
- ex1b_write: 7 syscalls

**2. Por que há diferença entre os dois métodos? Consulte o docs/printf_vs_write.md**

```
O printf() é uma função de biblioteca da linguagem C. Ele usa buffering, ou seja, acumula mensagens e envia ao kernel em blocos maiores, resultando em menos chamadas write(). Já o write() é uma syscall direta, cada chamada gera imediatamente uma transição para o kernel.
```

**3. Qual método é mais previsível? Por quê?**

```
O write() é mais previsível, pois cada chamada gera exatamente uma syscall e o comportamento não depende do buffering interno da biblioteca.
```

---

## Exercício 2 - Leitura de Arquivo

### Resultados da execução:

- File descriptor: 3
- Bytes lidos: 127

### Comando strace:

```bash
strace -e openat,read,close ./ex2_leitura
```

### Análise

**1. Qual file descriptor foi usado? Por que não 0, 1 ou 2?**

```
Foi usado o fd = 3, porque os descritores 0, 1 e 2 já estão reservados para stdin, stdout e stderr.
```

**2. Como você sabe que o arquivo foi lido completamente?**

```
Porque o read() retornou exatamente a quantidade de bytes que o arquivo tinha, e depois retornou 0 indicando EOF (fim do arquivo).
```

**3. O que acontece se esquecer de fechar o arquivo?**

```
O descritor de arquivo continua alocado, podendo causar fugas de recurso (muitos arquivos abertos até atingir o limite do sistema).
```

**4. Por que verificar retorno de cada syscall?**

```
Para garantir que a operação realmente funcionou (ex: open() pode falhar se o arquivo não existir, read() pode retornar erro de disco).
```

---

## Exercício 3 - Contador com Loop

### Resultados (BUFFER_SIZE = 64):

- Linhas: 25 (esperado: 25)
- Caracteres: 1300
- Chamadas read(): 21
- Tempo: 0.002090 segundos

### Experimentos com buffer:

| Buffer Size | Chamadas read() | Tempo (s) |
| ----------- | --------------- | --------- |
| 16          | 82              | 0.000456  |
| 64          | 21              | 0.002090  |
| 256         | 6               | 0.002065  |
| 1024        | 2               | 0.000270  |

### Análise

**1. Como o tamanho do buffer afeta o número de syscalls?**

```
Buffers maiores reduzem o número de chamadas read(), pois cada leitura traz mais dados.
```

**2. Como você detecta o fim do arquivo?**

```
Quando read() retorna 0, significa EOF.
```

**3. Todas as chamadas read() retornaram BUFFER_SIZE bytes?**

```
Não. Apenas as intermediárias retornam próximo do tamanho do buffer. A última geralmente retorna menos bytes, e no fim retorna 0.
```

**4. Qual é a relação entre syscalls e performance?**

```
Menos chamadas → menos transições usuário↔kernel → desempenho melhor.
```

---

## Exercício 4 - Cópia de Arquivo

### Resultados:

- Bytes copiados: 1364
- Operações: 6
- Tempo: 0.001741 segundos
- Throughput: 765.10 KB/s

### Verificação:

```bash
diff dados/origem.txt dados/destino.txt
```

Resultado: [x] Idênticos [ ] Diferentes

### Análise

**1. Por que devemos verificar que bytes_escritos == bytes_lidos?**

```
Para garantir que nenhum byte foi perdido na escrita.
```

**2. Que flags são essenciais no open() do destino?**

```
O_WRONLY | O_CREAT | O_TRUNC para permitir escrita, criar o arquivo se não existir e sobrescrever se já existir.
```

**3. O número de reads e writes é igual? Por quê?**

```
Sim, porque para cada bloco lido há uma escrita correspondente.
```

**4. Como você saberia se o disco ficou cheio?**

```
O write() retornaria um valor menor que o número de bytes pedidos ou erro ENOSPC.
```

**5. O que acontece se esquecer de fechar os arquivos?**

```
Os descritores continuam abertos, podendo causar vazamento de recursos e dados não serem efetivamente gravados.
```

---

## Análise Geral

### Conceitos Fundamentais

**1. Como as syscalls demonstram a transição usuário → kernel?**

```
Cada syscall (open, read, write, close) força o processo a sair do modo usuário e entrar no modo kernel, onde o SO realiza a operação.
```

**2. Qual é o seu entendimento sobre a importância dos file descriptors?**

```
Eles funcionam como "ponteiros" para recursos de E/S. Sem eles, o processo não teria como acessar arquivos, sockets ou terminais.
```

**3. Discorra sobre a relação entre o tamanho do buffer e performance:**

```
Buffers maiores diminuem o número de syscalls, aumentando a performance, mas usam mais memória. Buffers muito pequenos geram overhead.
```

### Comparação de Performance

```bash
# Teste seu programa vs cp do sistema
time ./ex4_copia
time cp dados/origem.txt dados/destino_cp.txt
```

**Qual foi mais rápido?** O ./ex4_copia foi mais rápido (0.011s vs 0.036s).

**Por que você acha que foi mais rápido?**

```
Porque ele fez apenas 6 operações, com um buffer relativamente eficiente (256 bytes). O cp pode usar verificações adicionais e overhead de sistema.
```

---

## Entrega

Certifique-se de ter:

- [x] Todos os códigos com TODOs completados
- [x] Traces salvos em `traces/`
- [x] Este relatório preenchido como `RELATORIO.md`

```bash
strace -e write -o traces/ex1a_trace.txt ./ex1a_printf
strace -e write -o traces/ex1b_trace.txt ./ex1b_write
strace -o traces/ex2_trace.txt ./ex2_leitura
strace -c -o traces/ex3_stats.txt ./ex3_contador
strace -o traces/ex4_trace.txt ./ex4_copia
```
