# Estruturas Binárias

Quando começamos a desenvolver projetos, dependendo de como as coisas são feitas, pra a economia de memória e talvez(se bem implementado), performance, uma das práticas é fazer as estruturas sempre pendendo ao binário. Um exemplo que é relativamente bem conhecido no backend, é o GRPC, que faz uso de Protocol Buffers pra envio de conteúdo. A diferença desse protocol buffer pra um JSON convencional é que, enquanto JSON é textual e precisa de parsing e tem um tamanho de payload razoavelmentr grande, um conteúdo enviado via Protocol buffer é serializado muito mais rápido pelo fato de não ser textual, e por isso, ser mais rápido e fácil de serializar e o conteúdo do payload então acaba sendo menor. 

## Números como Flags

Um BitVector pode ser usado como um sistema de flags.
Vamos supor que um usuário precise fazer uma determinada operação, mas somente se tiver uma permissão em específico. 
Isso pode ser feito com uma struct que se parece com o seguinte:
```
struct Flags {
  read: bool,
  write: bool,
  delete: bool,
  modify: bool,
}
```
Como na memória, 1 boolean = 1 byte, aqui estamos usando um total de 4bytes de memória pra poder representar no máximo 16 possibilidades. 
Como um número é somente 0 ou 1, ou seja, true ou false, a gente pode representar essas flags com um número. O tamanho mínimo de memória que pode ser alocado é 1 byte, ou seja 8bits. Então podemos usar esses 8 bits pra representar esses 16 estados.
Você precisa entender que nesse caso, cada flag, ou boolean, é 1 bit, então desses 8bits que vamos ter, apenas 4 serão usados.
Podemos fazer isso da seguinte maneira:
```
enum FlagsEnum {
  Read = 1 << 0,
  Write = 1 << 1,
  Delete = 1 << 2,
  Modify = 1 << 3
}
struct Flags(u8){
  static from_flags(flags:FlagsEnum) {
    return this(flags as u8);
  }
}
```
A ideia nisso é simples. Um número binário de 8 bits é representado como 0b0000xxxx, no caso, onde está "x" são os bits que vão ser usados(por ser binário, são 1 ou 0). 
### Left Shift
O operador de left shift(<<) pega um número N, e joga os bits dele M casas pro lado esquerdo. Em `FlagsEnum`, temos `Read = 1 << 0`, isso significa que o valor resultante tem os mesmos bits de "1" mas com eles jogados a 0 bits a esquerda, ou seja, 1.
Seguindo a lógica, temos que
`Read = 0b1`
`Write = 0b10`
`Delete = 0b100`
`Modify = 0b1000`
Então pra verificar se um desses números tem uma flag, só verificar se o número tem um desses bits como 1. 
### OR e AND
Os operadores OR e AND, que são usados em linguagens de programação no geral como `|` e `&` respectivamente, servem aqui fazer operações que são booleanas, mas a nivel de bit. 
Por exemplo a regra de que `false || true == true` também se aplica, mas nesse caso a todos os 8 bits, então `0b0 | 0b1 == 0b1`. Pra então adicionar uma flag, o código ficaria algo como
```
enum FlagsEnum {
  Read = 1 << 0,
  Write = 1 << 1,
  Delete = 1 << 2,
  Modify = 1 << 3
}
struct Flags(u8){
  static from_flags(flags:FlagsEnum):This {
    return this(flags as u8);
  }
  with_flag(flag:FlagsEnum){
    this.0 = this.0 | flag;
  }
}
```
A lógica é simples, se as flags são, 0b1001 e eu executo `Flags.with_flag(FlagsEnum.Write)`, é executado então `0b1001 | 0b0010` que vira `0b1011`

O AND serve pra verificar se os bits daquele número estão como 1 ou 0, invés de adicionar, ele serve pra filtrar. Isso é feito da seguinte maneira:
```
  struct Flags(u8){
    has_flag(flag:FlagsEnum):bool {
      return this.0 & flag == flag;
    }
  }
```

Supondo que a flag salva seja `0b1011` e a que verificamos é `0b0100`, pelo AND seguir a mesma lógica de um `&&`, ele verifica se ambos são 1, se forem, retorna 1, se não, 0. Logo `0b1011 & 0b0100` retorna `0b0000`, logo, é false. Caso fosse `0b1011 & 0b1000` retornaria `0b1000`, então a struct contém a flag `0b0100`.
# Estruturas Binárias

* Nota previa: Nenhum código aqui está sendo feito com uma sintaxe de uma linguagem real. O intuito disso é pra evitar de o conhecimento se atrelar à uma linguagem em específico

Quando começamos a desenvolver projetos, dependendo de como as coisas são feitas, pra a economia de memória e talvez(se bem implementado), performance, uma das práticas é fazer as estruturas sempre pendendo ao binário. Um exemplo que é relativamente bem conhecido no backend, é o GRPC, que faz uso de Protocol Buffers pra envio de conteúdo. A diferença desse protocol buffer pra um JSON convencional é que, enquanto JSON é textual e precisa de parsing e tem um tamanho de payload razoavelmentr grande, um conteúdo enviado via Protocol buffer é serializado muito mais rápido pelo fato de não ser textual, e por isso, ser mais rápido e fácil de serializar e o conteúdo do payload então acaba sendo menor. 

## Números como Flags

Um BitVector pode ser usado como um sistema de flags.
Vamos supor que um usuário precise fazer uma determinada operação, mas somente se tiver uma permissão em específico. 
Isso pode ser feito com uma struct que se parece com o seguinte:
```
struct Flags {
  read: bool,
  write: bool,
  delete: bool,
  modify: bool,
}
```
Como na memória, 1 boolean = 1 byte, aqui estamos usando um total de 4bytes de memória pra poder representar no máximo 16 possibilidades. 
Como um número é somente 0 ou 1, ou seja, true ou false, a gente pode representar essas flags com um número. O tamanho mínimo de memória que pode ser alocado é 1 byte, ou seja 8bits. Então podemos usar esses 8 bits pra representar esses 16 estados.
Você precisa entender que nesse caso, cada flag, ou boolean, é 1 bit, então desses 8bits que vamos ter, apenas 4 serão usados.
Podemos fazer isso da seguinte maneira:
```
enum FlagsEnum {
  Read = 1 << 0,
  Write = 1 << 1,
  Delete = 1 << 2,
  Modify = 1 << 3
}
struct Flags(u8){
  static from_flags(flags:FlagsEnum) {
    return this(flags as u8);
  }
}
```
A ideia nisso é simples. Um número binário de 8 bits é representado como 0b0000xxxx, no caso, onde está "x" são os bits que vão ser usados(por ser binário, são 1 ou 0). 
### Left Shift
O operador de left shift(<<) pega um número N, e joga os bits dele M casas pro lado esquerdo. Em `FlagsEnum`, temos `Read = 1 << 0`, isso significa que o valor resultante tem os mesmos bits de "1" mas com eles jogados a 0 bits a esquerda, ou seja, 1.
Seguindo a lógica, temos que
`Read = 0b1`
`Write = 0b10`
`Delete = 0b100`
`Modify = 0b1000`
Então pra verificar se um desses números tem uma flag, só verificar se o número tem um desses bits como 1. 
### OR e AND
Os operadores OR e AND, que são usados em linguagens de programação no geral como `|` e `&` respectivamente, servem aqui fazer operações que são booleanas, mas a nivel de bit. 
Por exemplo a regra de que `false || true == true` também se aplica, mas nesse caso a todos os 8 bits, então `0b0 | 0b1 == 0b1`. Pra então adicionar uma flag, o código ficaria algo como
```
enum FlagsEnum {
  Read = 1 << 0,
  Write = 1 << 1,
  Delete = 1 << 2,
  Modify = 1 << 3
}
struct Flags(u8){
  static from_flags(flags:FlagsEnum):This {
    return this(flags as u8);
  }
  with_flag(flag:FlagsEnum){
    this.0 = this.0 | flag;
  }
}
```
A lógica é simples, se as flags são, 0b1001 e eu executo `Flags.with_flag(FlagsEnum.Write)`, é executado então `0b1001 | 0b0010` que vira `0b1011`

O AND serve pra verificar se os bits daquele número estão como 1 ou 0, invés de adicionar, ele serve pra filtrar. Isso é feito da seguinte maneira:
```
  struct Flags(u8){
    has_flag(flag:FlagsEnum):bool {
      return this.0 & flag == flag;
    }
  }
```

Supondo que a flag salva seja `0b1011` e a que verificamos é `0b0100`, pelo AND seguir a mesma lógica de um `&&`, ele verifica se ambos são 1, se forem, retorna 1, se não, 0. Logo `0b1011 & 0b0100` retorna `0b0000`, logo, é false. Caso fosse `0b1011 & 0b1000` retornaria `0b1000`, então a struct contém a flag `0b0100`.

## 'Compactação'
Em determinadas ocasiões em que por exemplo, é necessário um ID único, a gente tende a usar números(obviamente se isso não gerar algum problema estrutural). Vamos supor a seguinte ideia:
```
class NameInternalizer<T> {
  names: Hashmap<String, T>;

  NameInternalizer(){
    this.names = new Hashmap(); 
  }
  intern(name: String, data:T) {
    this.names.insert(name, data);
  }
  has(name:String) -> bool {
    return this.names.contains(name);
  }
  get(name: String) -> Optional<T>{
    return this.names.get(name);
  }
}
```

Básicamente um wrapper. Supondo que a gente sabe que o tamanho da string não precisa ser muito grande, algo como `assert(name.len() <= MAX_LENGTH)` então deveria sempre passar. Uma forma de otimizar esse internalizador pode ser usando arrays e 'ids'.
Explicando onde esse NameInternalizer pode acabar pecando, a gente começa com um HashMap, que não é uma estrutura ruim, mas pra toda string adicionada, uma cópia dela deve ser feita, o que pode acabar aumentando o uso de memória. Uma das formas então é fazer com que a gente coloque o conteúdo da string todo em um array só, e retorne um ID. O ID pode ser interpretado como
```
class InternID {
  offset: int,
  length: int,
}```

Que no caso o 'offset' é onde a String se iniciar e o 'length' o tamanho. O código então ficaria algo como?

```
class NameInternalizer<T> {
  buffer: String,
  names: Hashmap<StringSlice, T>;

  NameInternalizer(){
    this.names = new Hashmap(); 
    this.buffer = new String();
  }
  intern(name: String, data:T) -> InternID {
    let offset = this.buffer.len();
    let len = name.len();
    buffer.append(name);
    names.insert(&buffer[offset..offset+len]);
    return new InternID(offset,len);
  }
}
```

Supondo que o tipo `int` é o equivalente ao `int`de C, ou seja, um signed int de 32bits, a gente tá fazendo com que, implicitamente, MAX_LENGTH seja 2Bilhões, o que eu duvido bastante que ocorra. Então pra usar o minimo de memória possível, a gente precisa saber qual a quantia minima de bits pra poder representar números de 0..MAX_LENGTH.
Pra isso, eu pessoalmente acho o número maior que MAX_LENGTH que é potencia de 2 mais próximo, e tiro Log2. Botando em pratica, vamos dizer que é 400 o tamanho, o númerio maior que 400 que também é potencia de 2 mais próximo é 512, e log2(512) = 9, logo é preciso no minimo 9 bits pra poder representar 400 possibilidades. 
Pra implementar esse InternID então fica 'simples', em termos porque também é preciso saber qual o offset limite que vai ter no buffer. Se a gente usar um `int` que é 32bits, e usar desses 32, 9 pra representar o tamanho, logo, restam 23 bits pra representar o offset, que dá mais ou menos 8.3m o valor máximo que um offset pode ter. Sabendo que cresce de 400 em 400(NO MÁXIMO), no pior caso a gente pode alocar até ~21k de strings(isso supondo que todas as strings tenham tamanho de 400)
Pra implementar isso então, a mesma lógica das flags, mas um pouco diferente:

```

const MAX_LENGTH = 400;

class InternID {
  ptr: int
  InternID(offset: int, length: int) {
    assert(length <= MAX_LENGTH);
    this.ptr = (offset << 9) | length;
  }
}
```

O que é feito aqui é que, o `offset` é jogado 9 casas à esquerda, logo, se offset = 1, ele vira 1000000000, que é a quantia maxima de bits que `length` pode ocupar, e o uso de um OR, é pra setar esses 9 bits pra terem os bits de `length`, então a gente coloca ele alí no meio.
Esse tipo de prática é bem utilizada com por exemplo: Cores.
A idéia é simples, em decimal pra ficar mais palpável: Suponha que dado um número M, você separa ele em 3 partes, cada parte tem um significado, R,G e B, respectivamente, a ideia então é que, quando tiver algo como 100088255, a gente separa essas 3 partes e vira 100, 088, 255, que são R,G e B. A ideia é a mesma, só que é como se invés disso, os 6 numeros de cima servem pra exprimir o offset no buffer e os 3 de baixo pra exprimir o tamanho do conteúdo nesse offset. então se fossemos fazer isso: 100088 é onde começa na string, então acessa string[100088] a até string[100088 + 255]. A questão é que isso é em binário. Rgb segue aquela primeira lógica das 3 partes, só que invés de decimal, é com binário.

Posteriormente todo tipo de leitura/escrita então seria feita com o ID e não a string em si e a existência da string só continuária em um mesmo lugar eo que fica no HashMap são somente ponteiros pra um pedaço da String maior. Outras estruturas que usam práticas do tipo são QUIC, com a definição de packets, e VarID, que no caso, esse último é mais simples de abordar, usa os 2 bits mais significantes como tamanho do conteúdo a vir.

 * 00, lê 1 byte
 * 01, lê 2 bytes
 * 10, lê 4 bytes
 * 11, lê 8 bytes.

Também bancos de dados e engines 2D/3D usam esse tipo de coisa
