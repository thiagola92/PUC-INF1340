# Problemas encontrados

* PostgreSQL 9.4 não suporta Procedure
    * Procedures foram alterados para funções que retornam `void` 
* `PrecoDeVendaMin()` estava retornando variável que não existia.
    * Atualizado para retornar variável certa.
* `AtualizarComissao()` não chamava `return` no final.
    * Atualizado para chamar `return null`.