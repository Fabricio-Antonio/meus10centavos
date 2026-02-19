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
# OR e AND
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

