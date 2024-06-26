# Key Points per LLVM

# Table of Contents

1. [Struttura LLVM](#struttura-llvm)
   - [Iteratori](#iteratori)
   - [Downcasting](#downcasting)
2. [Classi Importanti](#classi-importanti)

   - [PreservedAnalyses](#preservedanalyses)
   - [Value-User-Use](#relazione-user---use---value-in-llvm)
   - [Instruction](#instruction)

3. [Pass Manager llvm](#pass-manager-llvm)
4. [Scrittura di un nuovo passo](#scrittura-di-un-nuovo-passo)
   - [Creazione della classe](#1-creazione-della-classe)
5. [Passi di Trasformazione](#passi-di-trasformazione)

## Struttura LLVM

llvm traduce il nostro programma in un modulo operabile dalla sua toolchain, seguendo questa mappatura:

- Files $\rightarrow$ `Module` (Lista di `Function` e variabili globali)
- Funzioni $\rightarrow$ `Function` (Lista di BasicBlocks e argomenti)
- Basic Blocks $\rightarrow$ `BasicBlock` (lista di `Instruction`)
- Istruzioni $\rightarrow$ `Instruction` (singola istruzionea lvl di operazione elementare)

### Iteratori:

Notiamo quindi che le strutture llvm sono organizzate come liste, una lista di oggetti che a sua volta contiene un'altra lista (`doubleLinkedList`).
Per attraversare facilmente queste strutture dati possiamo sfruttare gli **Iteratori**.  
Tramite un iteratore possiamo navigare tutte le funzioni di un modulo

```c++
Module &M = ...;
for(auto iter = M.begin(); iter!= M.end(); ++iter){
    Function &F = *iter;
    // fare qualcosa con la singola istruzione
}
```

### DownCasting:

Il _DownCastign_ è il processo che consente di convertire un **puntatore** generico (un iteratore) **da** una classe _base_ **a** una classe _derivata_, consentendo l'accesso alle funzionalita specifiche di quella classe derivata.  
L'operazione di downcasting viene effettuata dalla funzione `dyn_cast<classeDerivata>(ptrClasseBase)`.

Se questa operazione ha successo, il risultato sarà un puntatore di tipo `classeDerivata*` che punta all'oggetto a cui è stato effettuato il cast, se fallisce ritorna un `nullptr`.

E per poter essere usata risulta necessaria l'inclusione del corretto file di header:

- Instruction $\rightarrow$ CallInst: `include Instruction.h`
- Module $\rightarrow$ Function: `include Module.h`

_nota_: `#include llvm/IR/classeBase.h`

_Es: downcasting da classe Instruction alla classe derivata CallInst (chiamata a funzione)_

```c++
#include "llvm/IR/Instructions.h"

Instruction *inst = ... ;
if (CallInst *call_inst = dyn_cast<CallInst>(inst))
    outs() << "I am a call instruction" << std::endl;


```

**ATTENZIONE:** Il downcasting è possibile solo tra classi che condividono una gerarchia diretta di ereditarietà! Se si tenta un downcasting senza seguire queste indicazioni `dyn_cast` restituirà un puntatore vuoto.

## Classi importanti

## `PreservedAnalyses`

È una classe utilizzata per indicare quali analisi sono state conservate dopo l'esecuzione di un passo. **Fondamentale** per comunicare al sistema di ottimizzazioni di llvm quali istruzioni siano ancora valide dopo che un passo è stato eseguito, per evitare di continuare nel processo di ottimizzazione con informazioni obsolete.  
Infatti il metodo `run()` restituisce proprio un oggetto `PreservedAnalyses` per comunicare al passmanager quali analisi siano ancora valide.  
_Reminder_: il metodo run() contiene il codice relativo a un passo!

## Relazione User - Use - Value in llvm:

### Value $\rightarrow$ valore

La classe `Value` è fondamentale per la struttura llvm, un oggetto di questo tipo rappresenta un valore o il risultato di un operazione, infatti avremo che **variabili, costanti e istruzioni** sono tutte istanze di Value.
Un nodo value ha un tipo (es: Integer, Float, ...) -> `getType()`.  
Un value può o no avere un nome -> `hasName()`, `getName()`.  
Un nodo value deve **avere** una lista di Users che lo utilizzano.

### User $\rightarrow$ istruzione

La classe `User` è chiave per llvm, essa eredita dalla classe `Value` e rappresenta un **istruzione che utilizza uno o più Value** come _operandi_.
Serve per rappresentare un entita che dipende da altri valori o istruzioni per funzionare correttamente.

> Un oggetto User rappresneta un istruzione che utilizzano altri Value come operandi.

Questa classe è responsabile di gestire gli effetti collaterali che un cambiamento nel codice potrebbe causare $\rightarrow$ se un istruzione viene modificata o cancellata, l'istruzione `User` che ne dipendeva verrà gestita per mantenere la corenza nel codice.

### Use $\rightarrow$ instr:value

La classe `Use` rappresenta l'associazione tra un'istruzione `User` e un valore `Value` suo operando.  
Un oggetto Use è usato quindi per tracciare e gestire le dipendenze tra un'istruzione e i valori che essa utilizza.  
Ogni istanza di Use rappresenat un collgamento tra un'istruzione User e uno dei sue operandi Value.

## Interazione tra le classi:

La relazione User-Use-Value in llvm mappa le dipendenze tra le istruzioni ed è **fondamentale** per gestire correttamente le **trasformazioni** di codice a seguito di ottimizzazioni al fine di assicurarsi che il progrmam risultante rimanga coerente e corretto.

### Dipendenza tra istruzioni:

Le istruzioni llvm sono costruite a partire della forma SSA ed in modo che ogni istruzione dipenda da altre per funzionare correttamente, questo tipo di dipendenza è rappresentato dalla classe `User`.

Es:

```
x = y + t
```

In questo esempio:

- `y`, `t` ed `x` sono istanze di `Value` in quanto rappresentano valori
- `x` è anche istanza di `User` in quanto rappresenta un istruzione che usa altri Value
- in `x` avremo due istanze `Use`, una x:y e una x:t

<br>

## `Instruction`

La classe Instruction rappresenta un'istruzione individuale nel codice sorgente llvm, in quanto tale gioca un ruolo fondamentale nella rappresentazione e nella manipolazione del codice IR.

Questa classe eredita da `User` e `Value` ed è istanza di entrambi gli oggetti in quanto è sia un valore (risultato dell'istruzione) che un utlizzatore di valori (usa operandi).
Es:

```llvm
%1 = add i32 10, 20
```

in questo esempio `%1` rappresenta un istruzione di addizione ma anche il valore che ne segue da tale operazione.

## Pass Manager LLVM

Il pass manager si occupa automaticamente di eseguire i passaggi invocati su tutte le funzioni all'interno di un modulo singolo o moduli diversi.

## Scrittura di un nuovo passo:

Per scrivere un nuovo llvm pass bisogna seguire i seguenti passaggi:  
(guardare appunti su Lab1 per implementazione concreta).

1. Creiamo il file sorgente .cpp con il nome del passo
   - Nella directory:
   ```bash
   llvm/lib/Transforms/Utils/NomePasso.cpp
   ```
2. Aggiungiamo il nostro passo al file `CmakeLists.txt`
   - Nella directory:
   ```bash
   llvm/lib/Transforms/Utils/CmakeLists.txt
   ```
3. Creiamo il file header .h e inseriamo il codice boilerplate necessario
   - Nella directory:
   ```bash
   llvm/Transforms/Utils/NomePasso.h
   ```
4. Scriviamo il codice sorgente del nostro passo e importiamo il file header
5. Inseriamo il passo in `PassRegistry.def`
   - Nella directory:
   ```bash
   llvm/lib/Passes/PassRegistry.def
   ```
6. Inseriamo l'inclusione del file header in `PassBuilder.cpp`
   - Nella directory:
   ```bash
   llvm/lib/Passes/PassBuilder.cpp
   ```
7. Invochiamo il nostro passo con `OPT`
   - Il comando opt sarà contenuto in:
   ```bash
   $ROOT/INSTALL/bin/opt
   ```

</br>

**Reminder**: I passi prendono in ingresso codice IR, per trasformare del codice in C in codice IR usiamo il seguente comando:

```bash
clang -O2 -emit-llvm -S -c TEST/file.c -o TEST/file.ll
```

Maggiori dettagli seguono sotto:

### 1. Creazione della classe

Ogni passo da aggiungere deve essere strutturato sotto forma di Classe C++ per scelte progettuali e architetturali di LLVM.  
Ogni classe che rappresenta un llvm pass, **dovrà** ereditare dalla classe `PassInfoMixin <NomePassaggio>` e implementare la funzione `run`.

La classe `PassInfoMixin<>` fornisce una struttura di base e funzionalità comuni per semplificare l'implementazione di nuovi passaggi.
Riduce la neccessità di scrivere codice boilerplate e mette a disposizione il meccanismo (( forse il metodo run() )) per registrare il passaggio nel sistema dei passaggi llvm.

```c++
class NewPass: public PassInfoMixin<NewPass>{
    PreservedAnalyses run(Function &F, FunctionAnalysisManager &AM);
}
```

# Passi di Trasformazione

I passi di trasformazione sono i responsabili dell'ottimizzazione della IR, prendono in ingresso una ir e producono un output ottimizzato secondo i canoni specificati dal passo stesso, modificano quindi il codice per creare una versione migliorata per il processore.
