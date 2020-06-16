# Acesso e execução de aplicações no cluster da Georgia State University

Este roteiro fornece instruções para acesso ao cluster da GSU e também para execução de aplicações nesse cluster través de *scripts*. As instruções foram preparadas pelo monitor Lucas Roges e revisadas pela professora Andrea Charão.

## Acesso

Para acessar a sua área de usuário dentro do cluster, é necessário um cliente SSH e as credenciais de acesso fornecidas pela GSU.

Utilize o comando abaixo para iniciar o acesso:

```
ssh <seu-usuario>@cderlogin.gsu.edu
```

E digitar a senha quando a mensagem abaixo surgir:

```
<seu-usuario>@cderlogin.gsu.edu's password:
```

Pronto, agora você está na máquina de *login*, de onde será possível submeter as execuções utilizando o gerenciador de tarefas [Slurm](https://slurm.schedmd.com/quickstart.html).

## O cluster

A configuração do cluster pode ser verificada [neste link](https://help.rs.gsu.edu/display/PD/CDER).

Ao submeter tarefas para o gerenciador Slurm, ele colocará a tarefa numa fila e irá procurar um nó do cluster (ou nós, para aplicações distribuídas) que possua recursos disponíveis para executá-la.

### Módulos

O cluster apresenta diversos pacotes básicos, que já são suficientes para execução de aplicações desenvolvidas com OpenMP (`gcc 4.8.5`). Além desses pacotes base, o cluster apresenta diversos módulos que podem ser carregados, caso os pacotes base não supram os requisitos para um dado programa.

Note que, no pacote base, há uma versão antiga do `gcc` , que suporta OpenMP na versão 3.1. Caso seja usado algum recurso de OpenMP de uma versão superior à 3.1, utilize os comandos abaixo para verificar outras versões disponíveis do `gcc` e carregar a versão escolhida.

```
# mostrar todos os módulos disponíveis
module avail

# mostrar os módulos do gcc disponíveis
module avail Compilers/gcc

# carregar o módulo do gcc 6.3.0
module load Compilers/gcc-6.3.0
```

Como vamos usar *scripts* mais a frente para executar as tarefas, esses comandos de carregamento de módulos podem ser inseridos dentro do próprio *script*. Deve-se ter cuidado para que a aplicação seja compilada após o carregamento dos módulos, para realmente utilizar a versão carregada.

## O Slurm

O Slurm é um escalonador que recebe tarefas submetidas pelos usuários, mantém uma fila com essas tarefas e as coloca nos nós disponíveis para execução.

O comando básico do Slurm é o `sinfo`, onde é possível visualizar a lista de nós, sua disponibilidade e estado.

Caso todos os nós estejam ocupados, ou os nós livres não suprem os requisitos para execução, uma tarefa submetida entrará na fila. A verificação da fila é feita através do comando `squeue`.

Um comando bem útil, para mostrar informações dos nós do cluster e de tarefas submetidas, é o `scontrol show`. Sua utilização pode ser feita seguindo os exemplos abaixo:

```
# informações acerca de um nó
scontrol show node <node-id>

# informações acerca de uma tarefa submetida
scontrol show job <job-id>
```

O parâmetro `<node-id>` pode ser obtido através de `sinfo` e o parâmetro `<job-id>` é o identificador da tarefa, que pode ser obtido assim que a tarefa é submetida.

## Submissão de tarefas no cluster

Para execução de uma ou mais tarefas utilizando um sistema com o Slurm, existem dois comandos possíveis para utilização: `srun` e `sbatch`.

Aqui, o foco será na submissão de *scripts*, o que resulta na utilização de `sbatch`. A saída abaixo, após a utilização de `sbatch` indica que a tarefa foi submetida.

```
sbatch <script-file>                # comando executado
Submitted batch job <job-id>        # saída
```

Caso deseje cancelar a execução da tarefa, basta utilizar `scancel` acompanhado do *id* da tarefa.

```
scancel <job-id>
```

Como já abordado anteriormente, `scontrol show` pode ser útil para verificar o estado da tarefa:

```
scontrol show job <job-id>
```

Após algum tempo de completude, esse comando não estará mais disponível para verificação, mas enquanto estiver disponível trará informações relacionadas ao estado da tarefa, tempo, nó alocado (caso já esteja executando), etc. Após a indisponibilidade desse comando, uma possibilidade é verificar os arquivos de *log* do *script* e da aplicação. Esses dois tipos de arquivos serão configurados no *script* apresentado a seguir.

## *Script* para execução

A execução de tarefas através de *scripts* facilita o entendimento e a instrumentação. Essa instrumentação envolve a configuração de *hardware* solicitada ao Slurm, arquivos de saída/erro e a configuração de parâmetros para a aplicação. O *script* também possibilita configurar repetições e a execução de diversos casos dentro de um único arquivo, o que é comum em computação paralela.

Seguindo o script disponível em [run_virusim.sh](run_virusim.sh), é possível realizar algumas execuções da aplicação proposta no trabalho 4.

### Cabeçalho do *script*

Abaixo, encontra-se o cabeçalho do *script*, onde são descritos alguns detalhes da tarefa e os requisitos para a tarefa a ser executada nos comandos que seguem. Cada uma das linhas apresenta um comentário para especificar seu significado.

```
#SBATCH -J t4               # nome da tarefa
#SBATCH -o t4-%j.out        # nome do arquivo de saída (%j se torna jobID), este arquivo pega a saída padrão do terminal
#SBATCH -e t4-%j.err        # nome do arquivo de erros (%j se torna jobID), este arquivo pega erros gerados pelo script no terminal
#SBATCH -c 16               # número máximo de threads necessárias
#SBATCH -p qCDER            # partição --qCDER (partições disponíveis ao executar `sinfo`)
#SBATCH -t 01:30:00         # tempo máximo necessário (hh:mm:ss) - 1h30min
```

A documentação para essas *flags*, e para outras que também podem ser usadas em outros casos, está disponível em [Slurm Workload Manager - sbatch](https://slurm.schedmd.com/sbatch.html). Essas opções também podem ser descritas na linhas de comando.

Outro ponto, que foi citado anteriormente, é saber que, com o cabeçalho acima, existirão arquivos de saída para a execução do *script*. Um para saída padrão e outro para erros. Além da saída padrão, é possível encaminhar os *logs* específicos de cada execução para um arquivo separado, o que será mostrado a seguir. Por fim, para *scripts* que falharam, o arquivo de erros pode ser útil para descobrir a causa de uma falha na execução.

### Corpo do *script*

Primeiramente, caso haja necessidade, podem ser carregados os módulos. Para execução de aplicações em OpenMP, pode ser que alguns recursos estejam disponíveis em versões mais recentes do `gcc`. Nesse caso, seguir os comandos apresentados na seção [Módulos](#módulos) e escolher a versão necessária.

A parte de carregamento de módulos, para compilação, requer que o programa seja compilado novamente. Para tal, basta executar `make clean` e `make` para garantir que a aplicação seja compilada utilizando o módulo carregado. Caso a aplicação já tenha sido compilada, essas etapas podem ser ignoradas.

Na sequência, podemos definir algumas variáveis que facilitarão a passagem de parâmetros para a aplicação. No bloco abaixo, as variáveis se referem aos parâmetros passados para o programa, os número de *threads* a serem usados para as execuções e a quantidade de repetições.

```
SIZE=30
TRIALS=5000
PROBS=101
NTHREADS=(1 2 4 8 16)
NEXECS=10
```

Por fim, utilizamos laços e as variáveis criadas para iterar por diferentes parâmetros e realizar as repetições para cada caso de teste. No caso do bloco abaixo, o parâmetro a ser variado é o número de *threads*, que é então definido através de uma variável de ambiente. Após, são realizadas 10 execuções para cada um desses casos. A saída de cada uma das execuções estará disponível em arquivos separados, nomeados de acordo com os parâmetros da execução em questão.


```
for i in ${NTHREADS[@]}; do
    export OMP_NUM_THREADS=$i
    #echo $OMP_NUM_THREADS
    for j in $(seq 1 $NEXECS); do
        ./virusim $SIZE $TRIALS $PROBS > logs/log-nt$i-$j
    done
done
```

## Referências

- [CDER](https://help.rs.gsu.edu/display/PD/CDER)
- [How to Submit a Job on CDER](https://help.rs.gsu.edu/display/PD/How+to+Submit+a+Job+on+CDER)
- [Slurm - sbatch](https://slurm.schedmd.com/sbatch.html)
