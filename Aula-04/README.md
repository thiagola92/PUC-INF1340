# Revisão do Teste

## 1
O que faz cada um dos items:  

* Otimizador  
    * Parse
        * Análise léxica
        * Análise sintática
        * Válida o nome
        * Válida semantica
    * Plano de execução  
* Processador de consultas
    * Executa a consulta
    * Pega os dados da base
* Gerente de concorrência
    * Trata da consistência
    * Trata da independência
* Gerente de transação
    * Garantir a automocidade
        * Não vai deixar que transações diferentes sejam executadas ao mesmo tempo
* Gerente de recuperação
    * Dar rollback em caso de problema na transação

## 2
Que informações são acessadas pelo otimizador de consulta?

* Se tem índice  
* Qual a organização do tabela
    * Sequencial
    * Indexado
    * Acesso direto

## 3
Quando que sequêncial desordenado é melhor que sequêncial indexado?  

* Quando insere dados novos?

## 4
Para que serve o catálogo?

* Armazenar metadados sobre os dados armazenados
    * Quantas tuplas tem na tabela
    * Tabela é pequena ou grande
    * Cabe em memória ou não
    * O atributo X tem quantas Y possibilidade

# Extra
Flashback

* Arquivo sequêncial não ordenado
    * Heap
* Arquivo sequêncial ordenado
    * Sorted
* Arquivo acesso direto
    * Hash
        * É apenas acesso direto quando cada chave leva apenas a um valor
        * Quando quase não tem colisão
        * 90% dos valores sem colisão seria o ideal
* Arquivo sequêncial indexado
    * Árvore B/B+

