---
title: Não bloqueie o Event Loop (ou o Worker Pool)
layout: docs.hbs
---

# Não bloqueie o Event Loop (ou o Worker Pool)

## Você deve ler esse guia?
Se você está escrevendo algo mais complicado que um breve script de linha de comando, ler este guia ajudará você a escrever aplicativos de maior desempenho e mais seguros.

Este documento foi escrito com servidores Node em mente, mas os conceitos são aplicados para aplicações Node complexas também.
Onde detalhes específicos do sistema operacional variam, este documento é centrado no Linux.

## Resumo
O Node.js executa código JavaScript no Event Loop (inicialização e callbacks), e oferece um Worker Pool para manipular tarefas custusas como I/O de arquivo.
Node escala bem, as vezes mais do que abordagens pesadas como Apache.
O segredo da escalabilidade do Node é que ele usa um pequeno número de threads para manipular muitos clientes.
Se o Node pode trabalhar com menos threads, ele poderá gastar mais tempo do seu sistema e memória trabalhando nos clientes em vez de disperdiçar recursos de espaço e tempo para as threads (memória e mudança de contexto).
Mas pelo fato do Node ter poucas threads, você precisa estruturar sua aplicação para usá-las com sabadoria.  

Aqui está um princípio básico para manter o servidor Node rápido: *Node é rápido quando o trabalho associado a cada cliente em um determinado momento é "pequeno"*.

Isso se aplica a callbacks no Event Loop e tarefas no Worker Pool.

## Por que eu devo evitar bloquear o Event Loop e o Worker Pool?
O Node usa um pequeno número de threads para manipular muitos clientes.
No Node existem dois tipos de threads: um Event Loop (também conhecido como main loop, main thread, event thread, etc.), e uma pool de `k` Workers em um Worker Pool (também conhecido como threadpool)

Se uma thread está levando muito tempo para excutar um callback (Event Loop) ou uma tarefa (Worker), nós a chamamos de "bloqueada".
Enquanto uma thread está bloqueada trabalhando para um cliente, ela não pode lidar com requisições de outros clientes.  
Isso fornece duas motivações para não bloquear o Event Loop nem o Worker Pool:

1. Performance: Se você executar regularmente atividades pesadas em qualquer tipo de thread, o *throughput* (requisições por segundo) do seu servidor sofrerá.
2. Segurança: Se for possível que para determinadas entradas uma de suas threads possam bloquear, um cliente malicioso pode enviar esse "evil input", para fazer suas threads bloquearem, e mantê-las trabalhando para outros clientes. Isso seria um ataque de [Negação de Serviço](https://en.wikipedia.org/wiki/Denial-of-service_attack)

## Uma rápida revisão do Node

O Node usa a Arquitetura Orientada a Eventos: ele tem um Event Loop para orquestração e um Worker Pool para tarefas custosas.

### Que código é executado no Event Loop?
Quando elas começam, aplicações Node primeiro concluem uma fase de inicialização, fazendo "`require`'ing" de módulos e registrando callbacks para events.
As Aplicações Node entram no Event Loop, respondendo requisições recebidas do cliente para executar o callback apropriado.
Esse callback executa de forma síncrona, e pode registrar requisições assíncronas para continuar o processamento após a conclusão.

Os callbacks para essas requisições assíncronas também serão executadas no Event Loop.

O Event Loop também atenderá às requisições assíncronas não-bloqueantes feitas por seus callbacks, por exemplo, I/O de rede.

Em resumo, o Event Loop executa as callbacks JavaScript registradas por eventos, e também é responsável atender requisições assíncronas não-bloqueantes, como I/O de rede.

### Que código é executado no Worker Pool?
O Worker Pool do Node é implementado na libuv ([docs](http://docs.libuv.org/en/v1.x/threadpool.html)), que expõe uma API geral para envio de tarefas.

O Node usa o Worker Pool para lidar com tarefas "custosas".
Isso inclui I/O para quais um sistem operacional não fornece uma versão não-bloqueante, bem como tarefas particularmente intensivas em CPU.

Estas são os módulos de APIs do Node que fazem uso desse Worker Pool:

1. I/O intensivo
    1. [DNS](https://nodejs.org/api/dns.html): `dns.lookup()`, `dns.lookupService()`.
    2. [Sistema de arquivo](https://nodejs.org/api/fs.html#fs_threadpool_usage): Todas APIs do sistema de arquivo exceto `fs.FSWatcher()` e aquelas que são explicitamente síncronas usam a threadpool da libuv.
2. CPU intensivo
    1. [Crypto](https://nodejs.org/api/crypto.html): `crypto.pbkdf2()`, `crypto.scrypt()`, `crypto.randomBytes()`, `crypto.randomFill()`, `crypto.generateKeyPair()`.
    2. [Zlib](https://nodejs.org/api/zlib.html#zlib_threadpool_usage): Todas APIs do zlib exceto aquelas que são explicitamente síncronas usam a threadpool da libuv.

Em muitas aplicações Node, essas APIs são as únicas fontes de tarefas para o Worker Pool. Aplicações e módulos que usam um [C++ add-on](https://nodejs.org/api/addons.html) podem enviar tarefas para o Worker Pool.

Para cobrir todos os aspectos, observamos que quando você chama uma dessas APIs a partir de um callback no Event Loop, o Event Loop paga alguns custos menores de configuração, pois entra nas ligações do Node C++ para essa API e envia uma tarefa para ao Worker Pool.
Esses custos são insignificantes em comparação ao custo total da tarefa, e é por isso que o Event Loop está sendo menos usado.
Ao enviar uma dessas tarefas para o Worker Pool, o Node fornece um ponteiro para a função C++ correspondente nas ligações Node C++.

### Como o Node decide qual código executar a seguir?
De forma abstrata, o Event Loop e o Worker Pool mantêm filas para eventos e tarefas pendentes, respectivamente.

Na verdade, o Event Loop não mantém realmente uma fila.
Em vez disso, ele possui uma coleção de descritores de arquivos que solicita ao sistema operacional para monitorar, usando um mecanismo como [epoll](http://man7.org/linux/man-pages/man7/epoll.7.html) (Linux), [kqueue](https://developer.apple.com/library/content/documentation/Darwin/Conceptual/FSEvents_ProgGuide/KernelQueues/KernelQueues.html) (OSX), event ports (Solaris), ou [IOCP](https://msdn.microsoft.com/en-us/library/windows/desktop/aa365198.aspx) (Windows).
Esses descritores de arquivos correspondem aos sockets de rede, aos arquivos que estão sendo monitorados e assim por diante.
Quando o sistema operacional diz que um desses descritores de arquivos está pronto, o Evente Loop o converte para o evento apropriado e chama os callbacks associados com esse evento.
Você pode aprender mais sobre esse processo [aqui](https://www.youtube.com/watch?v=P9csgxBgaZ8).

Por outro lado, o Worker Pool usa uma fila real cujas entradas são tarefas a serem processadas.
Um Worker abre uma tarefa nesse fila e trabalha nela, e quando concluída, o Worker gera um evento "Pelo menos uma tarefa está concluída" para o Event Loop.

### O que isso significa para o design da aplicação?
Em um sistema uma-thread-por-cliente tipo Apache, cada cliente pendente recebe sua própria thread.
Se uma thread que manipula um cliente bloqueado, o sistema operacional irá interropé-lo e dará a vez para outro cliente.
O sistema operacional garante, assim, que os clientes que exigem uma pequena quantidade de trabalho não sejam prejudicados por clientes que exigem mais trabalho.

Como o Node lida com muitos clientes com poucas threads, se uma thread bloqueia o processamento da requisição de um cliente, as requisições pendentes do cliente podem não ter uma volta até que a thread conclua seu callback ou tarefa.
*O tratamento justo dos clientes é, portanto, de responsabilidade de sua applicação*.
Isso significa que você não deve fazer muito trabalho para nenhum cliente em uma única tarefa ou callback.

This is part of why Node can scale well, but it also means that you are responsible for ensuring fair scheduling.
Isso faz parte do motivo pelo qual
The next sections talk about how to ensure fair scheduling for the Event Loop and for the Worker Pool.

## Não bloqueia o Event Loop
O Event Loop percebe cada nova conexão do cliente e orquestra a geração de uma resposta.
Todas as solicitações recebidas e respostas enviadas passam pelo Event Loop.
Isso significa que, se o Event Loop passar muito tempo em algum ponto, todos os clientes atuais e novos não serão atendidos.

Você nunca deve bloquear o Event Loop.
Em outras palavras, cada um de seus callbacks JavaScript devem ser concluídos rapidamente.
Isto, obviamente, também se aplica aos seus `wait`'s , seus` Promise.then`'s, e assim por diante.

Uma boa maneira de garantir isso é estudar sobre a ["complexidade computacional"] (https://en.wikipedia.org/wiki/Time_complexity) de seus callbacks.
Se o seu callback executar um número constante de etapas, independentemente de seus argumentos, você sempre dará a cada cliente pendente uma chance justa.
Se seu callback executa um número considerável de etapas, dependendo de seus argumentos, pense em quanto tempo os argumentos podem demorar.

Exemplo 1: Um callback em tempo constante.

```javascript
app.get('/constant-time', (req, res) => {
  res.sendStatus(200);
});
```

Exemplo 2: Um callback `O(n)`. Este callback será executado rapidamente para pequenos `n` e mais lentamente para grandes `n`.

```javascript
app.get('/countToN', (req, res) => {
  let n = req.query.n;

  // n iterations before giving someone else a turn
  for (let i = 0; i < n; i++) {
    console.log(`Iter {$i}`);
  }

  res.sendStatus(200);
});
```

Exemplo 3: Um callback `O(n^2)`. Este callback ainda será executado rapidamente para pequenos `n`, mas para grandes `n`, será executado muito mais lentamente que o exemplo anterior `O(n)`.

```javascript
app.get('/countToN2', (req, res) => {
  let n = req.query.n;

  // n^2 iterations before giving someone else a turn
  for (let i = 0; i < n; i++) {
    for (let j = 0; j < n; j++) {
      console.log(`Iter ${i}.${j}`);
    }
  }

  res.sendStatus(200);
});
```

### Quão cuidadoso você deve ser?
O Node usa a engine V8 do Google para JavaScript, o que é bastante rápido para muitas operações comuns.
Exceções a esta regra são regexps e operações JSON, discutidas abaixo.

No entanto, para tarefas complexas, considere limitar a entrada e rejeitar entradas muito longas.
Dessa forma, mesmo que seu callback tenha grande complexidade, limitando a entrada, você garante que o callback não pode demorar mais do que o pior caso na entrada aceitável mais longa.
Você pode avaliar o pior caso desse callback e determinar se o tempo de execução é aceitável no seu contexto.

### Bloqueando o Event Loop: REDOS
Uma maneira comum de bloquear desastrosamente o Event Loop é usar uma [expressão regular](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Regular_Expressions) "vulnerável".

#### Evitando expressões regulares vulneráveis
Uma expressão regular (regexp) corresponde a uma sequência de entrada diante de um padrão.
Geralmente pensamos em uma combinação regexp exigindo uma única passagem pela string de entrada --- tempo `O(n)` em que `n` é o comprimento da string de entrada.
Em muitos casos, basta uma única passegem.
Infelizmente, em alguns casos, a correspondência regexp pode exigir um número exponencial de viagens pela string de entrada ---  tempo `O(2^n)`.
Um número exponencial de viagens significa que, se o mecanismo exigir `x` viagens para determinar uma correspondência, serão necessárias `2*x` viagens se adicionarmos apenas mais um caractere à string de entrada.
Como o número de viagens está linearmente relacionado ao tempo necessário, o efeito dessa avaliação será bloquear o Event Loop.

Uma *expressão regular vulnerável* é aquela em que seu mecanismo de expressão regular pode levar um tempo exponencial, expondo você a [REDOS](https://www.owasp.org/index.php/Regular_expression_Denial_of_Service_-_ReDoS) no "evil input".
Se o seu padrão de expressão regular é vulnerável (ou seja, o mecanismo regexp pode levar um tempo exponencial) é realmente uma pergunta difícil de responder e varia dependendo de você estar usando Perl, Python, Ruby, Java, JavaScript, etc., mas aqui estão algumas regras práticas que se aplicam a todas essas linguagens:

1. Evite quantificadores aninhados como `(a+)*`. O mecanismo regexp do Node pode lidar com alguns deles rapidamente, mas outros são vulneráveis.
2. Evite OR's com cláusulas sobrepostas, como `(a|a)*`. Novamente, esses nem sempre são rápidos.
3. Evite usar referências anteriores, como `(a.*) \1`. Nenhum mecanismo regexp pode garantir a avaliação em tempo linear.
4. Se você estiver fazendo uma correspondência simples de string, use `indexOf` ou o equivalente local. Será mais barato e nunca levará mais que `O(n)`.

Se você não tiver certeza se sua expressão regular é vulnerável, lembre-se de que o Node geralmente não tem problemas para relatar uma *correspondência*, mesmo para uma regexp vulnerável e uma longa string de entrada.
O comportamento exponencial é acionado quando há uma incompatibilidade, mas o Node não pode ter certeza até que tente muitos caminhos pela string de entrada.

#### Um exemplo de REDOS
Aqui está um exemplo de regexp vulnerável, expondo seu servidor ao REDOS:

```javascript
app.get('/redos-me', (req, res) => {
  let filePath = req.query.filePath;

  // REDOS
  if (fileName.match(/(\/.+)+$/)) {
    console.log('valid path');
  }
  else {
    console.log('invalid path');
  }

  res.sendStatus(200);
});
```

The vulnerable regexp in this example is a (bad!) way to check for a valid path on Linux.
It matches strings that are a sequence of "/"-delimited names, like "/a/b/c".
It is dangerous because it violates rule 1: it has a doubly-nested quantifier.

If a client queries with filePath `///.../\n` (100 /'s followed by a newline character that the regexp's "." won't match), then the Event Loop will take effectively forever, blocking the Event Loop.
This client's REDOS attack causes all other clients not to get a turn until the regexp match finishes.

For this reason, you should be leery of using complex regular expressions to validate user input.

#### Anti-REDOS Resources
There are some tools to check your regexps for safety, like
- [safe-regex](https://github.com/substack/safe-regex)
- [rxxr2](http://www.cs.bham.ac.uk/~hxt/research/rxxr2/).
However, neither of these will catch all vulnerable regexps.

Another approach is to use a different regexp engine.
You could use the [node-re2](https://github.com/uhop/node-re2) module, which uses Google's blazing-fast [RE2](https://github.com/google/re2) regexp engine.
But be warned, RE2 is not 100% compatible with Node's regexps, so check for regressions if you swap in the node-re2 module to handle your regexps.
And particularly complicated regexps are not supported by node-re2.

If you're trying to match something "obvious", like a URL or a file path, find an example in a [regexp library](http://www.regexlib.com) or use an npm module, e.g. [ip-regex](https://www.npmjs.com/package/ip-regex).

### Blocking the Event Loop: Node core modules
Several Node core modules have synchronous expensive APIs, including:
- [Encryption](https://nodejs.org/api/crypto.html)
- [Compression](https://nodejs.org/api/zlib.html)
- [File system](https://nodejs.org/api/fs.html)
- [Child process](https://nodejs.org/api/child_process.html)

These APIs are expensive, because they involve significant computation (encryption, compression), require I/O (file I/O), or potentially both (child process). These APIs are intended for scripting convenience, but are not intended for use in the server context. If you execute them on the Event Loop, they will take far longer to complete than a typical JavaScript instruction, blocking the Event Loop.

In a server, *you should not use the following synchronous APIs from these modules*:
- Encryption:
  - `crypto.randomBytes` (synchronous version)
  - `crypto.randomFillSync`
  - `crypto.pbkdf2Sync`
  - You should also be careful about providing large input to the encryption and decryption routines.
- Compression:
  - `zlib.inflateSync`
  - `zlib.deflateSync`
- File system:
  - Do not use the synchronous file system APIs. For example, if the file you access is in a [distributed file system](https://en.wikipedia.org/wiki/Clustered_file_system#Distributed_file_systems) like [NFS](https://en.wikipedia.org/wiki/Network_File_System), access times can vary widely.
- Child process:
  - `child_process.spawnSync`
  - `child_process.execSync`
  - `child_process.execFileSync`

This list is reasonably complete as of Node v9.

### Blocking the Event Loop: JSON DOS
`JSON.parse` and `JSON.stringify` are other potentially expensive operations.
While these are `O(n)` in the length of the input, for large `n` they can take surprisingly long.

If your server manipulates JSON objects, particularly those from a client, you should be cautious about the size of the objects or strings you work with on the Event Loop.

Example: JSON blocking. We create an object `obj` of size 2^21 and `JSON.stringify` it, run `indexOf` on the string, and then JSON.parse it. The `JSON.stringify`'d string is 50MB. It takes 0.7 seconds to stringify the object, 0.03 seconds to indexOf on the 50MB string, and 1.3 seconds to parse the string.

```javascript
var obj = { a: 1 };
var niter = 20;

var before, str, pos, res, took;

for (var i = 0; i < niter; i++) {
  obj = { obj1: obj, obj2: obj }; // Doubles in size each iter
}

before = process.hrtime();
str = JSON.stringify(obj);
took = process.hrtime(before);
console.log('JSON.stringify took ' + took);

before = process.hrtime();
pos = str.indexOf('nomatch');
took = process.hrtime(before);
console.log('Pure indexof took ' + took);

before = process.hrtime();
res = JSON.parse(str);
took = process.hrtime(before);
console.log('JSON.parse took ' + took);
```

There are npm modules that offer asynchronous JSON APIs. See for example:
- [JSONStream](https://www.npmjs.com/package/JSONStream), which has stream APIs.
- [Big-Friendly JSON](https://www.npmjs.com/package/bfj), which has stream APIs as well as asynchronous versions of the standard JSON APIs using the partitioning-on-the-Event-Loop paradigm outlined below.

### Complex calculations without blocking the Event Loop
Suppose you want to do complex calculations in JavaScript without blocking the Event Loop.
You have two options: partitioning or offloading.

#### Partitioning
You could *partition* your calculations so that each runs on the Event Loop but regularly yields (gives turns to) other pending events.
In JavaScript it's easy to save the state of an ongoing task in a closure, as shown in example 2 below.

For a simple example, suppose you want to compute the average of the numbers `1` to `n`.

Example 1: Un-partitioned average, costs `O(n)`

```javascript
for (let i = 0; i < n; i++)
  sum += i;
let avg = sum / n;
console.log('avg: ' + avg);
```

Example 2: Partitioned average, each of the `n` asynchronous steps costs `O(1)`.

```javascript
function asyncAvg(n, avgCB) {
  // Save ongoing sum in JS closure.
  var sum = 0;
  function help(i, cb) {
    sum += i;
    if (i == n) {
      cb(sum);
      return;
    }

    // "Asynchronous recursion".
    // Schedule next operation asynchronously.
    setImmediate(help.bind(null, i+1, cb));
  }

  // Start the helper, with CB to call avgCB.
  help(1, function(sum){
      var avg = sum/n;
      avgCB(avg);
  });
}

asyncAvg(n, function(avg){
  console.log('avg of 1-n: ' + avg);
});
```

You can apply this principle to array iterations and so forth.

#### Offloading
If you need to do something more complex, partitioning is not a good option.
This is because partitioning uses only the Event Loop, and you won't benefit from multiple cores almost certainly available on your machine.
*Remember, the Event Loop should orchestrate client requests, not fulfill them itself.*
For a complicated task, move the work off of the Event Loop onto a Worker Pool.

##### How to offload
You have two options for a destination Worker Pool to which to offload work.
1. You can use the built-in Node Worker Pool by developing a [C++ addon](https://nodejs.org/api/addons.html). On older versions of Node, build your C++ addon using [NAN](https://github.com/nodejs/nan), and on newer versions use [N-API](https://nodejs.org/api/n-api.html). [node-webworker-threads](https://www.npmjs.com/package/webworker-threads) offers a JavaScript-only way to access Node's Worker Pool.
2. You can create and manage your own Worker Pool dedicated to computation rather than Node's I/O-themed Worker Pool. The most straightforward ways to do this is using [Child Process](https://nodejs.org/api/child_process.html) or [Cluster](https://nodejs.org/api/cluster.html).

You should *not* simply create a [Child Process](https://nodejs.org/api/child_process.html) for every client.
You can receive client requests more quickly than you can create and manage children, and your server might become a [fork bomb](https://en.wikipedia.org/wiki/Fork_bomb).

##### Downside of offloading
The downside of the offloading approach is that it incurs overhead in the form of *communication costs*.
Only the Event Loop is allowed to see the "namespace" (JavaScript state) of your application.
From a Worker, you cannot manipulate a JavaScript object in the Event Loop's namespace.
Instead, you have to serialize and deserialize any objects you wish to share.
Then the Worker can operate on its own copy of these object(s) and return the modified object (or a "patch") to the Event Loop.

For serialization concerns, see the section on JSON DOS.

##### Some suggestions for offloading
You may wish to distinguish between CPU-intensive and I/O-intensive tasks because they have markedly different characteristics.

A CPU-intensive task only makes progress when its Worker is scheduled, and the Worker must be scheduled onto one of your machine's [logical cores](https://nodejs.org/api/os.html#os_os_cpus).
If you have 4 logical cores and 5 Workers, one of these Workers cannot make progress.
As a result, you are paying overhead (memory and scheduling costs) for this Worker and getting no return for it.

I/O-intensive tasks involve querying an external service provider (DNS, file system, etc.) and waiting for its response.
While a Worker with an I/O-intensive task is waiting for its response, it has nothing else to do and can be de-scheduled by the operating system, giving another Worker a chance to submit their request.
Thus, *I/O-intensive tasks will be making progress even while the associated thread is not running*.
External service providers like databases and file systems have been highly optimized to handle many pending requests concurrently.
For example, a file system will examine a large set of pending write and read requests to merge conflicting updates and to retrieve files in an optimal order (e.g. see [these slides](http://researcher.ibm.com/researcher/files/il-AVISHAY/01-block_io-v1.3.pdf)).

If you rely on only one Worker Pool, e.g. the Node Worker Pool, then the differing characteristics of CPU-bound and I/O-bound work may harm your application's performance.

For this reason, you might wish to maintain a separate Computation Worker Pool.

#### Offloading: conclusions
For simple tasks, like iterating over the elements of an arbitrarily long array, partitioning might be a good option.
If your computation is more complex, offloading is a better approach: the communication costs, i.e. the overhead of passing serialized objects between the Event Loop and the Worker Pool, are offset by the benefit of using multiple cores.

However, if your server relies heavily on complex calculations, you should think about whether Node is really a good fit. Node excels for I/O-bound work, but for expensive computation it might not be the best option.

If you take the offloading approach, see the section on not blocking the Worker Pool.

## Don't block the Worker Pool
Node has a Worker Pool composed of `k` Workers.
If you are using the Offloading paradigm discussed above, you might have a separate Computational Worker Pool, to which the same principles apply.
In either case, let us assume that `k` is much smaller than the number of clients you might be handling concurrently.
This is in keeping with Node's "one thread for many clients" philosophy, the secret to its scalability.

As discussed above, each Worker completes its current Task before proceeding to the next one on the Worker Pool queue.

Now, there will be variation in the cost of the Tasks required to handle your clients' requests.
Some Tasks can be completed quickly (e.g. reading short or cached files, or producing a small number of random bytes), and others will take longer (e.g reading larger or uncached files, or generating more random bytes).
Your goal should be to *minimize the variation in Task times*, and you should use *Task partitioning* to accomplish this.

### Minimizing the variation in Task times
If a Worker's current Task is much more expensive than other Tasks, then it will be unavailable to work on other pending Tasks.
In other words, *each relatively long Task effectively decreases the size of the Worker Pool by one until it is completed*.
This is undesirable because, up to a point, the more Workers in the Worker Pool, the greater the Worker Pool throughput (tasks/second) and thus the greater the server throughput (client requests/second).
One client with a relatively expensive Task will decrease the throughput of the Worker Pool, in turn decreasing the throughput of the server.

To avoid this, you should try to minimize variation in the length of Tasks you submit to the Worker Pool.
While it is appropriate to treat the external systems accessed by your I/O requests (DB, FS, etc.) as black boxes, you should be aware of the relative cost of these I/O requests, and should avoid submitting requests you can expect to be particularly long.

Two examples should illustrate the possible variation in task times.

#### Variation example: Long-running file system reads
Suppose your server must read files in order to handle some client requests.
After consulting Node's [File system](https://nodejs.org/api/fs.html) APIs, you opted to use `fs.readFile()` for simplicity.
However, `fs.readFile()` is ([currently](https://github.com/nodejs/node/pull/17054)) not partitioned: it submits a single `fs.read()` Task spanning the entire file.
If you read shorter files for some users and longer files for others, `fs.readFile()` may introduce significant variation in Task lengths, to the detriment of Worker Pool throughput.

For a worst-case scenario, suppose an attacker can convince your server to read an *arbitrary* file (this is a [directory traversal vulnerability](https://www.owasp.org/index.php/Path_Traversal)).
If your server is running Linux, the attacker can name an extremely slow file: [`/dev/random`](http://man7.org/linux/man-pages/man4/random.4.html).
For all practical purposes, `/dev/random` is infinitely slow, and every Worker asked to read from `/dev/random` will never finish that Task.
An attacker then submits `k` requests, one for each Worker, and no other client requests that use the Worker Pool will make progress.

#### Variation example: Long-running crypto operations
Suppose your server generates cryptographically secure random bytes using [`crypto.randomBytes()`](https://nodejs.org/api/crypto.html#crypto_crypto_randombytes_size_callback).
`crypto.randomBytes()` is not partitioned: it creates a single `randomBytes()` Task to generate as many bytes as you requested.
If you create fewer bytes for some users and more bytes for others, `crypto.randomBytes()` is another source of variation in Task lengths.

### Task partitioning
Tasks with variable time costs can harm the throughput of the Worker Pool.
To minimize variation in Task times, as far as possible you should *partition* each Task into comparable-cost sub-Tasks.
When each sub-Task completes it should submit the next sub-Task, and when the final sub-Task completes it should notify the submitter.

To continue the `fs.readFile()` example, you should instead use `fs.read()` (manual partitioning) or `ReadStream` (automatically partitioned).

The same principle applies to CPU-bound tasks; the `asyncAvg` example might be inappropriate for the Event Loop, but it is well suited to the Worker Pool.

When you partition a Task into sub-Tasks, shorter Tasks expand into a small number of sub-Tasks, and longer Tasks expand into a larger number of sub-Tasks.
Between each sub-Task of a longer Task, the Worker to which it was assigned can work on a sub-Task from another, shorter, Task, thus improving the overall Task throughput of the Worker Pool.

Note that the number of sub-Tasks completed is not a useful metric for the throughput of the Worker Pool.
Instead, concern yourself with the number of *Tasks* completed.

### Avoiding Task partitioning
Recall that the purpose of Task partitioning is to minimize the variation in Task times.
If you can distinguish between shorter Tasks and longer Tasks (e.g. summing an array vs. sorting an array), you could create one Worker Pool for each class of Task.
Routing shorter Tasks and longer Tasks to separate Worker Pools is another way to minimize Task time variation.

In favor of this approach, partitioning Tasks incurs overhead (the costs of creating a Worker Pool Task representation and of manipulating the Worker Pool queue), and avoiding partitioning saves you the costs of additional trips to the Worker Pool.
It also keeps you from making mistakes in partitioning your Tasks.

The downside of this approach is that Workers in all of these Worker Pools will incur space and time overheads and will compete with each other for CPU time.
Remember that each CPU-bound Task makes progress only while it is scheduled.
As a result, you should only consider this approach after careful analysis.

### Worker Pool: conclusions
Whether you use only the Node Worker Pool or maintain separate Worker Pool(s), you should optimize the Task throughput of your Pool(s).

To do this, minimize the variation in Task times by using Task partitioning.

## The risks of npm modules
While the Node core modules offer building blocks for a wide variety of applications, sometimes something more is needed. Node developers benefit tremendously from the [npm ecosystem](https://www.npmjs.com/), with hundreds of thousands of modules offering functionality to accelerate your development process.

Remember, however, that the majority of these modules are written by third-party developers and are generally released with only best-effort guarantees. A developer using an npm module should be concerned about two things, though the latter is frequently forgotten.
1. Does it honor its APIs?
2. Might its APIs block the Event Loop or a Worker?
Many modules make no effort to indicate the cost of their APIs, to the detriment of the community.

For simple APIs you can estimate the cost of the APIs; the cost of string manipulation isn't hard to fathom.
But in many cases it's unclear how much an API might cost.

*If you are calling an API that might do something expensive, double-check the cost. Ask the developers to document it, or examine the source code yourself (and submit a PR documenting the cost).*

Remember, even if the API is asynchronous, you don't know how much time it might spend on a Worker or on the Event Loop in each of its partitions.
For example, suppose in the `asyncAvg` example given above, each call to the helper function summed *half* of the numbers rather than one of them.
Then this function would still be asynchronous, but the cost of each partition would be `O(n)`, not `O(1)`, making it much less safe to use for arbitrary values of `n`.

## Conclusion
Node has two types of threads: one Event Loop and `k` Workers.
The Event Loop is responsible for JavaScript callbacks and non-blocking I/O, and a Worker executes tasks corresponding to C++ code that completes an asynchronous request, including blocking I/O and CPU-intensive work.
Both types of threads work on no more than one activity at a time.
If any callback or task takes a long time, the thread running it becomes *blocked*.
If your application makes blocking callbacks or tasks, this can lead to degraded throughput (clients/second) at best, and complete denial of service at worst.

To write a high-throughput, more DoS-proof web server, you must ensure that on benign and on malicious input, neither your Event Loop nor your Workers will block.
