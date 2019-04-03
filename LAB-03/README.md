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
        KEY(Numero)
        REFERENCES NotasVenda(Numero),
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

## Alteraçõoes nas bases de dados anteriores

```SQL
ALTER TABLE NotasVenda
ADD
	CONSTRAINT cpf_vendedor
	FOREIGN
		KEY(CPFVendedor)
		REFERENCES Funcionario(CPF);
```

# 1

```SQL
CREATE TABLE NotaFiscalCompra(
    Numero                  INTEGER,
    DataCompra              DATE,
    CodigoFornecedor        INTEGER,
    
    PRIMARY
        KEY(Numero),
    FOREIGN
        KEY(CodigoFornecedor)
        REFERENCES Cliente(Codigo)
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
        KEY(Numero)
        REFERENCES NotaFiscalCompra(Numero),
    FOREIGN
        KEY(NumeroMercadoria)
        REFERENCES Mercadorias(NumeroMercadoria)
);
```

Tabela Cliente vai ser usada para armazenar também os Fornecedores.  

```SQL
CREATE VIEW Fornecedor
AS
	SELECT Cliente.*
	FROM Cliente, Notafiscalcompra
	WHERE Codigo = Codigofornecedor;
```

# 2

Uma função para pegar tudo vendido para a empresa.  
Uma função para pegar tudo comprado da empresa.  
As funções vão retornar cursor.  

```SQL
CREATE FUNCTION VendidoParaEmpresa(codigo INTEGER) RETURNS REFCURSOR AS $$
DECLARE
	cursorTabela REFCURSOR;
BEGIN
	OPEN cursorTabela FOR
	SELECT *
		FROM NotasVenda
		WHERE (codigo = CodigoCliente);
	RETURN cursorTabela;
END;
$$ LANGUAGE PLPGSQL;
```

```SQL
CREATE FUNCTION CompradoDaEmpresa(codigo INTEGER) RETURNS REFCURSOR AS $$
DECLARE
	cursorTabela REFCURSOR;
BEGIN
	OPEN cursorTabela FOR
	SELECT *
		FROM NotaFiscalCompra
		WHERE (codigo = CodigoFornecedor);
	RETURN cursorTabela;
END;
$$ LANGUAGE PLPGSQL;
```

# 3

Placeholder.

```SQL
CREATE FUNCTION EnviarNotificacaoPorEmail() 
RETURNS TRIGGER
AS $$
	BEGIN
	RETURN NULL;
	END;
$$ LANGUAGE PLPGSQL;
```

Gatilho para avisar se descer abaixo do mínimo.

```SQL
CREATE TRIGGER ProdutoQuantidadeAbaixoDoMinimo
AFTER
	UPDATE
	ON Mercadorias
	FOR EACH ROW
	WHEN (NEW.QuantidadeEstoque <= 3)
EXECUTE PROCEDURE EnviarNotificacaoPorEmail();
```

# 4

```SQL
CREATE VIEW DataQueProdutoFoiComprado AS
	SELECT NumeroMercadoria, DataCompra
	FROM ProdutosComprados, NotaFiscalCompra
	WHERE ProdutosComprados.Numero = NotaFiscalCompra.Numero;
```

```SQL
CREATE FUNCTION PrecoDeVendaMin(codigoProduto INTEGER) RETURNS NUMERIC AS $$
DECLARE
	dataDaUltimaVenda		DATE;
	codigoDaUltimaVenda		INTEGER;
	valorDaUltimaVenda		NUMERIC;
	
BEGIN
	SELECT MAX(DataCompra)
	INTO dataDaUltimaVenda
	FROM DataQueProdutoFoiComprado
	WHERE codigoProduto = NumeroMercadoria;
		
	SELECT Numero
	INTO codigoDaUltimaVenda
	FROM NotaFiscalCompra
	WHERE dataDaUltimaVenda = DataCompra;
	
	SELECT ValorUnitario
	INTO valorDaUltimaVenda
	FROM ProdutosComprados
	WHERE (codigoDaUltimaVenda = Numero
		  AND codigoProduto = NumeroMercadoria);
		
	RETURN valorMin;
END;
$$ LANGUAGE PLPGSQL;
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
