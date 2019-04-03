# Tabelas

## Bases de dados do sistema de vendas

```SQL
CREATE TABLE NotasVenda(
    Numero          INTEGER,
    DataEmissao     DATE,
    FormaPagamento  VARCHAR(50),
    CodigoCliente   INTEGER,
    CPFVendedor     VARCHAR(50),
    
    PRIMARY
        KEY(Numero),
    FOREIGN
        KEY(CodigoCliente)
        REFERENCES Cliente(Codigo)
);
```

```SQL
CREATE TABLE ItensNota(
    Numero              INTEGER,
    NumeroMercadoria    INTEGER,
    Quantidade          INTEGER,
    ValorUnitario       NUMERIC(10,2),
    
    PRIMARY
        KEY(Numero, NumeroMercadoria),
    FOREIGN
        KEY(NumeroMercadoria)
        REFERENCES Mercadorias(NumeroMercadoria)
);
```

```SQL
CREATE TABLE Mercadorias(
    NumeroMercadoria        INTEGER,
    Descricao               VARCHAR(255),
    QuantidadeEstoque       INTEGER,
    
    PRIMARY
        KEY(NumeroMercadoria)
);
```

```SQL
CREATE TABLE Cliente(
    Codigo              INTEGER,
    Nome                VARCHAR(255),
    Telefone            VARCHAR(255),
    Logradouro          VARCHAR(255),
    Numero              INTEGER,
    Complemento         VARCHAR(255),
    Cidade              VARCHAR(255),
    Estado              VARCHAR(2),
    NumeroContribuinte  INTEGER,
    
    PRIMARY
        KEY(Codigo)
);
```

## Base de dados do apoio ao controle de funcionários

```SQL
CREATE TABLE Funcionario(
    CPF                 VARCHAR(15),
    Nome                VARCHAR(255),
    Telefone            VARCHAR(255),
    Logradouro          VARCHAR(255),
    Numero              INTEGER,
    Complemento         VARCHAR(255),
    Cidade              VARCHAR(255),
    Estado              VARCHAR(2),
    CodigoDepartamento  INTEGER,
    
    PRIMARY
        KEY(CPF),
    FOREIGN
        KEY(CodigoDepartamento)
        REFERENCES Departamento(CodigoDepartamento)
        
);
```

```SQL
CREATE TABLE CargosFunc(
    CPF                 VARCHAR(15),
    CodigoCargo         INTEGER,
    DataInicio          DATE,
    DataFim             DATE,
    
    PRIMARY
        KEY(CPF, CodigoCargo, DataInicio),
    FOREIGN
        KEY(CPF)
        REFERENCES Funcionario(CPF),
    FOREIGN
        KEY(CodigoCargo)
        REFERENCES Cargo(Codigo)
);
```

```SQL
CREATE TABLE Departamento(
    CodigoDepartamento  INTEGER,
    Nome                VARCHAR(255),
    CPFChefe            VARCHAR(15),
    
    PRIMARY
        KEY(CodigoDepartamento)
);
```

```SQL
CREATE TABLE Cargo(
    Codigo      INTEGER,
    Descricao   VARCHAR(255),
    SalarioBase NUMERIC(15,2),
    
    PRIMARY
        KEY(Codigo)
);
```

# 1

## Base de dados do sistema de compras

```SQL
CREATE TABLE NotaFiscal(
    Numero                  INTEGER,
    NumeroDaCompra          INTEGER,
    DataCompra              DATE,
    CodigoFornecedor        INTEGER,
    
    PRIMARY
        KEY(Numero),
    FOREIGN
        KEY(NumeroDaCompra)
        REFERENCES ProdutosComprados(Numero),
    FOREIGN
        KEY(CodigoFornecedor)
        REFERENCES Fornecedor(Codigo)
);
```

```SQL
CREATE TABLE ProdutosComprados(
    Numero              INTEGER,
    NumeroMercadoria    INTEGER,
    Quantidade          INTEGER,
    ValorUnitario       NUMERIC(10,2),
    ValorTotal          NUMERIC,
    
    PRIMARY
        KEY(Numero, NumeroMercadoria),
    FOREIGN
        KEY(NumeroMercadoria)
        REFERENCES Mercadorias(NumeroMercadoria)
);
```

```SQL
CREATE TABLE Fornecedor(
    Codigo              INTEGER,
    Nome                VARCHAR(255),
    Telefone            VARCHAR(255),
    Logradouro          VARCHAR(255),
    Numero              INTEGER,
    Complemento         VARCHAR(255),
    Cidade              VARCHAR(255),
    Estado              VARCHAR(2),
    NumeroContribuinte  INTEGER,
    
    PRIMARY
        KEY(Codigo)
);
```

# 2

Como a idéia era identificar todos os produtos comprados ou fornecidos por um cliente/forncedor, eu queria exibir no final uma tabela com os produtos comprados pelo cliente/fornecedor.  
Utilizando procedure/functions não consigui exibir na tela o resultado da query.  

Essa query é a união de duas queries, uma que vai pegar os produtos comprados do ponto de vista de cliente e outra que vai olhar os produtos comprados do ponto de vista de fornecedor.  

```SQL
SELECT NumeroMercadoria, Descricao
    FROM (
    SELECT NumeroMercadoria AS MN
        FROM (
        SELECT Numero AS N
            FROM (
            SELECT Codigo
                FROM Cliente
                WHERE Cliente.Nome = 'Thiago' -- Nome do cliente/fornecedor
            ), NotasVenda
            WHERE CodigoCliente = Codigo
        ), ItensNota
        WHERE Numero = N
    ), Mercadorias
    WHERE NumeroMercadoria = MN
    UNION ALL
SELECT NumeroMercadoria, Descricao
    FROM (
    SELECT NumeroMercadoria AS MN
        FROM (
        SELECT Codigo
            FROM Fornecedor
            WHERE Nome = 'Thiago' -- Nome do cliente/fornecedor
        ), NotaFiscal
        WHERE CodigoFornecedor = Codigo
    ), Mercadorias
    WHERE NumeroMercadoria = MN;
```

# 3

Foi criado uma trigger que levanta um error quando o estoque está abaixo de 3.  

```SQL
CREATE OR REPLACE TRIGGER AlteracaoEstoque
    BEFORE
        UPDATE
        ON Mercadorias
        FOR EACH ROW
BEGIN
    IF(:NEW.QuantidadeEstoque <= 3) THEN
        raise_application_error(-20000, 'Estoque baixo, enviando e-mail');
    END IF;
END;
```

# 4

Essa trigger faz:
Uma query para descobrir a compra mais recente feita com aquela mercadoria.  
Outra query para descobrir o numero do produto comprado nessa compra mais recente.  
Ultima query para pegar o valor unitário desse produto.  


```SQL
CREATE OR REPLACE TRIGGER ValorMinDeVenda
    BEFORE
        INSERT
        ON ItensNota
        FOR EACH ROW
DECLARE
    MD      DATE;           -- Maior Data (ultima compra)
    NdC     INTEGER;        -- Numero da Compra
    PUC     NUMBER(10,2);   -- Preço Ultima Compra
BEGIN
    SELECT MAX(DataCompra)
        INTO MD
        FROM NotaFiscal
        WHERE NumeroMercadoria = :NEW.NumeroMercadoria;
        
    SELECT NumeroDaCompra
        INTO Ndc
        FROM NotaFiscal
        WHERE NumeroMercadoria = :NEW.NumeroMercadoria
        AND DataCompra = MD;
        
    SELECT ValorUnitario
        INTO PUC
        FROM ItensComprados
        WHERE NumeroMercadoria = :NEW.NumeroMercadoria
        AND Numero = Ndc;
        
    IF(:NEW.ValorUnitario < PUC) THEN
        raise_application_error(-20001, 'Vendendo por menos do que comprou');
    END IF;
END;
```

# 5

Foi criada uma constraint para impedir que valores sejam menores que 0.  

```SQL
ALTER TABLE Mercadorias
    ADD CONSTRAINT checkQuantidadeEstoque
        CHECK(QuantidadeEstoque >= 0);
```

Outra opção é fazer uma trigger para levantar erro quando estoque ficar abaixo de 0.

```SQL
CREATE OR REPLACE TRIGGER MinEstoque
    BEFORE
        INSERT
        ON Mercadorias
        FOR EACH ROW
BEGIN
    IF(:NEW.QuantidadeEstoque < 0) THEN
        raise_application_error(-20004, 'Estoque ficando abaixo de 0');
    END IF;
END;
```

# 6

Eu não queria ter que alterear as tabelas originais mas nesse caso eu precisava saber a quantidade máxima que um estoque conseguia armazenar.   

```SQL
ALTER TABLE Mercadorias
    ADD QuantidadeMaxEstoque INTEGER;
```

Faz duas queries, uma para obter a quantidade de itens no estoque agora e outra para descobrir o máximo.  
Gera um error case ultrapasse.  

```SQL
CREATE OR REPLACE TRIGGER LimiteEstoque
    BEFORE
        INSERT
        ON ItensComprados
        FOR EACH ROW
DECLARE
    QE      INTEGER;    -- Quantidade Estoque
    QME     INTEGER;    -- Quantidade Máximo Estoque
BEGIN
    SELECT QuantidadeEstoque
        INTO QE
        FROM Mercadorias
        WHERE NumeroMercadoria = :NEW.NumeroMercadoria;

    SELECT QuantidadeMaxEstoque
        INTO QME
        FROM Mercadorias
        WHERE NumeroMercadoria = :NEW.NumeroMercadoria;
    
    IF(:NEW.Quantidade + QE > QME) THEN
        raise_application_error(-20002, 'Ultrapassou o limite do estoque');
    END IF;
END;
```

# 7

Foi criado uma trigger para verificar se o cpf do vendedor existe, isso é feito apenas na hora da venda.  

```SQL
CREATE OR REPLACE TRIGGER IntegridadeCPF
    BEFORE
        INSERT
        ON NotasVenda
        FOR EACH ROW
DECLARE
    CPFtemp      VARCHAR(50);
BEGIN
    SELECT CPF
        INTO CPFtemp
        FROM Funcionario
        WHERE CPF = :NEW.CPFVendedor;
        
    IF (CPFtemp IS NULL) THEN
        raise_application_error(-20003, 'CPF não existe');
    END IF;
END;
```

# 8

Da query mais externa para a mais interna:  
Seleciona apenas os funcionarios que estão na tabela funcionario (já que vendas antigas podem ter funcionarios demitidos queremos apenas os que estão trabalhando atualmente).  
Query para obter o cargo de cada funcionario.  
Query para obter o salario base de cada funcionário.  
Query para descobrir quanto dinheiro foi feito com a venda de um certo item (valor unitário * quantidade).  
Query para somar o total de todas essas vendas de items.  
Query para descobrir o salário final (salário + comissão).  

```SQL
SELECT (SalarioBase + Comissao*0.05) AS Salario, Nome
    FROM (
    SELECT SUM(ValorTotal) AS Comissao, SalarioBase, Nome
        FROM (
        SELECT (ValorUnitario * Quantidade) AS ValorTotal, SalarioBase, Nome
            FROM (
            SELECT Numero AS N, CPF, SalarioBase, Nome
                FROM (
                SELECT Numero, CPF, CodigoCargo, Nome 
                    FROM (
                    SELECT NotasVenda.Numero, CPF AS C, Nome
                        FROM NotasVenda, Funcionario
                        WHERE CPFVendedor = CPF
                    ), CargosFunc
                    WHERE C = CPF
                ), Cargo
                WHERE CodigoCargo = Codigo
            ), ItensNota
            WHERE N = Numero
        )
        GROUP BY (Nome, SalarioBase)
    );
```

# 9

## Consulta de aquisições em atraso de um determinado fornecedor

Precisamos de uma data de previsão então foi adicionado a tabela.  

```SQL
ALTER TABLE NotaFiscal
    ADD DataPrevisao DATE;
```

Foi feito uma query para justamente exibir todos os items que estão atrasados.

```SQL
SELECT NotaFiscal.Numero, DataCompra
    FROM (
    SELECT Codigo
        FROM Fornecedor
        WHERE Codigo = 1    -- Trocar pelo id do fornecedor
    ), NotaFiscal
    WHERE CodigoFornecedor = Codigo
    AND CURRENT_DATE > DataPrevisao;
```

## Cadastro de produtos com todas as suas informações

```SQL
CREATE OR REPLACE PROCEDURE inserirMercadoria(Numero INTEGER, Des VARCHAR, Qtd INTEGER, QtdMax INTEGER)
AS
BEGIN
    INSERT
        INTO Mercadorias
        (NumeroMercadoria, Descricao, QuantidadeEstoque, QuantidadeMaxEstoque)
        VALUES
        (Numero, Des, Qtd, QtdMax);
END;
```

## Consulta dos N produtos mais vendidos

Lógica
```SQL
-- Deixando apenas as colunas que importam
SELECT NumeroMercadoria, Quantidade
    FROM ItensNota;
    
-- Fazendo o somatório das linhas
SELECT NumeroMercadoria, SUM(Quantidade)
    FROM (
    SELECT NumeroMercadoria, Quantidade
        FROM ItensNota
    )
    GROUP BY NumeroMercadoria;
    
-- Renomeando para poder organizar por aquela coluna
SELECT NumeroMercadoria, SUM(Quantidade) AS Quantidade
    FROM (
    SELECT NumeroMercadoria, Quantidade
        FROM ItensNota
    )
    GROUP BY NumeroMercadoria
    ORDER BY Quantidade DESC;
    
-- Renomeando para poder organizar por aquela coluna
SELECT NumeroMercadoria, SUM(Quantidade) AS Quantidade
    FROM (
    SELECT NumeroMercadoria, Quantidade
        FROM ItensNota
    )
    GROUP BY NumeroMercadoria
    ORDER BY Quantidade DESC; 
    
-- Limitando a quantidade de linhas que aparece
SELECT *
    FROM (
    SELECT NumeroMercadoria, SUM(Quantidade) AS Quantidade
        FROM (
        SELECT NumeroMercadoria, Quantidade
            FROM ItensNota
        )
        GROUP BY NumeroMercadoria
        ORDER BY Quantidade DESC
    )
    WHERE ROWNUM <= 2;
```

```SQL
SELECT *
    FROM (
    SELECT NumeroMercadoria, SUM(Quantidade) AS Quantidade
        FROM (
        SELECT NumeroMercadoria, Quantidade
            FROM ItensNota
        )
        GROUP BY NumeroMercadoria
        ORDER BY Quantidade DESC
    )
    WHERE ROWNUM <= 2;
```

## Consulta dos N maiores clientes

```SQL
SELECT ROWNUM, Nome, Gasto
    FROM (
    SELECT Nome, Gasto
        FROM (
        SELECT CodigoCliente, Gasto
            FROM (
            SELECT Numero AS N, SUM(Quantidade * ValorUnitario) AS Gasto
                FROM ItensNota
                GROUP BY Numero
            ), NotasVenda
            WHERE N = Numero
        ), Cliente
        WHERE CodigoCliente = Codigo
        ORDER BY Gasto DESC
    )
    WHERE ROWNUM <= 3;
```

## Consulta dos N maiores fornecedores

```SQL
SELECT ROWNUM, Nome, Gasto
    FROM (
    SELECT Nome, Gasto
        FROM (
        SELECT CodigoFornecedor, SUM(Gasto) AS Gasto
            FROM (
            SELECT Numero AS N, SUM(ValorTotal) AS Gasto
                FROM ItensComprados
                GROUP BY Numero
            ), NotaFiscal
            WHERE N = NumeroDaCompra
            GROUP BY CodigoFornecedor
        ), Fornecedor
        WHERE CodigoFornecedor = Codigo
        ORDER BY Gasto DESC
    )
    WHERE ROWNUM <= 3;
```

## Consulta a dados de um fornecedor/cliente X

```SQL
CREATE OR REPLACE FUNCTION informacaoCliente(Cod INTEGER)
    RETURN VARCHAR
AS
    Nome                VARCHAR(255);
    Telefone            VARCHAR(255);
    Logradouro          VARCHAR(255);
    Numero              INTEGER;
    Complemento         VARCHAR(255);
    Cidade              VARCHAR(255);
    Estado              VARCHAR(2);
    NumeroContribuinte  INTEGER;
BEGIN

    SELECT Nome, Telefone, Logradouro, Numero, Complemento, Cidade, Estado, NumeroContribuinte
        INTO Nome, Telefone, Logradouro, Numero, Complemento, Cidade, Estado, NumeroContribuinte
        FROM Cliente
        WHERE Codigo = Cod;
    
    RETURN Nome || ' ' || Telefone || ' ' || Logradouro || ' ' || Numero || ' ' || Complemento || ' ' || Cidade || ' ' || Estado || ' ' || NumeroContribuinte;
END;
```

```SQL
CREATE OR REPLACE FUNCTION informacaoFornecedor(Cod INTEGER)
    RETURN VARCHAR
AS
    Nome                VARCHAR(255);
    Telefone            VARCHAR(255);
    Logradouro          VARCHAR(255);
    Numero              INTEGER;
    Complemento         VARCHAR(255);
    Cidade              VARCHAR(255);
    Estado              VARCHAR(2);
    NumeroContribuinte  INTEGER;
BEGIN

    SELECT Nome, Telefone, Logradouro, Numero, Complemento, Cidade, Estado, NumeroContribuinte
        INTO Nome, Telefone, Logradouro, Numero, Complemento, Cidade, Estado, NumeroContribuinte
        FROM Fornecedor
        WHERE Codigo = Cod;
    
    RETURN Nome || ' ' || Telefone || ' ' || Logradouro || ' ' || Numero || ' ' || Complemento || ' ' || Cidade || ' ' || Estado || ' ' || NumeroContribuinte;
END;
```

Para testar basta usar

```SQL
BEGIN
    DBMS_OUTPUT.PUT_LINE(informacaoCliente(1));
    DBMS_OUTPUT.PUT_LINE(informacaoFornecedor(1));
END;
```

## Consulta a estoque e preço de um produto

Query para saber quantos produtos tem no estoque.  
Query para saber por quanto foi vendido o produto da ultima vez.  

```SQL
CREATE OR REPLACE FUNCTION consultaEstoquePreco(NM INTEGER)
RETURN VARCHAR
AS
    QT  INTEGER;
    VU  INTEGER;
BEGIN
    SELECT QuantidadeEstoque
        INTO QT
        FROM Mercadorias
        WHERE NumeroMercadoria = NM;
        
    SELECT ValorUnitario
        INTO VU
        FROM (
        SELECT MAX(Numero) AS N
            FROM ItensNota
            WHERE NumeroMercadoria = NM
        ), ItensNota
        WHERE Numero = N;
        
    RETURN QT || ' ' || VU;
END;
```

```SQL
BEGIN
    DBMS_OUTPUT.PUT_LINE(consultaEstoquePreco(1));
END;
```

## Consulta a dados de um pedido X

```SQL
CREATE OR REPLACE FUNCTION consultaPedido(pedido INTEGER)
RETURN VARCHAR
AS
    Nome                VARCHAR(255);
    Descricao           VARCHAR(255);
    Quantidade          INTEGER;
    ValorUnitario       NUMBER(10,2);
    ValorTotal          NUMBER;
    DataCompra          DATE;
BEGIN

    SELECT DataCompra
        INTO DataCompra
        FROM NotaFiscal
        WHERE pedido = Numero;
        
    SELECT Quantidade, ValorUnitario, ValorTotal
        INTO Quantidade, ValorUnitario, ValorTotal
        FROM (
        SELECT NumeroDaCompra AS N
            FROM NotaFiscal
            WHERE pedido = Numero
        ), ItensComprados
        WHERE Numero = N;
    
    SELECT Nome
        INTO Nome
        FROM (
        SELECT CodigoFornecedor AS CF
            FROM NotaFiscal
            WHERE pedido = Numero
        ), Fornecedor
        WHERE Fornecedor.Codigo = CF;
        
    SELECT Descricao
        INTO Descricao
        FROM (
        SELECT NumeroMercadoria AS NM
            FROM NotaFiscal
            WHERE pedido = Numero
        ), Mercadorias
        WHERE Mercadorias.NumeroMercadoria = NM;
        
    RETURN Descricao || ' ' || Quantidade || '' || ValorUnitario || '' || ValorTotal || '' || DataCompra || '' || Nome;
END;
```

## Consulta a dados de uma Nota Fiscal X

```SQL
CREATE OR REPLACE FUNCTION consultaNotaFiscal(pedido INTEGER)
RETURN VARCHAR
AS
    Nome                VARCHAR(255);
    Descricao           VARCHAR(255);
    Quantidade          INTEGER;
    ValorUnitario       NUMBER(10,2);
    ValorTotal          NUMBER;
    DataCompra          DATE;
BEGIN

    SELECT DataCompra
        INTO DataCompra
        FROM NotaFiscal
        WHERE pedido = Numero;
        
    SELECT Quantidade, ValorUnitario, ValorTotal
        INTO Quantidade, ValorUnitario, ValorTotal
        FROM (
        SELECT NumeroDaCompra AS N
            FROM NotaFiscal
            WHERE pedido = Numero
        ), ItensComprados
        WHERE Numero = N;
    
    SELECT Nome
        INTO Nome
        FROM (
        SELECT CodigoFornecedor AS CF
            FROM NotaFiscal
            WHERE pedido = Numero
        ), Fornecedor
        WHERE Fornecedor.Codigo = CF;
        
    SELECT Descricao
        INTO Descricao
        FROM (
        SELECT NumeroMercadoria AS NM
            FROM NotaFiscal
            WHERE pedido = Numero
        ), Mercadorias
        WHERE Mercadorias.NumeroMercadoria = NM;
        
    RETURN Descricao || ' ' || Quantidade || '' || ValorUnitario || '' || ValorTotal || '' || DataCompra || '' || Nome;
END;
```
